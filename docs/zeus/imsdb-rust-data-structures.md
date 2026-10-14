# Zeus における IMS DB / TM 統合アーキテクチャとデータ構造詳細解説

本ドキュメントでは、**Zeus（Rust による z/OS アプリケーションランタイム）** における **IMS DB / TM サブシステム（`zeus-ims`）** について、DBD/PSB 解析、B木索引とチェックサム付き WAL 永続化、階層セグメントのデータ構造、DL/I 呼び出しインターフェース（`CBLTDLI` / `EXEC DLI` / `PLITDLI`）、バッチ実行制御（`DFSRRC00` / アンロード・再ロード）、および IMS TM メッセージ処理キューの内部アーキテクチャを詳細に解説します。

---

## 1. IMS サブシステムのアーキテクチャ概要

### 1.1 目的と由来
メインフレームにおける階層型データベース（IMS DB）およびトランザクションマネージャ（IMS TM）は、Bank of Z や AWS CardDemo（承認モジュール `app-authorization-ims-db2-mq`）などの基幹資産で幅広く使用されています。

Zeus の IMS 実装（`crates/zeus-ims`）は、GnuCOBOL zosplus の `extras/imsdb` で開発された高品質な Rust コア（B木索引、チェックサム付き WAL、SSA 評価器等）を共通の技術基盤としつつ、**Zeus の自己完結型ランタイムおよび EBCDIC 統一メモリ空間に最適化された形で統合**されています。

主要な設計差異:
- **C ABI（FFI）の完全排除**: GnuCOBOL のような C 言語バインディングを介さず、Zeus のインタプリタ実行ループから Rust 関数として直接かつゼロコピーで DL/I を呼び出します。
- **統一 WAL とファイルロック排他**: 単一の WAL ファイル（`ZEUS_IMS_WAL` / `ims/ims.wal`）に全 DBD を保持し、バッチジョブ（`zjob`）と CICS リージョン（`zcics`）が同一データベースへ同時アクセス可能です。
- **多言語対応**: COBOL（`CBLTDLI`, `EXEC DLI`）に加え、**PL/I（`PLITDLI`）** からの呼び出しにネイティブ対応。
- **IMS TM メッセージキューの統合**: `CSLUSPOC`（SPOC コマンド）によるトランザクション定義と、キュー（`ims/tmq`）を通じたメッセージ駆動型処理（`DFSRRC00 PARM='MSG'`）。

### 1.2 全体アーキテクチャ構成図

```text
+--------------------------------------------------------------------------+
|                  COBOL / PL/I アプリケーション (.cbl / .pli)              |
|  - CALL 'CBLTDLI' USING DLI-GU, DB-PCB, IO-AREA, SSA-1...                |
|  - EXEC DLI GN USING PCB(1) INTO(SEG-AREA) WHERE(...) END-EXEC           |
|  - CALL PLITDLI(PARM-COUNT, DLI-ISRT, PCB-PTR, IO-AREA, SSA-1)           |
+--------------------------------------------------------------------------+
                                     |
                                     v  [直接 Rust 関数呼び出し / ゼロコピー]
+--------------------------------------------------------------------------+
| 1. DL/I インターフェース層 (zeus-ims: dli.rs, runtime.rs)                 |
|   - CBLTDLI / EXEC DLI / PLITDLI 引数アンパッキング                      |
|   - PCB (Program Communication Block) アドレス解決 & 状態コード (DIB/STATUS)設定|
|   - 階層位置付けカーソル管理 (Current Position, Parent Position)         |
+--------------------------------------------------------------------------+
                                     |
                                     v  [SSA 構文解析 & 条件評価]
+--------------------------------------------------------------------------+
| 2. SSA 評価エンジン (zeus-ims: dli.rs)                                    |
|   - コマンドコード解釈: D (パス呼出し), F (第1子), L (末尾), C (連結キー), P (親子関係) |
|   - ブール修飾 (* / & / # [AND], + / | [OR]) & 比較演算 (EQ, NE, GT, GE, LT, LE)|
|   - EBCDIC バイト比較 & パック10進数 (COMP-3) 数値比較                   |
+--------------------------------------------------------------------------+
                                     |
                                     v  [階層走査 & レコード操作]
+--------------------------------------------------------------------------+
| 3. 階層ストレージ & B木索引エンジン (zeus-ims: store.rs, catalog.rs)     |
|   - メタデータ管理: DBDLIB (DBDマクロ) & PSBLIB (PSBマクロ) のパース       |
|   - B木索引: ルートキー昇順探索 & 高速ランダムアクセス                    |
|   - 階層順走査 (Pre-order Traversal): 双子 (Twin) ポインタ & 親子関係管理 |
+--------------------------------------------------------------------------+
                                     |
                                     v  [永続化 & トランザクション管理]
+--------------------------------------------------------------------------+
| 4. WAL (Write-Ahead Logging) 永続化層 (zeus-ims: store.rs)                |
|   - チェックサム (CRC32) 付きページフレーム管理                           |
|   - ルート単位の行ロック + WAL ファイルレベル排他ロック (プロセス間整合性)|
|   - コミット時自動フラッシュ / 異常終了時自動バックアウト (ロールバック)  |
+--------------------------------------------------------------------------+
                                     |
                                     v  [単一永続化ファイル]
                      +-----------------------------+
                      | ims/ims.wal (または環境変数) |
                      +-----------------------------+
```

---

## 2. 内部データ構造とメモリレイアウト

### 2.1 セグメントヘッダーと物理表現
`zeus-ims` では、各セグメントは以下のメタデータを含むバイナリブロックとして管理されます:

```rust
pub struct SegmentRecord {
    pub dbd_id: u32,             // DBD 識別番号
    pub seg_code: u16,           // セグメント型コード (1=ルート, 2, 3...)
    pub flags: u8,               // 削除フラグ、可変長フラグ等
    pub parent_offset: u64,      // 親セグメントの WAL オフセット
    pub twin_next_offset: u64,   // 次の双子セグメントへのオフセット
    pub child_first_offset: u64, // 最初の子セグメントへのオフセット
    pub key_bytes: Vec<u8>,      // シーケンスフィールドの生バイト列 (EBCDIC)
    pub data_bytes: Vec<u8>,     // 業務データ本体 (EBCDIC 生バイト列)
}
```

### 2.2 階層関係（ツリー構造）の走査順序
IMS DB の基本原則である**「深さ優先・先行順（Pre-order Traversal）」**を忠実に再現します:
1. ルートセグメント（Level 1）
2. 第1の子セグメント（Level 2）
3. その孫セグメント（Level 3）
4. 孫の双子セグメント（Next Twin）
5. 次の子セグメント（Next Child）
6. 次のルートセグメント

無限定 `GN`（Get Next）呼び出しは、現在のセグメントポインタから先行順で次の有効なセグメントを返却します。`GNP`（Get Next within Parent）は、現在の親セグメントの境界を越えずに配下の子孫のみを走査します。

---

## 3. DL/I 呼び出しと SSA コマンドコード処理

### 3.1 対応する DL/I 呼び出し関数

| 関数コード | 操作名称 | 動作概要 |
| :--- | :--- | :--- |
| **`GU`** (Get Unique) | 直接取得 | SSA で指定されたキーに合致するセグメントを探索し、位置付けを確立。 |
| **`GHU`** (Get Hold Unique) | 保持付き直接取得 | 更新・削除を前提とした取得。排他ロックを獲得。 |
| **`GN`** (Get Next) | 順次取得 | 現在位置から階層順に次のセグメントを取得。 |
| **`GHN`** (Get Hold Next) | 保持付き順次取得 | 次のセグメントを更新ロック付きで取得。 |
| **`GNP`** (Get Next within Parent) | 親配下の順次取得 | 確立されている親セグメントの配下にある子セグメントのみを順次取得。 |
| **`GHNP`** (Get Hold Next in Parent)| 保持付き親配下取得 | 親配下の子セグメントを排他ロック付きで順次取得。 |
| **`ISRT`** (Insert) | 挿入 | 新しいセグメントを階層位置のキー順序に従って追加。 |
| **`DLET`** (Delete) | 削除 | `GH*` で保持されたカレントセグメントおよびその全子孫セグメントを削除。 |
| **`REPL`** (Replace) | 更新 | `GH*` で保持されたセグメントのデータ部を置換（キー変更は禁止）。 |
| **`CHKP` / `XRST`** | チェックポイント/再起動 | トランザクション同期点の記録および記号チェックポイントからの復元。 |

### 3.2 SSA（Segment Search Argument）コマンドコードの実装
Zeus は IBM メインフレームの標準的な SSA コマンドコードを高度に解釈します:
- **`*D` (Path Call)**: パス呼び出し。ルートから目的の子セグメントまでの経路上にあるすべての親セグメントのデータを、ユーザー入出力域に連結して一括返却。
- **`*F` (First)**: 親配下の最初の子双子セグメントから検索を開始。
- **`*L` (Last)**: 親配下の末尾の子双子セグメントを取得。
- **`*C` (Concatenated Key)**: 上位セグメントからの連結キーを指定して一意に位置付け。
- **`*P` (Set Parentage)**: 検索が成功したセグメントに親（Parentage）を固定（以降の `GNP` の基準とする）。

---

## 4. JCL バッチ実行と IMS ユーティリティ (`DFSRRC00`)

### 4.1 `PGM=DFSRRC00` の動作仕様
JCL バッチジョブにおいて、`//STEP EXEC PGM=DFSRRC00,PARM='...'` は Zeus 組み込みの IMS ランタイムコントローラ（`crates/zeus-jcl/src/ims.rs`）によって直接実行されます。

- **`PARM='DLI,PROGNAME,PSBNAME'`**:
  バッチ DLI 領域。PSBLIB から PSB を読み込み、PCB マスクを生成して COBOL の `PROCEDURE DIVISION USING` または `ENTRY 'DLITCBL' USING` に引数として渡します。正常終了時は自動コミット、異常終了時は WAL を自動ロールバックします。
- **`PARM='BMP,PROGNAME,PSBNAME'`**:
  BMP（Batch Message Processing）領域。I/O PCB が先頭に渡され、オンライン環境と並行してバッチ更新を実行。
- **`PARM='ULU,DFSURGU0,DBDNAME'` (アンロードユーティリティ)**:
  指定された DBD の全セグメントを階層順（キー昇順）で走査し、標準のアンロード形式レコードとして `DD DFSUROUT` へ出力。CardDemo のアンロードバイナリと 100% 互換のバイトストリームを生成します。
- **`PARM='ULR,DFSURGL0,DBDNAME'` (再ロードユーティリティ)**:
  `DD DFSUINPT` のアンロードファイルからセグメントを連続読み込みし、B木索引と WAL を再構築。

---

## 5. IMS TM (Transaction Manager) とメッセージキュー

### 5.1 SPOC によるトランザクション定義
Zeus は IMS のシステム操作コマンド（SPOC: Single Point of Control）をエミュレートする `CSLUSPOC` ユーティリティを備えています:

```jcl
//DEFTRAN  EXEC PGM=CSLUSPOC
//SYSIN    DD   *
  CREATE TRAN NAME(AUTH01) SET(PGM(COPAUS0C))
/*
```

この定義により、トランザクションコード `AUTH01` と COBOL プログラム `COPAUS0C`（および同名の PSB）がカタログに関連付けられます。

### 5.2 メッセージ駆動型実行モデル
1. **メッセージ投入**:
   CLI コマンド `zjob ims send AUTH01 "PAYLOAD..." --lterm LTERM01` により、入力メッセージがキューファイル（`ims/tmq`）へ永続化されます。
2. **メッセージ領域起動**:
   `EXEC PGM=DFSRRC00,PARM='MSG'` により、メッセージ処理領域（MPR）がキューを監視。
3. **I/O PCB 経由の対話**:
   - COBOL プログラムは I/O PCB に対して `GU` を発行し、次のメッセージテキストおよび送信元端末情報（LTERM 名、タイムスタンプ）を受信。
   - 処理完了後、`ISRT` で応答メッセージを I/O PCB（または代替 PCB）へ返却。
   - 正常終了時に出力をキューへ確定し、入力メッセージを削除。

---

## 6. 実機・ベンチマーク検証実績

Zeus の IMS サブシステムは、実資産を用いた包括的なテストを実施しています:

1. **CardDemo IMS 承認モジュール**:
   - `tools/carddemo-ims.sh test` により、実機アンロードデータ（`AWS.M2.CARDDEMO.IMSDATA.DBPAUTP0`）224 セグメントの再ロード（`DFSURGL0`）を実施。
   - `DFSURGU0` による再アンロード結果が、元の z/OS アンロードバイナリと**バイト単位で完全一致**することを確認。
   - バッチプログラム `PAUDBUNL`（`CBLTDLI GN/GNP`）、`CBPAUP0C`（`EXEC DLI BMP CHKP` による期限切れデータ削除）、`PAUDBLOD` によるデータ往復検証に成功。
   - CICS オンライン画面 `COPAUS0C` / `COPAUS1C` からの IMS DB 照会・ページ送りが正常動作。
2. **Bank of Z IMS 経路**:
   - `tools/bankofz-ims.sh` により、DBD/PSB ロード、顧客データロード、PL/I プログラム（`IBLOGIN`, `IBACSUM` 等）を含む IMS TM トランザクションの一連の認証・残高照会シーケンスを完全検証。
