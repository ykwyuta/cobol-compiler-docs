# GnuCOBOLにおけるDb2埋め込みSQL対応のアーキテクチャと処理シーケンス詳細報告書

## 1. 調査概要と全体アーキテクチャ

GnuCOBOLにおけるIBM Db2対応は、メインフレーム（z/OS）上のCOBOLで書かれた `EXEC SQL ... END-EXEC` を修正することなく、オープン環境（Linux/Docker）上のIBM Db2（Db2 Community Edition等）に対して直接実行できるように設計・実装されています。

### 1.1 4層アーキテクチャ

本実装は、**COBOLコード層**、**Pythonプリプロセッサ層**、**C言語ランタイム層（CLIドライバ境界）**、**Rust統合層（CICS/JCL連携）**、および**Db2サーバー**の協調によって成り立っています。

```text
+-------------------------------------------------------------------+
| 1. COBOLコード層                                                   |
|    EXEC SQL SELECT ... INTO :HV-NAME FROM CUSTOMER END-EXEC       |
+---------------------------------+---------------------------------+
                                  │ cobc --db2-sql
                                  ▼
+-------------------------------------------------------------------+
| 2. プリプロセッサ層 (bin/db2sql-preprocess.py)                    |
|    SQL構文解析・? パラメータ置換・CALL "DB2SQL_EXEC" コード生成     |
+---------------------------------+---------------------------------+
                                  │ GnuCOBOL コンパイル & リンク
                                  ▼
+-------------------------------------------------------------------+
| 3. C言語ランタイム層 (extras/db2sql/db2sql.c / libdb2sql.so)      |
|    COBOL型 ⇄ C/SQL型変換・IBM CLI (ODBC) API 呼び出し              |
|    SQLAllocHandle, SQLDriverConnect, SQLPrepare, SQLExecute ...   |
+-----------------+-------------------------------+-----------------+
                  │                               │
                  │ SQLCODE / SQLSTATE            │ UOW (トランザクション連携)
                  ▼                               ▼
+--------------------------------+ +--------------------------------+
| 4. Db2 データベース (サーバー) | | 5. Rust実装連携層              |
|    icr.io/db2_community/db2    | |  - cobcics (CICS SYNCPOINT連携)|
|    ポート 50000 (TCP/IP)       | |  - pl1c (PL/I埋め込みSQL)      |
|    AutoCommit OFF トランザクション | |  - jclsh (VSAM-Db2マッピング)  |
+--------------------------------+ +--------------------------------+
```

---

## 2. 各層の実装詳細と役割分担

### 2.1 COBOLコード層 (`demo.cbl`, Bank of Z 資産等)
- **`EXEC SQL INCLUDE SQLCA END-EXEC`**: `SQLCODE`（`PIC S9(9) COMP-5`）および `SQLSTATE`（`PIC X(5)`）を含む通信領域を展開。
- **ホスト変数**: `:HV-NAME`（`PIC X`）、`:HV-ID`（`COMP-5`）、`:HV-AMOUNT`（`COMP-3` パック10進数）等。
- **NULL指示変数**: `:HV-VAR :HV-IND`（`INDICATOR` 句、`PIC S9(4) COMP`）。
- **可変長文字列（VARCHAR）**: 49レベルのグループ項目（2バイト長 + 文字列）。
- **カーソル構文**: `DECLARE CURSOR FOR SELECT`, `OPEN`, `FETCH INTO`, `CLOSE`。

### 2.2 プリプロセッサ層 (`bin/db2sql-preprocess.py`)
`cobc --db2-sql` オプションにより、COBOLパーサーの手前で実行されます。
- 入力ホスト変数をすべて `?` プレースホルダーに置換。
- `SELECT ... INTO` の受け取り先ホスト変数をSQL文から抽出し、独立した出力引数リストに分離。
- メタデータ文字列 `DB2-SQL-META` を生成（例: `"01,0210"` → 入力1個、出力2個、うち1番目に出力NULL指示子あり）。
- 固定形式COBOLの72桁境界（Area B）を守りながら、`MOVE` 文と `CALL "DB2SQL_EXEC"` や `CALL "DB2SQL_OPEN"` に展開。

### 2.3 C言語ランタイム層 (`extras/db2sql/db2sql.c`)
IBM公式の **Data Server Driver for ODBC and CLI (`sqlcli.h`, `sqlcli1.h`)** を直接利用する動的ライブラリ（`libdb2sql.so`）です。
- **型変換エンジン**:
  - GnuCOBOLの内部型（`cob_field`）を検査（`COB_TYPE_ALPHANUMERIC`, `NUMERIC_BINARY`, `NUMERIC_PACKED`, `NUMERIC_COMP5`, `GROUP`）。
  - `COMP-3`（パック10進数）を `SQL_DECIMAL` として文字列経由で安全にバインド。
  - `VARCHAR`（49レベル）の2バイトビッグエンディアン長ヘッダを解析してバインド。
- **接続・認証管理**:
  - 初回SQL実行時に環境変数（`DB2_USER`, `DB2_PASSWORD`, `DB2_HOST`, `DB2_PORT`, `DB2_DATABASE` または `DB2_DSN`）から接続文字列を自動生成。
  - `SQLSetConnectAttr(..., SQL_ATTR_AUTOCOMMIT, SQL_AUTOCOMMIT_OFF)` により自動コミットを無効化。
- **カーソルテーブル**:
  - 最大32個のアクティブカーソル（`cursors[DB2_MAX_CURSORS]`）を管理。
  - カーソル名に `SQLSetCursorName` を適用し、`WHERE CURRENT OF <cursor>`（位置決め更新・削除）をサポート。
- **SQLCAフィードバック**:
  - `cob_put_s64_param` と `cob_put_param_str` により、COBOL領域の `SQLCODE` と `SQLSTATE` を確実に更新。
  - 正常終了: `SQLCODE=0`, `SQLSTATE="00000"`
  - 該当データなし: `SQLCODE=100`, `SQLSTATE="02000"`
  - 単一行SELECTで複数行ヒット: `SQLCODE=-811`, `SQLSTATE="21000"`
  - NULL値なのに指示変数なし: `SQLCODE=-305`, `SQLSTATE="22002"`

### 2.4 Rust実装連携層 (`extras/cobcics`, `extras/pl1c`, `extras/jclsh`)
Db2本体のアクセスはC言語CLIで行われますが、**トランザクションの統括や異言語・異機能連携はRust側で緊密に制御**されています。

1. **CICSオンライン環境とのUOW連携 (`extras/cobcics/src/worker.rs` & `bridge.c`)**:
   - CICSワーカー（Rustホストから起動されるプロセス）内で `bridge.c` が `DB2SQL_END_TASK(commit)` を呼び出します。
   - 通常のタスク終了（`RETURN`）では `DB2SQL_END_TASK(1)` で自動コミット。
   - 異常終了（`ABEND`）や通信障害時は `DB2SQL_END_TASK(0)` で自動ロールバック。
   - `EXEC CICS SYNCPOINT` では、CICSローカルVSAMの更新とDb2の更新を単一の同期点として不可分に確定（Commit）します。
2. **PL/I言語コンパイラでのDb2生成 (`extras/pl1c/src/main.rs`)**:
   - Rust製のPL/IコンパイラがPL/Iコード中の `EXEC SQL` を解析し、同様のDb2 C ABI（`pli_db2_select`, `pli_db2_open`, `pli_db2_commit` 等）呼び出しを生成します。
3. **JCLバッチ実行でのVSAM-Db2バックエンド (`extras/jclsh/src/vsam_sql.rs`)**:
   - Rust製JCL実行ツール `jclsh` において `vsam_backend=db2` 設定時、VSAMデータセットのテーブル作成（`create`）、削除（`drop`）、インポート/エクスポートをDb2上で実行制御します。

---

## 3. 詳細シーケンス解説

以下に、**認証接続**、**トランザクション開始**、**クエリ実行**、**コミット**に至る完全な詳細シーケンスを示します。

```mermaid
sequenceDiagram
    autonumber
    actor Program as COBOL業務プログラム
    participant Bridge as C言語ランタイム (db2sql.c)
    participant CLI as IBM Db2 CLI / ODBC Driver
    participant Db2 as Db2 Community Edition (サーバー)
    participant CICS as Rust CICS ホスト (cobcics)

    Note over Program, Db2: 【フェーズ 1: 接続と認証 (初回実行時)】
    Program->>Bridge: CALL "DB2SQL_EXEC" (最初のSQL文)
    Bridge->>Bridge: connect_db2() を呼び出し
    Bridge->>Bridge: 環境変数検証 (DB2_USER, DB2_PASSWORD, DB2_HOST, DB2_PORT, DB2_DATABASE)
    Bridge->>CLI: SQLAllocHandle(SQL_HANDLE_ENV, &db2_env)
    CLI-->>Bridge: SQL_SUCCESS
    Bridge->>CLI: SQLAllocHandle(SQL_HANDLE_DBC, &db2_connection)
    CLI-->>Bridge: SQL_SUCCESS
    Bridge->>CLI: SQLDriverConnect(connection_string: "DATABASE=...;UID=...;PWD=...")
    CLI->>Db2: TCPハンドシェイク & ユーザー認証 (ポート 50000)
    Db2-->>CLI: 認証成功・接続確立
    CLI-->>Bridge: SQL_SUCCESS

    Note over Bridge, Db2: 【フェーズ 2: トランザクション制御設定 (AutoCommit OFF)】
    Bridge->>CLI: SQLSetConnectAttr(SQL_ATTR_AUTOCOMMIT, SQL_AUTOCOMMIT_OFF)
    CLI->>Db2: SET AUTOCOMMIT = OFF
    Db2-->>CLI: OK
    opt DB2_SCHEMA 設定あり
        Bridge->>CLI: SQLExecDirect("SET CURRENT SCHEMA <schema>")
        CLI->>Db2: SET CURRENT SCHEMA
        Db2-->>CLI: OK
    end

    Note over Program, Db2: 【フェーズ 3: クエリの準備とパラメータバインド】
    Bridge->>CLI: SQLAllocHandle(SQL_HANDLE_STMT, &statement)
    CLI-->>Bridge: SQL_SUCCESS
    Bridge->>CLI: SQLPrepare(statement, "INSERT INTO ACCOUNT VALUES (?, ?, ?)")
    CLI->>Db2: SQL文解析・アクセスプラン作成 (キャッシュ)
    Db2-->>CLI: 準備完了
    Note over Bridge: COBOL引数の型検査 (COMP-5, COMP-3, PIC X, NULL指示子)
    Bridge->>CLI: SQLBindParameter(Param 1, SQL_DECIMAL, buffers[0], ...)
    Bridge->>CLI: SQLBindParameter(Param 2, SQL_VARCHAR, buffers[1], ...)
    Bridge->>CLI: SQLBindParameter(Param 3, SQL_VARCHAR, buffers[2], &indicators[2])

    Note over Program, Db2: 【フェーズ 4: クエリ実行 (暗黙的トランザクション開始)】
    Bridge->>CLI: SQLExecute(statement)
    CLI->>Db2: 実行要求送信
    Note over Db2: ★ Db2サーバー内でトランザクションが自動開始される<br/>対象行の排他ロック (X Lock) を獲得
    Db2-->>CLI: 実行成功 (1行挿入完了)
    CLI-->>Bridge: SQL_SUCCESS
    Bridge->>CLI: SQLFreeHandle(SQL_HANDLE_STMT, statement)
    Bridge->>Program: SQLCODE = 0, SQLSTATE = "00000" をSQLCAに設定してリターン

    Note over Program, Db2: 【フェーズ 5: 単一行 SELECT ... INTO の実行】
    Program->>Bridge: CALL "DB2SQL_EXEC" (SELECT BALANCE INTO :HV-BAL FROM ...)
    Bridge->>CLI: SQLAllocHandle + SQLPrepare("SELECT BALANCE FROM ... WHERE ID = ?")
    Bridge->>CLI: SQLBindParameter(WHERE条件)
    Bridge->>CLI: SQLExecute()
    CLI->>Db2: 検索実行 (S Lock 獲得)
    Db2-->>CLI: 検索結果セット準備
    Bridge->>CLI: SQLFetch() (1行目取得)
    CLI-->>Bridge: 1行目データ返却
    Bridge->>Bridge: COBOL出力変数 HV-BAL に値を格納
    Bridge->>CLI: SQLFetch() (2行目存在チェック)
    CLI-->>Bridge: SQL_NO_DATA
    Note over Bridge: 2行目がないことを確認 (多重行ヒットエラー -811 の防止)
    Bridge->>Program: SQLCODE = 0, SQLSTATE = "00000"

    Note over Program, CICS: 【フェーズ 6: コミット処理 (明示的またはCICS連携)】
    alt パターン A: COBOLによる明示的 COMMIT
        Program->>Bridge: CALL "DB2SQL_COMMIT"
        Bridge->>CLI: SQLEndTran(SQL_HANDLE_DBC, db2_connection, SQL_COMMIT)
        CLI->>Db2: COMMIT WORK
        Db2-->>CLI: コミット完了 (WALフラッシュ、ロック全解放)
        Bridge->>Program: SQLCODE = 0, SQLSTATE = "00000"
    else パターン B: CICSオンラインタスク正常終了 (Rust連携)
        Program->>CICS: EXEC CICS RETURN (業務終了)
        CICS->>Bridge: COBCICS_RETURN() -> DB2SQL_END_TASK(commit=1)
        Bridge->>CLI: SQLEndTran(SQL_HANDLE_DBC, db2_connection, SQL_COMMIT)
        CLI->>Db2: COMMIT WORK
        Db2-->>CLI: コミット完了
        Bridge->>CLI: SQLDisconnect() & SQLFreeHandle()
    else パターン C: CICS ABEND / エラーロールバック (Rust連携)
        Program->>CICS: EXEC CICS ABEND (異常終了)
        CICS->>Bridge: COBCICS_ABEND() -> DB2SQL_END_TASK(commit=0)
        Bridge->>CLI: SQLEndTran(SQL_HANDLE_DBC, db2_connection, SQL_ROLLBACK)
        CLI->>Db2: ROLLBACK WORK
        Db2-->>CLI: ロールバック完了 (変更取り消し、ロック解放)
        Bridge->>CLI: SQLDisconnect() & SQLFreeHandle()
    end
```

---

## 4. 細部仕様と技術的ポイント

### 4.1 認証と資格情報の安全な管理
- 資格情報（`DB2_USER`, `DB2_PASSWORD` 等）はソースコードや生成COBOLコードには一切埋め込まれません。
- 実行時の環境変数、または `.runtime.env` ファイル経由でプロセスに注入され、`connect_db2()` 内でメモリ上に一時的に接続文字列を構築後、直ちに `memset(connection_string, 0, ...)` でクリアされます。

### 4.2 トランザクション境界とAutoCommit=OFFの必然性
Db2 CLIのデフォルトは1文ごとに自動コミットされる `SQL_AUTOCOMMIT_ON` ですが、本実装では接続直後に `SQL_AUTOCOMMIT_OFF` に設定されます。
- これにより、メインフレームDb2と同様に、最初の更新文（INSERT/UPDATE/DELETE）の発行時にDb2サーバー側で暗黙的にトランザクションが開始されます。
- COBOL側が明示的に `EXEC SQL COMMIT WORK` を呼ぶか、CICS同期点（`SYNCPOINT` または正常タスク `RETURN`）に到達するまでロックが維持され、業務データの一貫性が完全に保護されます。

### 4.3 COBOL特有のデータ型の厳密な整合
1. **COMP-3 (Packed Decimal)**:
   - 内部表現がBCD（2进化10進）であるため、C言語側で一度フォーマット文字列に展開し、CLI側には `SQL_DECIMAL` かつ適切な桁数（Digits）と小数桁（Scale）を指定してバインドします。
2. **VARCHAR (49-Level Groups)**:
   - メインフレームDb2標準の `01 VAR. 49 VAR-LEN PIC S9(4) COMP. 49 VAR-TEXT PIC X(n).` 形式を自動認識し、先頭2バイトの長さヘッダに従ってバイト長を設定し、末尾スペースの不要なパディングを防ぎます。
3. **NULL指示子（Indicators）**:
   - COBOLの `INDICATOR` 変数が `-1` の場合は `SQL_NULL_DATA` をセットし、値がNULLであることをDb2に通知します。
   - 取得時、Db2側がNULLを返したにもかかわらずCOBOL側に指示変数が用意されていない場合は、IBM仕様通り `SQLCODE = -305`（`22002`）を返却します。

### 4.4 単一行SELECTにおける厳密な検査 (`SQLCODE = -811`)
メインフレームの `SELECT ... INTO` は「結果がちょうど1行であること」を前提としています。
`db2sql.c` では、1行目の `SQLFetch` に成功した後、**あえて2回目の `SQLFetch` を実行**します。2回目で `SQL_NO_DATA` 以外が返った場合、複数行が該当したと判断して即座に `SQLCODE = -811`（`21000`）を設定し、COBOL側にエラーを通知します。
