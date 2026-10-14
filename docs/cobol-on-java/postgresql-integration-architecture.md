# cobol-on-java における PostgreSQL 統合アーキテクチャと生バイト格納詳細解説

本ドキュメントでは、**cobol-on-java（Java 21 / JVM バイトコード直接生成方式）** における **PostgreSQL データベース統合アーキテクチャ（`cobol-db2-jdbc` / `cobol-ims-rdb`）** について、標準 PostgreSQL JDBC ドライバ（pgjdbc）連携、`BYTEA` 列を活用した EBCDIC 生バイト格納、`COLLATE "C"` によるバイト順階層走査保証、Spring Boot DataSource / HikariCP 統合、および SQL 方言・カタログ互換レイヤの実装詳細を解説します。

---

## 1. PostgreSQL 統合のアーキテクチャ概要

### 1.1 目的とクラウドネイティブ DB の活用
メインフレーム資産をオープン環境へ移行するにあたり、オープンソースの **PostgreSQL**（および AWS Aurora PostgreSQL、Google Cloud SQL for PostgreSQL 等のマネージド DB）をデータストアとして活用することは、ライセンス費用の削減やスケーラビリティの観点から強く求められます。

`cobol-on-java` では、以下の設計思想に基づいて PostgreSQL との統合を実現しています:

1. **標準 JDBC エコシステムによる完全な抽象化**:
   - PostgreSQL 専用の独自クライアントを実装するのではなく、業界標準の **PostgreSQL JDBC Driver（`org.postgresql.Driver`）** を利用。接続プーリング（HikariCP）や SSL/TLS 暗号化、認証（SCRAM-SHA-256）を完全にドライバと Spring Boot に委譲。
2. **生バイト列（`BYTEA`）による EBCDIC 完全保護**:
   - 階層型データベース（IMS DB）や VSAM ファイルの RDB 格納において、PostgreSQL の `BYTEA`（可変長バイナリ列）を活用し、文字コード変換による情報欠落を 100% 排除。
3. **`COLLATE "C"` によるメインフレーム順序保証**:
   - PostgreSQL の照合順序機能（`COLLATE "C"`）をインデックスおよびソート列に指定することで、EBCDIC バイト値および階層パスの前後関係を完全維持。

### 1.2 全体アーキテクチャ構成図

```text
+--------------------------------------------------------------------------+
|                  COBOL / PL/I アプリケーション (.cbl / .class)            |
|  - EXEC SQL ... (Db2 埋め込み SQL)                                       |
|  - CALL 'CBLTDLI' USING ... (IMS 階層 DB 呼び出し)                        |
+--------------------------------------------------------------------------+
                                     │
                                     ▼
+------------------------------------+-------------------------------------+
| 1. SQL レイヤ: cobol-db2-jdbc       | 2. IMS レイヤ: cobol-ims-rdb        |
|   - Db2 方言 ⇄ PostgreSQL 方言変換 |   - HIERARCHY_PATH 採番 & 順序走査  |
|   - SYSIBM.* カタログビュー互換     |   - ルートアンカーロック (FOR UPDATE) |
+------------------------------------+-------------------------------------+
                                     │
                                     ▼  [Spring Framework データソース境界]
+--------------------------------------------------------------------------+
| 3. Spring Boot 4.1 DataSource & トランザクション管理                      |
|   - com.zaxxer.hikari.HikariDataSource (高性能コネクションプール)         |
|   - org.springframework.jdbc.datasource.DataSourceTransactionManager     |
+--------------------------------------------------------------------------+
                                     │
                                     ▼  [標準 JDBC 通信]
+--------------------------------------------------------------------------+
| 4. PostgreSQL JDBC ドライバ (org.postgresql:postgresql:42.7.x)            |
|   - SCRAM-SHA-256 認証 & SSL/TLS 通信                                    |
|   - バイナリ転送モード (BYTEA / PreparedStatements)                      |
+--------------------------------------------------------------------------+
                                     │ TCP/IP 通信 (ポート 5432)
                                     ▼
+--------------------------------------------------------------------------+
| 5. PostgreSQL データベース (オンプレミス / AWS Aurora / Cloud SQL)        |
|   - 照合順序: COLLATE "C" (厳密な ASCII/バイト値ソート順)                |
|   - テーブル群: 業務リレーショナルテーブル, IMS_SEGMENT_STORE, IMS_ROOT_INDEX |
+--------------------------------------------------------------------------+
```

---

## 2. `BYTEA` 列を活用した生バイト格納とデータ完全性

メインフレームの基幹データ（`COMP-3` パック10進数、ゾーン10進数、マルチバイト文字を含む固定長レコード）を RDB に格納する際、テキスト列（`VARCHAR` / `TEXT`）を使用すると、UTF-8 変換によって不正バイトシーケンスが破壊されるリスクがあります。

### 2.1 バイナリストレージのテーブル設計
`cobol-ims-rdb` では、全データを生バイト列（`BYTEA`）として保持します:

```sql
CREATE TABLE IMS_SEGMENT_STORE (
    DBD_NAME       VARCHAR(8)   NOT NULL,
    ROOT_KEY_RAW   BYTEA        NOT NULL,  -- EBCDIC 生バイト (キー)
    HIERARCHY_PATH VARCHAR(256) NOT NULL,  -- 階層走査パス
    SEG_NAME       VARCHAR(8)   NOT NULL,
    SEQ_KEY_RAW    BYTEA,
    SEG_DATA_RAW   BYTEA        NOT NULL,  -- レコード本体 (生バイト列)
    PRIMARY KEY (DBD_NAME, ROOT_KEY_RAW, HIERARCHY_PATH)
);
```

### 2.2 JDBC レベルでのバイナリストリーム転送
- Java 側の `ByteArrayView` から直接 `byte[]` を取得し、`PreparedStatement.setBytes(colIndex, bytes)` を呼び出します。
- PostgreSQL JDBC ドライバは、PostgreSQL のフロントエンド/バックエンドプロトコルにおいてバイナリ形式でデータを直接送受信するため、文字コード変換が一切介在せず、1 ビットの欠落もない完全な透過性が保証されます。

---

## 3. `COLLATE "C"` による階層順序とソートの保証

### 3.1 ロケール依存ソートの課題
PostgreSQL の既定の照合順序（例: `en_US.UTF-8` や `ja_JP.UTF-8`）では、英数字や記号（`-`, `/`, `.` 等）のソート順序が言語辞書順（Case-insensitive や記号無視）に従うため、システム的な順序保証が崩れる問題があります。

### 3.2 厳格なバイト値ソートの適用
`cobol-on-java` では、階層パス列および EBCDIC 照合が必要な列に対して、PostgreSQL 組み込みの **`COLLATE "C"`** を明示的に指定します:

```sql
-- 階層先行順（Pre-order Traversal）を保証するインデックス定義
CREATE INDEX IDX_IMS_SEG_PATH ON IMS_SEGMENT_STORE (
    DBD_NAME, 
    ROOT_KEY_RAW, 
    HIERARCHY_PATH COLLATE "C" ASC
);
```

これにより、`/000001/01-000100` のような階層パス文字列が、辞書規則に惑わされることなく正確にバイト値（ASCII コード値）の昇順でソートされ、DL/I の深さ優先先行順走査と完全に一致します。

---

## 4. SQL 方言の吸収とシステムカタログ互換

### 4.1 SQL 構文のトランスパイル
`cobol-db2-jdbc` の SQL プリコンパイラは、COBOL ソース中の Db2 特有の構文を PostgreSQL 互換 SQL へ動的に正規化します:

| IBM Db2 構文 | PostgreSQL 変換後構文 | 備考 |
| :--- | :--- | :--- |
| `FETCH FIRST n ROWS ONLY` | `LIMIT n` (または標準 `FETCH FIRST n ROWS ONLY`) | PostgreSQL 11 以降は標準構文をサポート。 |
| `CURRENT TIMESTAMP` | `CURRENT_TIMESTAMP` | 構文互換。 |
| `CURRENT DATE` | `CURRENT_DATE` | 構文互換。 |
| `VALUES (1, 2) INTO :HV1, :HV2` | `SELECT 1, 2` | 単一値代入文の標準化。 |

### 4.2 システムカタログビューの提供
アプリケーションがメタデータ取得のために `SYSIBM` カタログを参照する場合に備え、PostgreSQL 側に互換ビューを配備可能です:

```sql
CREATE SCHEMA IF NOT EXISTS SYSIBM;

CREATE OR REPLACE VIEW SYSIBM.SYSTABLES AS
SELECT 
    schemaname AS CREATOR,
    tablename  AS NAME,
    'T'        AS TYPE
FROM pg_catalog.pg_tables;
```

---

## 5. 他処理系との比較

| 比較項目 | cobol-on-java (`cobol-db2-jdbc`) | Zeus (`zeus-pg`) | GnuCOBOL (`extras/db2sql`) |
| :--- | :--- | :--- | :--- |
| **ドライバ・接続方式** | **公式 PostgreSQL JDBC (pgjdbc) + HikariCP**| Pure Rust PostgreSQL クライアント | `libpq` の C 言語動的リンク |
| **生バイト格納** | **`BYTEA` 列 + ByteArrayView 直結** | DRDA レイヤでの変換 | C 言語バッファ / BLOB |
| **ソート順序保証** | **`COLLATE "C"` インデックスによる順序保証** | `pg-ebcdic037.rules` (ICU 照合順序) | PostgreSQL のロケール設定に依存 |
| **コネクション管理** | **Spring Boot / HikariCP による高度なプーリング**| 自前ソケット管理 | 単一接続または外部 ODBC プール |
| **クラウド親和性** | **AWS Aurora, GCP Cloud SQL, Azure Database 最適化**| 単一 TCP ソケット接続 | ネイティブクライアント依存 |
| **可観測性** | **OpenTelemetry, Micrometer, JDBC メトリクス** | 内部ログ | OS システムコール追跡 |
