# Javaコーディング規約（AI指示用 / AI Prompt & Coding Guidelines）

本ドキュメントは、AIアシスタントがJavaソースコードの新規作成、リファクタリング、コードレビューを実施する際に厳格に遵守すべきコーディング標準および設計規約である。
コード生成時は以下のルールを最優先で適用すること。

---

## 1. 規約遵守レベルの定義
- **【必須 (MUST / PROHIBITED)】**: いかなる場合も厳格に従うこと。違反はバグまたは規約違反とみなす。
- **【推奨 (SHOULD / PREFER)】**: 特別な設計上の理由・制約がない限り従うこと。
- **【状況依存 (CONDITIONAL)】**: プロジェクト方針や要件に応じて選択・統一すること。

---

## 2. 設計原則 & 基本方針 (Core Principles)

### 2.1 オブジェクト指向・インターフェース活用
- **【必須】インターフェース型での宣言**: 実装クラスに適切なインターフェースが存在する場合は、必ずインターフェース型で変数・引数・戻り値を宣言すること。
  ```java
  // Good
  List<Entry> list = new ArrayList<>();
  Map<String, String> map = new HashMap<>();

  // Bad
  ArrayList<Entry> list = new ArrayList<>();
  HashMap<String, String> map = new HashMap<>();
  ```
- **【必須】単一責任の原則 (Single Responsibility)**: メソッドおよびクラスは単一の明確な役割のみを持たせること。処理と検証が混在するメソッド（例: `checkAndDo()`）は分割すること。
- **【必須】非推奨 (Deprecated) APIの禁止**: `@Deprecated` が付与されたAPI・メソッドは使用しないこと。
- **【必須】使われないコードの排除**: デッドコード、到達不能コード、不要なフィールド/メソッドを生成しないこと。

### 2.2 カプセル化 & 継承の制御
- **【必須】適切な可視性 (Access Modifiers)**: フィールドは原則 `private` とし、メソッドやクラスも必要最小限の公開範囲（`private` > package-private > `protected` > `public`）に設定すること。
- **【必須】`final` の積極的活用**: 継承を想定しないクラス、オーバーライドさせないメソッド、変更されない定数には `final` を付与すること。
- **【必須】フィールドの隠蔽 (Shadowing) 禁止**: 親クラスのフィールドと同名のフィールドを子クラスで再定義しないこと。
- **【必須】`@Override` の明示**: スーパークラスのメソッドやインターフェースを実装・オーバーライドする際は、必ず `@Override` アノテーションを付与すること。
- **【必須】親の `private` メソッドと同名定義の禁止**: 親クラスの `private` メソッドと同名のメソッドを子クラスで定義しないこと。
- **【推奨】インナークラス・無名クラスの制限**: 原則として1ファイル1クラスとし、インナークラスや無名クラスの乱用を避けること（ラムダ式やEnum定数固有メソッドは許可）。

---

## 3. 命名規則 (Naming Conventions)

| 対象 | 命名規則 | 形式 | 良い例 | 悪い例 |
| :--- | :--- | :--- | :--- | :--- |
| **パッケージ** | 全て小文字、意味のある単語 | lowercase | `com.example.service.order` | `com.example.service.Order` |
| **クラス / Record / Interface / Enum** | 単語の先頭を大文字 | UpperCamelCase | `UserProfile`, `OrderService`, `Rect` | `userProfile`, `order_service` |
| **メソッド** | 単語の区切りのみ大文字、動詞で開始 | lowerCamelCase | `getName()`, `calculateTotal()` | `getname()`, `CalculateTotal()` |
| **変数 / 引数 / フィールド** | 単語の区切りのみ大文字 | lowerCamelCase | `carNumber`, `itemPrice`, `thisIsIpAddress` | `car_number`, `CarNumber`, `p` |
| **定数 (`static final`) / 列挙定数** | 全て大文字、単語間をアンダースコア区切り | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `SYSTEM_NAME`, `FALL` | `maxRetryCount`, `Max_Retry` |

### 3.1 命名の詳細ルール
- **【必須】大文字・小文字のみでの区別禁止**: 大文字・小文字の違いだけで別々の変数を定義しないこと（例: `int num` と `int Num` の混在禁止）。
- **【必須】コンストラクタと同名メソッドの禁止**: クラス名と同じ名前の通常メソッドを作成しないこと。
- **【必須】変換メソッドの命名**: オブジェクト変換メソッドは `to` + 目的型名（例: `toString()`, `toArray()`, `toList()`）とすること。
- **【必須】Getter / Setter の命名**: 
  - ゲッター: `get` + 属性名（例: `getName()`）
  - `boolean` ゲッター: `is` + 属性名（例: `isOpen()`, `isValid()`）
  - セッター: `set` + 属性名（例: `setName(String name)`）
- **【必須】boolean型のメソッド/変数命名**: `true`/`false` の状態が明確にわかる命名（`isReady`, `hasPermission`, `exists` 等）とし、`flag` 等の曖昧な名前は禁止。
- **【必須】引数名とフィールド名の区別**: 引数名とインスタンス変数名を一致させないこと（※自動生成されるコンストラクタやSetter/Getterを除く）。引数名に `_` を付与して区別する記法（例: `_value`）は禁止。
- **【推奨】ローカル変数のスコープに応じた命名**: スコープが数行程度の狭い変数は簡潔な名前でも良いが、利用範囲が広い変数は省略せず明確な名前を付けること。forループカウンタは `i`, `j`, `k` を使用すること。

---

## 4. フォーマット & 構文スタイル (Formatting & Syntax)

- **【必須】ブロック `{ }` の省略禁止**: `if`, `else`, `while`, `for`, `do while` などの制御文では、処理が1行であっても必ず `{ }` を記述すること。
- **【必須】空ブロックの禁止**: 処理ステートメントのない空の `{}` ブロックを作成しないこと。
- **【必須】1行1ステートメント**: `{` の直後に同一行で処理を書いたり、1行にセミコロンで区切って複数の処理を書かないこと。
- **【必須】空白（スペース）の配置ルール**:
  - カンマ（`,`）および for文のセミコロン（`;`）の後には半角空白を1文字入れる。
  - 代入演算子（`=`, `+=` 等）、ビット演算子、論理演算子（`&&`, `||`）、関係演算子（`==`, `<`, `>=` 等）、算術演算子（`+`, `-`, `*`, `/`, `%`）の前後には空白を入れる。
  - 単項演算子（`++`, `--`, `!`）とオペランドの間には空白を入れない。
- **【必須】return文の括弧の排除**: 単純な式を返す return 文で冗長な外側カッコ `()` を付けないこと（例: `return answer;` ○ / `return (answer);` ✕）。
- **【必須】boolean値の冗長比較禁止**: `if (isValid == true)` や `if (hasItem == false)` と書かず、`if (isValid)` や `if (!hasItem)` と記述すること。
- **【推奨】不等号の向きの統一**: 不等号は左向き（`<`, `<=`）に統一して記述すること（例: `if (min <= x && x <= max)`）。
- **【必須】1ステートメント1変数宣言**: `int a, b, c;` のようにカンマで複数の変数をまとめて宣言しないこと。
- **【必須】Java標準の配列宣言記法**: `int[] array` と記述し、C言語スタイル `int array[]` は禁止。

---

## 5. 変数・型・スコープ (Variables, Types & Scope)

### 5.1 変数のスコープと再利用
- **【必須】ローカル変数の宣言位置**: 利用する直前で宣言し、スコープを最小限に抑えること。
- **【必須】変数の使い回し禁止**: 1つの変数を異なる目的で再利用せず、役割ごとに新しい変数を定義して初期化すること。
- **【必須】引数への再代入禁止**: メソッドの引数に新しい値を再代入しないこと（引数は `final` または実質的finalとして扱う）。
- **【推奨】実質的 final (Effectively Final) の活用**: 変数宣言への明示的な `final` 付与は省略可能だが、値の変更を行わない実質的finalを基本とすること。

### 5.2 マジックナンバー排除 & 定数定義
- **【必須】リテラルの定数化**: `-1`, `0`, `1` などの自明なループ初期値を除き、数値や文字列リテラルは `private static final` 等の定数として定義すること。定数名はその値の意味を表す命名にすること（例: `private static final int ZERO = 0;` は禁止）。
- **【必須】定数の不変性 (Immutability)**:
  - 定数はプリミティブ型または不変オブジェクトで定義すること。
  - `public static final` で可変配列を定義して公開しないこと（`List.of()` による不変リストに置き換える）。
  - 不変コレクション定数の初期化には `List.of()`, `Set.of()`, `Map.of()` を使用すること。

### 5.3 型推論 `var` の適用基準
- **【推奨】右辺から型が明白な場合に `var` を使用**:
  ```java
  // Good: 右辺で型が自明
  var name = "Alice";
  var list = new ArrayList<String>();
  var items = List.of("A", "B");
  var user = (UserProfile) obj;

  // Bad: メソッド戻り値など右辺から型が推測困難な場合
  var data = service.fetch(); // 型が不明瞭になるため避ける
  ```

---

## 6. 制御構文 & パターンマッチング (Control Flow & Pattern Matching)

### 6.1 switch式 & switch文
- **【必須】switch式のアロー構文 (`->`)**: switch式ではフォールスルー事故を防ぐため、常にアロー構文を使用すること。
- **【必須】`yield` / 中括弧 `{}` の省略**: アロー構文で1式で書ける場合は、`{}` および `yield` を省略すること。
- **【必須】網羅されたEnumの `default` 句排除**: 全列挙定数が case 句で網羅されている場合、デッドコードとなる `default` 句は記述しないこと。
- **【推奨】if-else代入からswitch式への置換**: 条件分岐で1つの変数に値を代入する場合は、if-else文ではなくswitch式を使用して実質的finalとすること。
  ```java
  // Good
  var discount = switch (customerType) {
      case VIP -> 0.20;
      case REGULAR -> 0.05;
      case GUEST -> 0.0;
  };
  ```

### 6.2 パターンマッチング & 型判定
- **【必須】クラス名文字列での型比較禁止**: `o.getClass().getName().equals("...")` による判定は禁止。
- **【必須】instanceof パターンマッチングの使用**: 型チェックとキャストを個別に行わず、パターンマッチング構文を使用すること。
  ```java
  // Good
  if (obj instanceof String s) {
      log.info(s.toLowerCase());
  }

  // Bad
  if (obj instanceof String) {
      String s = (String) obj;
      log.info(s.toLowerCase());
  }
  ```
- **【推奨】switchパターンマッチングの活用**: 複数の型による分岐は `instanceof` の if-else チェーンではなく switch パターンマッチングを使用すること。
  ```java
  // Good
  String formatted = switch (obj) {
      case Integer i -> String.format("int %d", i);
      case Long l    -> String.format("long %d", l);
      case Double d  -> String.format("double %f", d);
      case String s  -> String.format("String %s", s);
      default        -> Objects.toString(obj);
  };
  ```

### 6.3 ループ処理
- **【必須】全要素走査での拡張for文**: 配列やコレクションの全要素を走査する場合は拡張for文を使用すること。
- **【必須】ループ内でのループ変数の改変禁止**: forループ内でカウンタ変数を手動加算したり、拡張for文の要素変数に再代入しないこと。
- **【必須】配列のコピー**: 配列のコピーには `Arrays.copyOf()` を使用すること。

---

## 7. 文字列 & 数値 & 日付操作 (Strings, Numbers & Dates)

### 7.1 文字列操作
- **【必須】文字列の一致比較**: 必ず `equals()` を使用すること（`==` 禁止）。定数・リテラルと比較する場合は `定数.equals(変数)` の形式でNullPointerExceptionを防ぐこと。
- **【必須】文字列リテラルのインスタンス化禁止**: `new String("...")` は使用しないこと。
- **【必須】ループ内文字列連結**: ループ内の文字列連結には `+` 演算子を使用せず、`StringBuilder`、`StringJoiner`、または `String.join()` を使用すること。
- **【必須】単一ステートメントでの連結**: 単一ステートメントでの連結（`return a + b + c;`）には `StringBuilder` を明示せず `+` 演算子を使用すること（コンパイラが最適化する）。
- **【必須】型変換**: プリミティブ型からの文字列変換には `String.valueOf()` を使用すること。

### 7.2 数値計算 (`BigDecimal`)
- **【必須】正確な計算での `BigDecimal` 使用**: 金額計算や丸め誤差が許されない数値計算では、浮動小数点数（`float`, `double`）を避け `BigDecimal` を使用すること。
- **【必須】`BigDecimal` の値比較**: スケール違いによる誤判定を防ぐため、`equals()` ではなく `compareTo() == 0` を使用すること。
- **【必須】`BigDecimal` の文字列表現**: 指数表記を避けるため、`toString()` ではなく `toPlainString()` を使用すること。
- **【推奨】`BigDecimal` のゼロ・符号判定**: `compareTo(BigDecimal.ZERO)` よりも効率的な `signum() == 0` を優先すること。

### 7.3 日付・時間
- **【推奨】Modern Date and Time API**: Java 8以降の `java.time` パッケージ（`LocalDate`, `LocalDateTime`, `DateTimeFormatter` 等）を使用すること。

---

## 8. コレクション & Stream API & Optional

### 8.1 コレクション (Collections)
- **【必須】レガシーコレクションの禁止**: `Vector`, `Hashtable`, `Enumeration` は使用せず、`List`, `Map`, `Set`, `Iterator` を使用すること。
- **【必須】ジェネリクスの明示**: コレクション宣言時は必ず型引数を指定すること（Raw型禁止）。
- **【必須】不変リストの生成**: 固定要素のList生成には `Arrays.asList()` ではなく `List.of()` を使用すること。
- **【必須】空コレクション/配列の返却**: メソッドの戻り値としてコレクションや配列を返す場合、要素数0のときは `null` を返さず、空のコレクション（`Collections.emptyList()`, `List.of()`）やサイズ0の配列を返すこと。
- **【必須】Listのソート**: `Collections.sort(list)` ではなく `list.sort(comparator)` を使用すること。
- **【推奨】`Collection.forEach()` の制限**: 複数行のラムダを渡す `Collection.forEach()` はデバッグが困難なため避け、拡張for文を使用すること。ただし `list.forEach(this::process)` のようにメソッド参照で1行で完結する場合は許可する。
- **【推奨】順序保持コレクション (Sequenced Collections)**: Java 21以降で順序を持つコレクションの両端操作を行う際は、`getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()` などの専用APIを使用すること。

### 8.2 ラムダ式 & メソッド参照
- **【必須】メソッド参照の優先**: ラムダ式が単一のメソッド呼び出しである場合は、メソッド参照（例: `String::compareToIgnoreCase`, `BigDecimal::add`）を使用すること。
- **【必須】型推論の活用**: ラムダ式の引数型宣言は省略すること（例: `(s1, s2) -> s1 + s2` ○ / `(String s1, String s2) -> s1 + s2` ✕）。
- **【必須】1行原則**: ラムダ式は原則1行とし、中括弧 `{}` と `return` を省略すること。複数行にわたる処理は `private` メソッドに切り出してメソッド参照で呼び出すこと。

### 8.3 Stream API
- **【必須】並列ストリームの原則禁止**: `parallelStream()` および `.parallel()` はスレッドプール競合やオーバーヘッドの原因となるため原則禁止。
- **【必須】改行位置**: Stream APIのメソッドチェーンは、各中間処理・終端処理のドット（`.`）の直前で改行すること。
- **【必須】Streamインスタンスの変数再利用禁止**: Streamは一度終端処理を行うと消費されるため、変数に代入して複数の処理で再利用しないこと。
- **【推奨】中間処理の行数制限**: Streamの中間処理は3つ（3行）程度を目安とし、過度に複雑なチェーンは避けること。処理途中にインラインコメントを挟まないこと。

### 8.4 Optional
- **【必須】値取り出し時の変数代入回避**: 同一メソッド内で直ちに値を取り出す場合は、Optional型の変数に代入せずチェーンして取得すること。
  ```java
  // Good
  Employee employee = findEmployee(id)
          .orElseThrow(() -> new IllegalArgumentException("Not found: " + id));

  // Bad
  Optional<Employee> employeeOpt = findEmployee(id);
  Employee employee = employeeOpt.orElseThrow(...);
  ```

---

## 9. モダンJava仕様 (Modern Java Features)

### 9.1 レコード (Record)
- **【推奨】不変データキャリアへのRecord適用**: データの保持を主目的とするクラスは `record` を積極的に活用すること。
- **【必須】アクセサの上書き禁止**: Recordが自動生成するゲッターアクセサを不要にオーバーライドしないこと。
- **【推奨】レコードパターンによる分解**: レコードの型チェックとフィールド抽出にはレコードパターンを使用すること。
  ```java
  // Good
  if (obj instanceof Point(int x, int y)) {
      log.info("X: {}, Y: {}", x, y);
  }
  ```

### 9.2 シールクラス (Sealed Classes)
- **【状況依存】ドメイン制約と網羅性確保**: 継承階層を厳密に制限し、パターンマッチングの網羅性チェックを活かす場合に `sealed` インターフェース/クラスを活用すること。

### 9.3 テキストブロック (Text Blocks)
- **【必須】複数行文字列での適用**: SQL、JSON、HTML等の複数行文字列リテラルには、文字列結合（`+`）ではなくテキストブロック（`"""`）を使用すること。
- **【必須】単一行でのテキストブロック禁止**: 単一行の文字列には通常の文字列リテラル（`"..."`）を使用すること（二重引用符のエスケープ回避目的を除く）。
- **【必須】インデントと配置**: 開始 `"""` は行末に置き、テキストブロックのインデントは周辺コードのインデントに揃えること。複雑なメソッド引数内にインラインで巨大なテキストブロックを直接記述せず、一度ローカル変数や定数に代入すること。

---

## 10. リソース管理 & 例外処理 (Resource Management & Exceptions)

### 10.1 リソース管理
- **【必須】try-with-resources 文の使用**: `InputStream`, `OutputStream`, `Connection`, `Reader`, `Writer` など `AutoCloseable` / `Closeable` を実装したリソースは、必ず try-with-resources 文で確実に解放すること（手動の `close()` 呼び出しは禁止）。
- **【必須】解放が必要な自作クラス**: 明示的な終了処理が必要なクラスを作成する場合は、`AutoCloseable` を実装すること。

### 10.2 例外処理
- **【必須】具体的な例外型の catch**: `catch (Exception e)` や `catch (Throwable t)` などの広すぎる例外キャッチを行わず、発生しうる具体的な例外クラス（`IOException`, `SQLException` 等）を指定すること。
- **【必須】汎用 `Exception` の throw 禁止**: `throw new Exception();` のように汎用例外を直接インスタンス化してスローしないこと。
- **【必須】空 catch ブロックの禁止**: 例外をもみ消す空の `catch` ブロックは禁止。仕様上どうしても処理をスキップする場合は、ブロック内に `// ignore` と明記すること。
- **【必須】`finalize()` の絶対禁止**: `finalize()` メソッドのオーバーライドおよび呼び出しは絶対に行わないこと。

---

## 11. メンバー記述順序 & Javadoc・コメント規約

### 11.1 クラスメンバーの記述順序
クラス内のメンバーは以下の順序で配置すること：
1. `static` フィールド（定数含む）
2. `static` イニシャライザ
3. `static` メソッド
4. インスタンスフィールド
5. インスタンスイニシャライザ
6. コンストラクタ
7. インスタンスメソッド

※ 各カテゴリ内での可視性順: `public` $\rightarrow$ `protected` $\rightarrow$ package-private $\rightarrow$ `private`

### 11.2 Javadoc & コメント
- **【必須】Javadocの必須タグ**:
  - クラス/インターフェース: クラスの概要説明
  - メソッド: メソッドの概要説明、`@param`, `@return`, `@throws` (または `@exception`)
- **【必須】Javadocの不要タグ**:
  - クラス/インターフェース: `@author`, `@version`は書かないこと。存在したら除去すること
- **【必須】自明なコメント・不要コメントの排除**:
  - コードを見れば自明な内容（例: `// 名前を取得する` -> `getName()`）は書かないこと。
  - バージョン管理システムで追跡できる変更履歴や、コメントアウトされた古いコードを残さないこと。
- **【必須】「Why（理由・意図）」の記述**: コメントには処理の内容（What）ではなく、なぜその処理が必要なのか（Why、ビジネスルール、前提条件、副作用）を記述すること。
- **【推奨】サンプルコードの `{@snippet}`**: Javadoc内にサンプルコードを記載する場合は `{@snippet ...}` タグを使用すること。
- **【推奨】TODOコメント形式**: 実装待ち・確認待ちのTODOコメントは `// TODO: 内容 (関連チケット/課題)` の形式で記述すること。

