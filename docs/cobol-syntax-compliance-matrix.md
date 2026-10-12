# z/OS COBOL 構文対応状況マトリクス (Enterprise COBOL for z/OS Syntax Compliance Matrix)

## 1. 概要と評価基準

本ドキュメントは、IBM 公式の **Enterprise COBOL for z/OS (Language Reference SC27-8713 / Enterprise COBOL 4.2 〜 6.x)** で規定されている COBOL 言語仕様の全構文要素を体系化し、本研究リポジトリで比較対象としている **3つのプロジェクト（Zeus、GnuCOBOL zosplus、cobol-on-java）** の準拠状況・サポート範囲・制限事項を横断的にまとめたものです。

### 評価基準・記号の定義
- **●（完全対応）**: 当該構文を構文解析でき、z/OS 互換の意味論（ランタイム実行結果）を充足している。
- **▲（部分対応 / 制限あり）**: 主要な構文は通るが、一部のオプション・特定の型・組み合わせ・未対応サブセットが存在する。
- **×（未対応）**: 現時点でパーサーで拒否される、またはコンパイル/実行が未実装。
- **―（対象外 / 廃止）**: Enterprise COBOL 6.x で廃止された古い構文、または意図的にサポート対象外としている機能。

---

## 2. ソース形式・プリプロセッサ・コンパイラ指示文

| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **固定形式 (Fixed Format)** | 列 1-6(連番), 7(標識: `*`, `/`, `-`, `D`), 8-72(A/B領域), 73-80(識別) | ● | ● | ● | 全プロジェクトで完全サポート。デバッグ行（`D`）も対応 |
| **自由形式 (Free Format)** | カラム制限なし、`*>` による行末コメント | ● | ● | ● | `--free` オプション等で対応 |
| **`COPY` 文** | `COPY member [IN/OF lib] [REPLACING ==a== BY ==b==] [SUPPRESS]` | ● | ● | ● | 擬似テキスト置換、ネスト COPY に対応 |
| **`REPLACE` 文** | `REPLACE ==text1== BY ==text2==` / `REPLACE OFF` | ● | ● | ● | ソース全体のテキスト置換 |
| **条件付きコンパイル** | `>>DEFINE`, `>>IF`, `>>ELSE`, `>>END-IF`, `>>EVALUATE` | ▲ | ● | ● | Zeus は基本の条件付きコンパイルに対応 |
| **プロセス行指示** | `CBL` / `PROCESS` によるコンパイラオプション埋め込み | ● | ● | ● | ソース冒頭のコンパイラ指令 |

---

## 3. 見出し部 (IDENTIFICATION DIVISION)

| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`PROGRAM-ID`** | プログラム名の宣言。 | ● | ● | ● | 基本構文は共通 |
| **`IS INITIAL PROGRAM`** | 呼出終了後、次回呼出時に WORKING-STORAGE を初期値にリセット | ● | ● | ● | 初期状態復元セマンティクス |
| **`IS COMMON PROGRAM`** | 入れ子プログラムで兄弟プログラムからの呼び出しを許可 | ● | ● | × | cobol-on-java は入れ子プログラム未対応 |
| **`IS RECURSIVE PROGRAM`** | 再帰呼び出しを許可（ローカルストレージのスタック確保） | ● | ● | ● | 再帰サポート |
| **注記段落** | `AUTHOR`, `INSTALLATION`, `DATE-WRITTEN`, `SECURITY` 等 | ● | ● | ● | 互換性のため受理してスキップ |
| **入れ子プログラム (`NESTED`)** | 単一ソースファイル内にプログラムを入れ子定義 | ● | ● | × | cobol-on-java は並列定義 (`END PROGRAM`) のみ対応 |

---

## 4. 環境部 (ENVIRONMENT DIVISION)

### 4.1 構成節 (CONFIGURATION SECTION)
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`SOURCE-COMPUTER` / `OBJECT-COMPUTER`** | コンパイル・実行環境の宣言 | ● | ● | ● | 構文上受理 |
| **`PROGRAM COLLATING SEQUENCE`** | プログラム全体の照合順序（EBCDIC / ASCII / 独自順）指定 | ● | ● | ● | Zeus / cobol-on-java は EBCDIC 照合が既定 |
| **`SPECIAL-NAMES`** | 記号文字、通貨記号、小数点、スイッチ、照合名定義 | ● | ● | ● | 主要句をサポート |
| **`CURRENCY SIGN`** | 通貨記号の変更（`CURRENCY SIGN IS '¥'`） | ● | ● | ▲ | cobol-on-java は複数通貨記号 (`PICTURE SYMBOL`) 未対応 |
| **`DECIMAL-POINT IS COMMA`** | カンマとピリオドの数値表記入れ替え | ● | ● | ● | 全プロジェクトで完全対応 |

### 4.2 入出力節 (INPUT-OUTPUT SECTION)
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`FILE-CONTROL` / `SELECT`** | 論理ファイル名と外部ファイル名の関連付け | ● | ● | ● | JCL の DD 名と紐付け |
| **`ASSIGN TO`** | 外部データセット名 / DD 名への割り当て | ● | ● | ● | 環境変数 `DD_<NAME>` 等で解決 |
| **`ORGANIZATION`** | `SEQUENTIAL`, `INDEXED`, `RELATIVE` (VSAM KSDS/RRDS/ESDS) | ● | ● | ● | VSAM 3編成をサポート |
| **`ACCESS MODE`** | `SEQUENTIAL`, `RANDOM`, `DYNAMIC` | ● | ● | ● | 直接・動的アクセス対応 |
| **`RECORD KEY` / `ALTERNATE RECORD KEY`** | 索引ファイルの主キー・副キー定義 (`WITH DUPLICATES`) | ● | ● | ● | 代替索引（AIX）を含む |
| **`FILE STATUS`** | 2バイトのファイル状態コード受取変数 | ● | ● | ● | 10進ステータスコードを生成 |

---

## 5. データ部 (DATA DIVISION)

### 5.1 節 (Sections) & レベル番号
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`FILE SECTION`** | ファイル記述項 (`FD`)、整列併合用 (`SD`) | ● | ● | ● | 全プロジェクト対応 |
| **`WORKING-STORAGE SECTION`** | 静的作業場所記憶域 | ● | ● | ● | 全プロジェクト対応 |
| **`LOCAL-STORAGE SECTION`** | 自動スタック記憶域（再帰・スレッドセーフ用） | ● | ● | ● | 全プロジェクト対応 |
| **`LINKAGE SECTION`** | 呼出引数・CICS COMMAREA・BMS バッファ受け渡し領域 | ● | ● | ● | アドレスポインタ連携 |
| **`REPORT SECTION`** | Report Writer 用報告書記述項 (`RD`) | ● | ● | ▲ | cobol-on-java は固定改ページのみ展開 |
| **レベル番号 (01〜49, 77)** | レコード・基本項目・集団項目の階層構造 | ● | ● | ● | 全プロジェクト対応 |
| **レベル 66 (`RENAMES`)** | 既存項目の範囲を再グループ化 | ● | ● | × | cobol-on-java は未対応 |
| **レベル 88 (条件名)** | 値および範囲 (`THRU`) を持つ条件名 | ● | ● | ● | 全プロジェクト対応 |

### 5.2 主要データ記述句 (Clauses)
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`PICTURE` (基本・編集)** | `9`, `X`, `A`, `S`, `V`, `P`, `Z`, `*`, `+`, `-`, `CR`, `DB`, `$` 等 | ● | ● | ● | 編集文字列の完全展開 |
| **`USAGE DISPLAY`** | ゾーン10進数 / 文字列項目 | ● | ● | ● | EBCDIC 生バイト保持 |
| **`USAGE COMP-3` / `PACKED-DECIMAL`** | パック10進数 (1バイトに2桁 + 符号ニブル `C`/`D`/`F`) | ● | ● | ● | z/OS 特有の符号正規化 (`NUMPROC`) 準拠 |
| **`USAGE COMP` / `COMP-4` / `BINARY`** | 2進整数 (2/4/8 バイト、ビッグエンディアン) | ● | ● | ▲ | cobol-on-java は 16バイト(19-31桁) 未対応 |
| **`USAGE COMP-5`** | ネイティブ2進（PIC 桁数による切り捨てを行わない2進） | ● | ● | ▲ | cobol-on-java は 8バイト長まで |
| **`USAGE COMP-1` / `COMP-2`** | 単精度 (4B) / 倍精度 (8B) 16進浮動小数点 (HFP) | ▲ | ● | ▲ | Zeus/cobol-on-java は一部算術・DISPLAY に制限 |
| **`USAGE POINTER`** | メモリアドレス参照ポインタ | ● | ● | ▲ | cobol-on-java はポインタ算術・FP 非対応 |
| **`USAGE INDEX`** | 表引き用インデックス | ● | ● | ● | `SET UP/DOWN` 等に対応 |
| **`USAGE NATIONAL` / `UTF-8`** | UTF-16 (`PIC N`) / UTF-8 (`PIC U`) 項目 (v6.3+) | × | ▲ | × | GnuCOBOL のみ一部対応。他は `PIC X` 日本語バイト処理 |
| **`USAGE DISPLAY-1`** | ホスト DBCS 日本語項目 (SO/SI 含む) | × | ▲ | × | 日本語専用 USAGE は未対応 |
| **`OCCURS` (固定長)** | 表配列定義 (`ASCENDING/DESCENDING KEY`, `INDEXED BY`) | ● | ● | ● | 全プロジェクト対応 |
| **`OCCURS DEPENDING ON` (ODO)** | 可変長配列 (Complex ODO、ODO スライド対応) | ● | ● | ● | 全プロジェクト対応 |
| **`REDEFINES`** | 同一記憶域の多重定義・再定義 | ● | ● | ● | 全プロジェクト対応 |
| **`SIGN LEADING/TRAILING [SEPARATE]`** | 符号位置指定（独立符号 / 埋め込み符号ニブル） | ● | ● | ● | ホスト互換の符号ニブル再現 |
| **`JUSTIFIED RIGHT`** | 文字列の右詰め転記 | ● | ● | ● | 全プロジェクト対応 |
| **`BLANK WHEN ZERO`** | ゼロ時の空白表示 | ● | ● | ● | 全プロジェクト対応 |
| **`SYNCHRONIZED (SYNC)`** | ハードウェア境界アライメント（2/4/8バイト境界） | ● | ● | ● | 全プロジェクト対応 |

---

## 6. 手続き部 (PROCEDURE DIVISION) - 文 (Statements)

### 6.1 基本データ転送・初期化
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`MOVE`** | 基本転記、編集転記、型変換、切り捨て規則 | ● | ● | ● | 全プロジェクト対応 |
| **`MOVE CORRESPONDING`** | 集団項目内の同名項目一括転記 | ● | ● | ● | 全プロジェクト対応 |
| **`INITIALIZE`** | データ項目の規定値初期化 (`REPLACING` 句対応) | ● | ● | ▲ | cobol-on-java は REPLACING 変数指定が未対応 |
| **`SET` (指標・アドレス)** | `SET index UP/DOWN BY n`, `SET ptr TO ADDRESS OF` | ● | ● | ● | 基本アドレス設定をサポート |

### 6.2 算術演算文
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`ADD` / `SUBTRACT`** | 加算・減算 (`TO`, `FROM`, `GIVING`) | ● | ● | ● | 全プロジェクト対応 |
| **`MULTIPLY` / `DIVIDE`** | 乗算・除算 (`BY`, `INTO`, `GIVING`, `REMAINDER`) | ● | ● | ● | 剰余算を含む |
| **`COMPUTE`** | 算術式評価 (`+`, `-`, `*`, `/`, `**`) | ● | ● | ● | 演算優先順位・精度規則 |
| **`ROUNDED` (四捨五入)** | 演算結果の丸め処理 (MODE1 / MODE2) | ● | ● | ● | 全プロジェクト対応 |
| **`ON SIZE ERROR`** | 桁あふれ・ゼロ除算例外トラップ | ● | ● | ● | 全プロジェクト対応 |
| **`ADD/SUB CORRESPONDING`** | 集団項目内の同名項目一括算術 | ● | ● | ● | 全プロジェクト対応 |

### 6.3 制御構文・分岐
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`IF ... ELSE ... END-IF`** | 条件分岐文 (入れ子 IF, 暗黙のピリオド終端) | ● | ● | ● | 全プロジェクト対応 |
| **`EVALUATE`** | 多岐分岐 (`WHEN ALSO`, `THRU`, `ANY`, `TRUE/FALSE`) | ● | ● | ● | 完全対応 |
| **`PERFORM` (基本 / TIMES)** | 段落・節呼び出し、指定回数繰り返し | ● | ● | ● | インライン / アウトオブライン |
| **`PERFORM UNTIL`** | 条件反復 (`WITH TEST BEFORE` / `AFTER`) | ● | ● | ● | 全プロジェクト対応 |
| **`PERFORM VARYING`** | 複数インデックスの変動・多重ループ | ● | ● | ● | 全プロジェクト対応 |
| **`GO TO`** | 無条件分岐、`GO TO ... DEPENDING ON` | ● | ● | ● | 全プロジェクト対応 |
| **`ALTER`** | `GO TO` 先動的書き換え (廃止推奨だが現役資産に存在) | ● | ● | ● | CCVS85 合格のため全対応 |
| **`CONTINUE` / `NEXT SENTENCE`** | 制御継続 / 次文へのスキップ | ● | ● | ● | 全プロジェクト対応 |

### 6.4 プログラム間連絡・終了
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`CALL` (静的 / 動的)** | サブプログラム呼出 (`USING`, `RETURNING`, `ON EXCEPTION`) | ● | ● | ● | 全プロジェクト対応 |
| **`BY REFERENCE / CONTENT`** | 参照渡し / 値コピー渡し | ● | ● | ● | 全プロジェクト対応 |
| **`BY VALUE`** | C言語等の値渡しスタック規約 | ● | ● | ▲ | cobol-on-java は COBOL 呼出では未対応 |
| **`CANCEL`** | サブプログラムのメモリ解放・再初期化予約 | ● | ● | ▲ | cobol-on-java は JVM クラス破棄せずメモリ初期化 |
| **`ENTRY`** | 代替入口点定義 (`ENTRY 'SUB' USING ...`) | ● | ● | ● | 全プロジェクト対応 |
| **`EXIT` / `EXIT PROGRAM`** | 段落脱出 / サブプログラム終了 | ● | ● | ● | 全プロジェクト対応 |
| **`GOBACK`** | メインプログラムなら終了、サブプログラムなら呼出元復帰 | ● | ● | ● | 全プロジェクト対応 |
| **`STOP RUN`** | 実行単位 (Run Unit) の完全終了 | ● | ● | ● | 戻りコード反映 |

### 6.5 文字列操作・表操作
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`STRING`** | 文字列連結 (`DELIMITED BY`, `POINTER`, `ON OVERFLOW`) | ● | ● | ● | 全プロジェクト対応 |
| **`UNSTRING`** | 文字列分解 (`DELIMITED BY`, `INTO`, `TALLYING`, 等) | ● | ● | ● | 全プロジェクト対応 |
| **`INSPECT`** | `TALLYING`, `REPLACING`, `CONVERTING` (文字置換・集計) | ● | ● | ● | 全プロジェクト対応 |
| **`SEARCH` (線形探索)** | 表の先頭または現在位置からの線形検索 (`AT END`, `WHEN`) | ● | ● | ● | 全プロジェクト対応 |
| **`SEARCH ALL` (2分探索)** | ソート済み表に対する高速 2 分探索 | ● | ● | ▲ | cobol-on-java は多次元表の探索に制限あり |

### 6.6 ファイル入出力文
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`OPEN` / `CLOSE`** | `INPUT`, `OUTPUT`, `I-O`, `EXTEND` | ● | ● | ● | 全モード対応 |
| **`READ`** | `NEXT`, `INTO`, `AT END`, `INVALID KEY` | ● | ● | ● | 順読み・直接読み |
| **`WRITE`** | `FROM`, `BEFORE/AFTER ADVANCING` (印刷改行制御), `INVALID KEY` | ● | ● | ● | SYSOUT プリンタ出力対応 |
| **`REWRITE`** | 既存レコードの上書き更新 | ● | ● | ● | 全プロジェクト対応 |
| **`DELETE`** | レコードの物理・論理削除 | ● | ● | ● | VSAM KSDS / RRDS 対応 |
| **`START`** | 索引ファイルのポジショニング (`KEY EQUAL / GREATER / NOT LESS`) | ● | ● | ● | VSAM カーソル移動 |

### 6.7 整列・併合文 (SORT / MERGE)
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`SORT`** | ソートファイル (`SD`) を用いた並べ替え | ● | ● | ● | 全プロジェクト対応 |
| **`USING` / `GIVING`** | ファイル直接入力・直接出力 | ● | ● | ● | 全プロジェクト対応 |
| **`INPUT / OUTPUT PROCEDURE`**| ソート前後の手動レコード処理 (`RELEASE`, `RETURN`) | ● | ● | ● | 全プロジェクト対応 |
| **`MERGE`** | 複数ソート済みファイルの併合 | ● | ● | ● | 全プロジェクト対応 |

### 6.8 現代的データ交換文 (XML / JSON)
| 構文要素 / キーワード | IBM Enterprise COBOL 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`XML PARSE`** | XML テキストのイベント駆動型解析 (XMLSS) | × | ▲ | × | GnuCOBOL のみ一部対応 |
| **`XML GENERATE`** | COBOL データ構造から XML テキスト生成 | × | ▲ | × | GnuCOBOL のみ一部対応 |
| **`JSON GENERATE`** | COBOL データ構造から JSON 文字列生成 (v6.1+) | × | ▲ | × | GnuCOBOL のみ一部対応 |
| **`JSON PARSE`** | JSON 文字列から COBOL データ構造へ取り込み (v6.2+) | × | ▲ | × | GnuCOBOL のみ一部対応 |

---

## 7. 組み込み関数 (INTRINSIC FUNCTIONS)

Enterprise COBOL で頻繁に使用される主要関数の準拠状況です。

| 組み込み関数名 | 関数機能 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) |
| :--- | :--- | :---: | :---: | :---: |
| **`CURRENT-DATE`** | 現在日時・タイムゾーン (YYYYMMDDhhmmsshh±hhmm) | ● | ● | ● |
| **`WHEN-COMPILED`** | コンパイル日時 | ● | ● | ● |
| **`DATE-OF-INTEGER` / `DAY-OF-INTEGER`** | ユリウス日・通算日数からの日付変換 | ● | ● | ● |
| **`INTEGER-OF-DATE` / `INTEGER-OF-DAY`** | 日付からの通算日数計算 | ● | ● | ● |
| **`LENGTH` / `BYTE-LENGTH`** | データ項目のバイト長・文字数取得 | ● | ● | ● |
| **`LOWER-CASE` / `UPPER-CASE`** | 大文字・小文字変換 | ● | ● | ● |
| **`REVERSE`** | 文字列の反転 | ● | ● | ● |
| **`TRIM`** | 文字列の前後の空白削除 (LEADING/TRAILING) | ● | ● | ● |
| **`NUMVAL` / `NUMVAL-C` / `NUMVAL-F`**| 編集文字列・通貨付き文字列から数値への変換 | ● | ● | ● |
| **`INTEGER` / `INTEGER-PART`** | 整数値の切り捨て・丸め | ● | ● | ● |
| **`MOD` / `REM`** | 剰余算 (モジュロ・余り) | ● | ● | ● |
| **`MAX` / `MIN`** | 引数リスト中の最大値・最小値 | ● | ● | ● |
| **`SUM`** | 引数の合計値 | ● | ● | ● |
| **`SQRT` / `LOG` / `LOG10` / `EXP`** | 平方根・対数・指数関数 | ● | ● | ● |
| **`ORD` / `CHAR`** | 文字の順序位置取得 / 順序位置からの文字復元 | ● | ● | ● |
| **`HEX-OF` / `BIT-OF`** | 16進表現・ビット文字列表現の取得 (v6.x) | ▲ | ● | ▲ |
| **`NATIONAL-OF` / `DISPLAY-OF`** | Unicode(UTF-16) ↔ EBCDIC コード変換 | × | ▲ | × |

---

## 8. サブシステム連携文 (EXEC CICS / EXEC SQL / EXEC DLI)

| 構文要素 / キーワード | 仕様概要 | Zeus (Rust) | GnuCOBOL zosplus (C) | cobol-on-java (JVM) | 備考・差異 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`EXEC CICS SEND / RECEIVE MAP`** | 3270 BMS 画面入出力 | ● | ● | ▲ | cobol-on-java は中立スナップショット/Web画面 |
| **`EXEC CICS LINK / XCTL`** | プログラム間制御移行・COMMAREA 受け渡し | ● | ● | ● | 全プロジェクト対応 |
| **`EXEC CICS READ / WRITE / REWRITE`** | CICS ファイル制御 (VSAM KSDS/RRDS/ESDS) | ● | ● | ● | 全プロジェクト対応 |
| **`EXEC CICS START / RETRIEVE`** | 非同期タスク起動・データ授受 | ● | ● | ● | 全プロジェクト対応 |
| **`EXEC CICS READQ / WRITEQ (TS/TD)`** | 一時記憶キュー・一時データキュー | ● | ● | ● | 全プロジェクト対応 |
| **`EXEC CICS ENQ / DEQ`** | リソース排他制御 | ● | ● | ● | コルーチン/スレッド同期 |
| **`EXEC CICS SYNCPOINT`** | トランザクションコミット・ロールバック | ● | ● | ● | 2フェーズコミット連携 |
| **`EXEC CICS CHANNEL / CONTAINER`** | 大容量データ受け渡し (32KB超) | ● | × | ▲ | Zeus は Bank of Z で実装済み |
| **`EXEC SQL` (単一行 DML)** | `SELECT INTO`, `INSERT`, `UPDATE`, `DELETE` | ● | ● | ● | ホスト変数・標識変数対応 |
| **`EXEC SQL` (カーソル操作)** | `DECLARE CURSOR`, `OPEN`, `FETCH`, `CLOSE` | ● | ● | ● | 静的カーソルを完全対応 |
| **`EXEC SQL` (動的 SQL)** | `PREPARE`, `EXECUTE`, `DESCRIBE`, SQLDA | ● | ▲ | × | cobol-on-java は動的 SQL 未対応 |
| **`EXEC DLI` / `CALL 'CBLTDLI'`** | IMS DB 階層型データベースアクセス (`GU`, `GN`, `ISRT` 等) | ● | ● | ● | SSA コマンドコードを含む |

---

## 9. 削除済み旧構文 (Enterprise COBOL 6.x で廃止された構文)

以下の構文は、IBM Enterprise COBOL for z/OS 6.x において言語仕様から削除（廃止）されています。

| 構文 | 代替構文 | Zeus | GnuCOBOL zosplus | cobol-on-java | 各プロジェクトの対応方針 |
| :--- | :--- | :---: | :---: | :---: | :--- |
| **`EXAMINE` 文** | `INSPECT` 文 | ● (CCVS85互換) | ● (旧規格互換) | 拒否 (Fail-Closed) | cobol-on-java は 6.x 準拠で明示的エラー停止 |
| **`TRANSFORM` 文** | `INSPECT CONVERTING` | ● | ● | 拒否 (Fail-Closed) | 同上 |
| **`NOTE` 文** | `*` 注釈行 | ● | ● | 拒否 (Fail-Closed) | 同上 |
| **`ON` 文 (条件実行)** | カウンタ変数 + `IF` | ● | ● | 拒否 (Fail-Closed) | 同上 |
| **`CURRENT-DAY`** | `FUNCTION CURRENT-DATE` | ● | ● | 拒否 (Fail-Closed) | 同上 |

---

## 10. プロジェクト別設計思想と論点まとめ

1. **Zeus (Rust)**
   - **ターゲット**: Enterprise COBOL 4.2 (z/OS 1.12 世代)。
   - **特徴**: NIST CCVS85 の全対象モジュールに合格し、古い規格（COBOL-85）から実務拡張（Enterprise COBOL）までを統合パーサーで一括処理。XML/JSON などの最新 Web 構文は割り切り、メインフレーム実資産（CardDemo, GenApp, Bank of Z）を過不足なく動かすことにフォーカスしている。

2. **GnuCOBOL zosplus (C/Native)**
   - **ターゲット**: COBOL 85 / 2002 / 2014 + 各種ベンダー方言 (`-std=ibm`)。
   - **特徴**: オープンソースとして最も構文網羅率が高く、XML/JSON や Report Writer まで幅広くカバーする。ただし、z/OS 固有の振る舞い（EBCDIC内部配置、厳密な桁あふれ、JCL/CICS/IMS連携）は外部モジュールやコンパイラ設定オプションの組み合わせで実現している。

3. **cobol-on-java (JVM)**
   - **ターゲット**: Enterprise COBOL for z/OS 6.x + Language Environment。
   - **特徴**: **「推測による実装を行わない」「未対応構文は黙って無視せず明示的にエラー停止する（Fail-Closed）」**という厳格な設計。6.x で削除された旧構文はあえて拒否し、Hercules オラクルで検証済みの構文のみを高品質にサポート。最新 Java/Spring/Web 基盤との結合を最重要視している。
