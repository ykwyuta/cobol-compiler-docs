# z/OS ワークロード移行・実行方式 比較研究プロジェクト
(z/OS Workload Migration & Execution Architecture Study)

## 1. プロジェクトの目的

本プロジェクトの目的は、IBM メインフレーム（z/OS）上で稼働する基幹業務資産（COBOL、PL/I、JCL、CICS、Db2、IMS、BMS 等）を**「原本無修正（ソースコード無変更）」**でオープン環境（Linux / クラウド / コンテナ等）において動作させるための **3つの異なるアーキテクチャ・アプローチ** を比較・検証し、それぞれの実装方式・特徴・利点、および移行における核心的な技術論点を明確化することです。

代表的なベンチマーク資産（AWS CardDemo、CICS GenApp、Bank of Z 等）を同一のテスト対象とし、それぞれのアプローチがどのような設計判断のもとに課題を解決しているのかを体系的にドキュメント化・分析します。

---

## 2. 比較対象の3つのアプローチ

本リポジトリでは、`resource/` 配下に Git Submodule として以下の3つの独立した処理系・ランタイムを取り込んでいます。

| アプローチ | リポジトリ (`resource/`) | 実装言語 / コア技術 | 主な実行モデル |
| :--- | :--- | :--- | :--- |
| **A. Zeus** | `Zeus` | Rust | 独自コンパイラ (`zcobc`) → 中間表現 (ZIR) → Rust インタプリタ実行 |
| **B. GnuCOBOL (zosplus)** | `gnucobol-zosplus` | C / Rust | COBOL → C言語トランスパイル → ネイティブマシン語実行 (`gcc`/`clang`) |
| **C. cobol-on-java** | `cobol-on-java` | Java 21 / ASM | COBOL/PL/I/HLASM → JVM バイトコード直接生成 (`.class`) |

---

## 3. 各アプローチの実装・特徴・利点

### A. Zeus（Rust によるスタンドアロン z/OS エミュレーション）
- **実装概要**:
  - Rust でゼロから実装された z/OS 互換アプリケーションランタイム。
  - Enterprise COBOL 4.2 準拠の独自コンパイラ `zcobc`、JCL 実行エンジン `zjob`、TN3270 サーバー内蔵 CICS `zcics`、DRDA プロトコルを話す Db2 クライアント `zeus-drda`（PostgreSQL 翻訳アダプタ対応）、IMS DB サポート等を単一のRustエコシステム内で完結。
- **特徴**:
  - メモリ空間内でもデータを **EBCDIC 生バイト** および **z/OS ネイティブのストレージレイアウト（ゾーン10進数、パック10進数 COMP-3 等）** のまま保持。
  - 単一バイナリによる自己完結性が高く、重厚な外部仮想マシンやミドルウェアに依存しない。
- **利点**:
  - メモリ安全性と高速な起動速度（コンテナやサーバーレス環境に適した軽量性）。
  - z/OS の内部動作（TCB コルーチン実行、DRDA 通信、3270 スクリーンバッファ等）を低レイヤで極めて精密に模倣可能。

### B. GnuCOBOL zosplus（Cトランスパイル ＋ ネイティブコンパイル）
- **実装概要**:
  - 実績あるオープンソース COBOL コンパイラ GnuCOBOL（C言語へのトランスパイラ）をベースに、z/OS 拡張群を追加したフォーク実装。
  - Rust 製 JCL ランタイム `jclsh`、CICS エミュレーション `cobcics`、Db2 連携 `db2sql`、IMS DB `imsdb`、PL/I 連携 `pl1c` などを拡張 (`extras/`)。
- **特徴**:
  - COBOL ソースコードを C 言語コードに変換し、ホスト環境のネイティブコンパイラ（GCC/Clang）で実行ファイルを生成。
  - Linux / Docker コンテナ基盤との極めて高い親和性。
- **利点**:
  - マシン語ネイティブ実行による**圧倒的な純粋計算・大量バッチ処理性能**。
  - 長年の GnuCOBOL コミュニティによる COBOL 規格準拠度（COBOL 85 / 2002 / 2014）と安定性の恩恵。
  - 標準 C/Linux ツールチェイン（gdb、perf、valgrind 等）を用いたプロファイリングや解析が容易。

### C. cobol-on-java（JVM バイトコード直接生成 ＋ エンタープライズ Java 統合）
- **実装概要**:
  - Java 21 + ASM を用い、COBOL / PL/I / HLASM のソースコードから直接 JVM バイトコード（`.class`）を生成。
  - Spring Boot 4.x、JDBC、JMS（RabbitMQ）、Thymeleaf（Web による 3270 画面再現）、Maven プラグイン（`cobol-maven-plugin`）など、モダン Java エコシステムと緊密に連携。
- **特徴**:
  - メインフレームエミュレータ（Hercules）や z/OS 実機プログラミングによる**厳密な期待値採取（Oracle）と三層検証**を実施。
  - 連続バイト列上のビューとしてデータ項目を表現し、Java 上でバイト透明性と EBCDIC/文字コード整合性を両立。
- **利点**:
  - **エンタープライズ Java / クラウドネイティブ基盤との完全な統合**（Spring Security、接続プール、APM 監視、分散トレーシング、Kubernetes 等の恩恵をそのまま享受）。
  - Java 開発者が既存の開発・CI/CD パイプライン（Maven/JUnit）のまま資産を保守・テスト可能。
  - Web UI（HTML/CSS Grid/Thymeleaf）による 3270 画面のブラウザネイティブ再現。

---

## 4. アーキテクチャ比較表

| 比較項目 | Zeus (Rust) | GnuCOBOL zosplus (C/Native) | cobol-on-java (JVM) |
| :--- | :--- | :--- | :--- |
| **コンパイル方式** | COBOL → ZIR → インタプリタ実行 | COBOL → C言語 → ネイティブマシン語 | COBOL/PL/I/HLASM → JVM バイトコード |
| **内部データ表現** | EBCDIC 生バイト保持 | 設定により ASCII / EBCDIC | EBCDIC 生バイト（連続バイト列のビュー） |
| **計算・実行性能** | インタプリタ（起動最速・中速実行） | ネイティブ（最高速のバッチスループット） | JIT コンパイル（ウォームアップ後高速） |
| **JCL 処理系** | Rust 内蔵 `zjob` | Rust 実装 `jclsh` | Java 内部モデル `cobol-job` |
| **CICS / 画面** | `zcics` (TN3270 / 3270E プロトコル) | `cobcics` (3270 / コンソール) | `cobol-spring-boot-4` (Thymeleaf / Web UI) |
| **DB 連携 (Db2)** | `zeus-drda` (DRDA) / PostgreSQL 翻訳 | `db2sql` (IBM Db2 CLI / PostgreSQL) | `cobol-db2` (JDBC / Spring DataSource) |
| **IMS DB / TM** | 内蔵 IMS エンジン | `extras/imsdb` | `cobol-ims` (RDB 生バイト格納 / JMS 連携) |
| **周辺言語 (PL/I, ASM)** | PL/I (部分対応) | `pl1c` | PL/I (コンパイラ) + HLASM (アセンブラ) |
| **エコシステム親和性** | スタンドアロン / マイクロコンテナ | Linux CLI / Docker コンテナ | Spring Boot / Maven / クラウドJava基盤 |

---

## 5. 何が論点なのか（移行・実行における核心的技術課題）

メインフレーム資産をオープン環境へ移行する際、アプローチごとに以下の重要な設計判断とトレードオフが存在します。

1. **文字コード・メモリ配置の完全一致 vs オープン環境親和性**
   - **論点**: EBCDIC と ASCII/UTF-8 の変換タイミング、パック10進数（COMP-3）やゾーン10進数の桁あふれ・丸め規則、REDEFINES によるメモリの多重定義。
   - **Zeus / cobol-on-java**: メモリ内部を徹底して EBCDIC 生バイト列として扱い、外部I/O境界でのみ変換することで原本の動作完全性を最優先。
   - **GnuCOBOL**: オープン環境のC言語ネイティブ型やシステムコールを活かしつつ、どこまで方言・互換性オプションでカバーするか。

2. **実行ランタイムのライフサイクルとインフラ運用**
   - **論点**: メインフレームの「ステップ単位・ジョブ単位でプロセスが終了しリソースが初期化される世界」を、クラウドや長時間稼働プロセスでどう扱うか。
   - **Zeus / GnuCOBOL**: プロセス起動・終了が極めて軽量なため、従来のバッチジョブ（1ステップ = 1プロセス）モデルと自然に合致。
   - **cobol-on-java**: JVM の起動負荷を避けるため、長時間稼働するアプリケーションサーバー（Spring Boot）上でジョブやトランザクションを実行するアーキテクチャ設計が必要。

3. **CICS / 3270 端末の着地点**
   - **論点**: 既存の端末エミュレータ（TN3270）をそのまま使い続けるか、Web（HTML5/ブラウザ）へ置き換えるか。
   - **Zeus**: TN3270/TN3270E サーバープロトコルを忠実に実装し、既存のエミュレータ端末から接続可能。
   - **cobol-on-java**: 3270 の画面データ構造（BMS）を CSS Grid / Thymeleaf によるモダン Web 画面へ変換し、エミュレータレスの脱レガシーを目指す。

4. **階層型 DB (IMS DB) や 独自ファイル (VSAM) のデータストア設計**
   - **論点**: オープン系 RDBMS（PostgreSQL, MySQL）や分散 KVS へどのようにマッピングするか。
   - スキーマをリレーショナルに正規化すると COBOL 側の DL/I 呼び出しや VSAM キー検索が破綻するため、**「RDB の BLOB 列に生バイトとして格納し、インデックスのみをRDBで管理する透過的アプローチ」**の採用可否やロック制御（ルートアンカーロック、競合リトライ）が論点となります。

5. **互換性の検証根拠（オラクル）**
   - **論点**: 「メインフレームと同じ動作をする」ことを何を基準にして証明・担保するか。
   - NIST CCVS85 などの標準適合性テストに加え、Hercules（ハードウェア命令レベルのエミュレータ）や実機プログラミングによって同一テストを流し、バイト単位で期待値を突き合わせる検証基盤の重要性。

---

## 6. ディレクトリ構成

```
.
├── .gitmodules              # サブリポジトリ設定
├── README.md                # 本ドキュメント（プロジェクト概要・比較研究）
├── docs/                    # アーキテクチャ解説・比較ドキュメント群
│   ├── cobol-syntax-compliance-matrix.md # z/OS COBOL 構文対応状況マトリクス
│   ├── pli-syntax-compliance-matrix.md   # z/OS PL/I 構文対応状況マトリクス
│   ├── 3270-emulator-compliance-matrix.md # 3270 エミュレータ・データストリーム仕様準拠レポート
│   ├── cobol-on-java/       # cobol-on-java に関する調査・設計記録
│   ├── gnucobol/            # GnuCOBOL (zosplus) アーキテクチャ解説
│   └── zeus/                # Zeus アーキテクチャ解説
└── resource/                # サブリポジトリ（実コード資産）
    ├── Zeus/                # [A] Rust 製 z/OS アプリケーションランタイム
    ├── gnucobol-zosplus/    # [B] C/Native GnuCOBOL z/OS 拡張
    └── cobol-on-java/       # [C] Java 21 / JVM COBOL 処理系
```

