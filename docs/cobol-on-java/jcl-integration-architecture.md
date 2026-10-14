# cobol-on-java における JCL 統合アーキテクチャとジョブ実行エンジン詳細解説

本ドキュメントでは、**cobol-on-java（Java 21 / JVM バイトコード直接生成方式）** における **JCL（Job Control Language）実行エンジン（`cobol-job`）** について、Java 製 JCL パーサー、Spring Batch / JVM 実行ライフサイクルとの統合、DD 文とオープン系リソース（ファイル / クラスパス / JDBC / 仮想スプール）のマッピング、条件分岐（COND / IF-THEN-ELSE）、およびメインフレーム標準ユーティリティの Java エミュレーションを詳細に解説します。

---

## 1. JCL ジョブエンジンのアーキテクチャ概要

### 1.1 目的と「JVM 内長時間稼働プロセス・Spring 統合」の思想
メインフレーム（z/OS）における基幹バッチ処理は、JCL によってジョブステップの順序、入出力データセットの割り当て、条件判定、プログラム起動が定義されています。

GnuCOBOL や Zeus のような「OS プロセス起動型」のアプローチに対し、`cobol-on-java` では **「Java 仮想マシン（JVM）および Spring Batch のライフサイクルに最適化されたジョブ実行エンジン（`cobol-job`）」** を採用しています。

JVM の起動負荷（クラスロード、JIT コンパイルのウォームアップ）を排除するため、長時間稼働するバッチサーバー（Spring Boot / Spring Batch）上でジョブステップを軽量な Java スレッド / タスクとして連続実行し、JVM の JIT 最適化を最大限に引き出します。

### 1.2 全体アーキテクチャ構成図

```text
+--------------------------------------------------------------------------+
|                          JCL ソース原本 (.jcl)                           |
|  //JOBNAME  JOB  (ACCT),'TITLE',CLASS=A                                  |
|  //STEP01   EXEC PGM=CBACT01C,PARM='20261014'                            |
|  //ACCTFILE DD   DSN=AWS.M2.CARDDEMO.ACCTDATA,DISP=SHR                   |
|  //SYSOUT   DD   SYSOUT=*                                                |
+--------------------------------------------------------------------------+
                                     │
                                     ▼  [cobol-job: JclParser]
+--------------------------------------------------------------------------+
| 1. JCL 構文解析 & ジョブ定義層 (cobol-job: JclParser, ProcedureExpander)   |
|   - 80 桁カード化・継続行・インストリームデータ (DD *, DD DATA) の解析     |
|   - JCLLIB ORDER / INCLUDE MEMBER の再帰的展開                           |
|   - カタログド PROC 展開 & パラメータ置換 (&VAR, &&TEMP)                  |
|   - 抽象ジョブ定義モデル (JobDefinition, StepDefinition) の構築           |
+--------------------------------------------------------------------------+
                                     │
                                     ▼
+--------------------------------------------------------------------------+
| 2. Spring Batch 統合 & 実行制御層 (JobLauncher, StepExecution)           |
|   - JobExecution / StepExecution コンテキストの生成                      |
|   - 前ステップ終了コード (RC / ABEND) に基づく COND / IF ガード評価      |
|   - ステップ単位のトランザクション境界 (PlatformTransactionManager) 管理 |
+--------------------------------------------------------------------------+
                                     │
                                     ▼  [DD リソースバインド: ResourceResolver]
+--------------------------------------------------------------------------+
| 3. DD データセットマッピング層 (cobol-job: DatasetManager)                |
|   - DSN プレースホルダー / プロファイル解決 (application.yml 連携)        |
|   - リソースプロバイダ:                                                  |
|     * ローカルファイル (順次 FB/VB, VSAM クラスター)                     |
|     * クラスパスリソース (テスト用シードデータ)                           |
|     * JDBC 仮想テーブル (VSAM/順次ファイルを RDB 列へマッピング)          |
|     * 仮想スプール (SYSOUT / SYSIN)                                      |
|   - DISP 状態マシン (NEW/OLD/SHR/MOD -> CATLG/KEEP/DELETE)               |
+--------------------------------------------------------------------------+
                                     │
                                     ▼  [ステップ実行委託]
+------------------------------------+-------------------------------------+
| 実行委託 A: 翻訳済み COBOL / PL/I   | 実行委託 B: Java エミュレーション   |
| (同一 JVM 内で直接メソッド呼出し)   | メインフレームユーティリティ:       |
|  - CobolSession 経由の起動         |  - IEFBR14 (DD DISP 後処理)         |
|  - JCL PARM の LINKAGE 注入        |  - IEBGENER (レコードコピー)        |
|  - DD 名環境設定                   |  - IDCAMS (VSAM 定義/削除/REPRO)    |
|                                    |  - SORT (Java 照合順序ソート)       |
+------------------------------------+-------------------------------------+
```

---

## 2. JCL パースと PROC 展開

### 2.1 80桁カード構文の Java オブジェクト化
`cobol-job` の `JclParser` は、メインフレーム JCL のパンチカード規則を解釈し、型安全な Java オブジェクト（`JobModel`）へ変換します:

1. **カード文の抽出**:
   - `//`（ステートメント）、`//*`（コメント）、`/*`（デリミタ）。
   - カンマ継続および 16 桁目からの継続行の自動結合。
2. **インストリームデータ**:
   - `DD *` および `DD DATA,DLM=xx` によるインラインデータをキャプチャし、一時的な `InputStream` リソースとして登録。
3. **プロシージャ展開**:
   - `JCLLIB` で指定されたライブラリパスからカタログド PROC を探索。
   - `EXEC procname` のキーワード引数を PROC 本文へ上書きマージし、シンボル置換（`&NAME`）を解決。
   - `STEPNAME.DDNAME` による DD 文のオーバーライド合成。

---

## 3. DD 文とオープン系リソースのマッピング

### 3.1 柔軟なデータセットリゾルバ
メインフレームの `//DDNAME DD DSN=...` は、Spring Boot の設定ファイル（`application.yml`）や環境設定に基づいて、オープン環境の多様なストレージへ透過的にルーティングされます:

```yaml
cobol:
  job:
    dataset-mapping:
      # メインフレーム DSN とローカルファイルパスのマッピング
      "AWS.M2.CARDDEMO.ACCTDATA": "file:///data/carddemo/acctdata.dat"
      "AWS.M2.CARDDEMO.CUSTDATA": "file:///data/carddemo/custdata.dat"
      # 一時データセットの格納先
      temp-dir: "/tmp/cobol-jobs"
```

### 3.2 抽象リソースポート
COBOL プログラム内部で `SELECT ACCTFILE ASSIGN TO UT-S-ACCTFILE` と定義されている場合、ランタイムは以下の優先順位でリソースを解決します:
1. JCL の `//ACCTFILE DD` カードで明示的にバインドされたファイルまたはスプール。
2. 環境変数 `DD_ACCTFILE`。
3. クラスパスリソース（`classpath:data/ACCTFILE`）。
4. 仮想スプール（`SYSOUT=*` の場合、Spring Boot のロギングフレームワーク Logback / SLF4J またはジョブログファイルへ自動保存）。

### 3.3 DISP 状態マシンのエミュレーション
- `DISP=(NEW,CATLG,DELETE)`: ステップ開始時にファイルを新規作成。ジョブ正常終了時にカタログ登録、異常終了（ABEND）時にファイルを自動物理削除。
- `DISP=(OLD,KEEP)`: 既存ファイルを共有ロックまたは排他オープン。
- `DISP=MOD`: ファイル末尾への追記モード。

---

## 4. 条件分岐とステップ実行制御

### 4.1 COND パラメータと復帰コード
各ステップの終了時に、COBOL プログラムの `RETURN-CODE` 特殊レジスタの値が `StepExecution.exitCode` として記録されます。

`cobol-job` の条件評価エンジンは、後続ステップの実行前に以下の条件を判定します:
- **`COND=(code, operator, step)`**:
  メインフレームの逆論理規則（「条件が真ならステップをスキップ」）を厳密に評価。
- **`COND=EVEN`**:
  先行ステップで ABEND が発生していても強制実行。
- **`COND=ONLY`**:
  先行ステップで ABEND が発生した場合のみ実行。

### 4.2 構造化 IF 構文 (`IF / THEN / ELSE / ENDIF`)
```jcl
//IF01    IF (STEP01.RC = 0) THEN
//STEP02  EXEC PGM=PROG02
//        ELSE
//STEP03  EXEC PGM=PROG03
//ENDIF   ENDIF
```
構文木として解釈された IF ブロックに基づき、Spring Batch の `Decider` パターンに準拠して実行ステップを動的に分岐させます。

---

## 5. メインフレームユーティリティの Java エミュレーション

メインフレームのバッチジョブで多用される標準ユーティリティを、ネイティブバイナリに依存せず Java 実装として提供しています:

1. **`IEFBR14`**:
   何もしないプログラム。DD カードの `DISP` 状態遷移（データセットの事前削除や空作成）を行うために利用。
2. **`IEBGENER`**:
   `SYSUT1` からレコードを読み込み、そのまま `SYSUT2` へバイト単位で完全コピー。
3. **`IDCAMS` (VSAM ユーティリティ)**:
   - `DEFINE CLUSTER`: VSAM クラスター（KSDS/ESDS/RRDS）のメタデータ定義。
   - `DELETE`: クラスターまたはデータセットの削除。
   - `REPRO`: 順次データセットと VSAM 間の双方向レコードロード。
4. **`SORT` (DFSORT / SyncSort エミュレーション)**:
   `SYSIN` の `SORT FIELDS=(pos, len, type, order)` を解析し、Java の高性能なインメモリ/外部マージソートを用いて、EBCDIC 照合順序に基づくレコード並べ替えを実行。

---

## 6. 他処理系との比較

| 比較項目 | cobol-on-java (`cobol-job`) | Zeus (`zeus-jcl`) | GnuCOBOL (`extras/jclsh`) |
| :--- | :--- | :--- | :--- |
| **実行モデル** | **同一 JVM 内スレッド実行 (Spring Batch 統合)**| Zeus 単一バイナリ内高速実行 | ステップごとに Docker コンテナを起動 |
| **プロセス起動負荷** | **ゼロ（JIT コンパイルの恩恵を継続享受）** | 極めて軽量（ミリ秒） | コンテナ起動オーバーヘッド（秒単位） |
| **リソースバインド** | **Spring Resource (ファイル, S3, JDBC, スプール)**| カタログ / ホストファイルシステム | ホストディレクトリのマウント |
| **スプール・ログ** | **SLF4J / Logback / 分散ログ収集基盤 (ELK)**| ローカルジョブディレクトリ | `result.json` および SYSOUT |
| **スケーラビリティ** | **Kubernetes 上での Spring Cloud Data Flow 連携**| 単一ホスト / コンテナ実行 | Docker Compose / スクリプト |
