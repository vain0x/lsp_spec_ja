# Language Server Protocol Specification - 3.14
以下の文書は [Language Server Protocol Specification - 3.14](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/) の日本語訳である。

<a rel="license" href="http://creativecommons.org/licenses/by/3.0/"><img alt="クリエイティブ・コモンズ・ライセンス" style="border-width:0" src="https://i.creativecommons.org/l/by/3.0/88x31.png" /></a>

## Base Protocol
ベースプロトコルはヘッダ部とコンテント部で構成される(HTTP と似ている)。ヘッダ部
とコンテント部は `\r\n` で分割される。

### Header Part
ヘッダ部はヘッダフィールドで構成される。それぞれのヘッダフィールドは `: `(コロ
ンとスペース) で分割された名前と値で構成される。それぞれのヘッダフィールドは
`\r\n` で終了する。最後のヘッダフィールドとヘッダ全体はそれぞれ `\r\n` で終了す
ることと、少なくとも1つのヘッダが必須であることを踏まえると、常にメッセージのコ
ンテント部のすぐ前には2つの `\r\n` があることになる。

現在次のヘッダフィールドがサポートされている:

| ヘッダフィールド名 | 値の型 | 備考 |
| ------------------ | ------ | ---- |
| Content-Length | number | コンテント部のバイト数。このヘッダは必須 |
| Content-Type | string | コンテント部の mime type。デフォルトは `application/vscode-jsonrpc; charset=utf-8` |

ヘッダ部は 'ascii' でエンコードされる。これはヘッダ部とコンテント部を分割する
`\r\n` も含む。

### Content Part
実際のメッセージの中身が含まれる。メッセージのコンテント部にはリクエスト、レス
ポンス、通知を記述するために [JSON-RPC](https://www.jsonrpc.org/) が使われる。
コンテント部は `Content-Type` フィールドで与えられる `charset` でエンコードされ
る。デフォルトでは `utf-8` で今はそれしかサポートされていない。サーバまたはクラ
イアントが `utf-8` 以外の指定をヘッダでされた場合は、エラーを返すべきである。

(以前のバージョンでは `utf8` が使われていたが、これは[仕
様](http://www.iana.org/assignments/character-sets/character-sets.xhtml)による
と正しいエンコード形式名ではない。) 後方互換性のため、サーバ、クライアント双方
は `utf8` を `utf-8` として扱うことを強く推奨する。

### Example:
```
Content-Length: ...\r\n
\r\n
{
	"jsonrpc": "2.0",
	"id": 1,
	"method": "textDocument/didOpen",
	"params": {
		...
	}
}
```

### Base Protocol JSON structures
次の TypeScript 定義はベースとなる [JSON-RPC
protocol](https://www.jsonrpc.org/specification) を記述する。

#### Abstract Message
メッセージは JSON-RPC として定義される。LSP では常に `jsonrpc` バージョンとして
"2.0" を使う。

```ts
interface Message {
	jsonrpc: string;
}
```

#### Request Message
リクエストメッセージはクライアントサーバ間のリクエストを記述する。全ての処理さ
れたリクエストは必ず送信元にレスポンスを返さなければならない。

```ts
interface RequestMessage extends Message {

	/**
	 * リクエストID。
	 */
	id: number | string;

	/**
	 * 実行されるメソッド。
	 */
	method: string;

	/**
	 * メソッドのパラメータ。
	 */
	params?: Array<any> | object;
}
```

#### Response Message
レスポンスメッセージはリクエストの結果として送信される。リクエストが結果を提供
しない場合でもリクエスト受信者は JSON RPC 仕様に従うためにレスポンスメッセージ
を返す必要がある。この場合、`ResponseMessage` の `result` プロパティはリクエス
トの成功を伝えるために `null` を入れるべきである。

```ts
interface ResponseMessage extends Message {
	/**
	 * リクエストID。
	 */
	id: number | string | null;

	/**
	 * リクエストの結果。成功時は必須である。
	 * メソッドがエラーを返した場合は空でなければならない。
	 */
	result?: string | number | boolean | object | null;

	/**
	 * リクエストが失敗した場合のエラー。
	 */
	error?: ResponseError<any>;
}

interface ResponseError<D> {
	/**
	 * 発生したエラー種別を表す数字。
	 */
	code: number;

	/**
	 * エラーの概要。
	 */
	message: string;

	/**
	 * エラーについての情報を不可するプリミティブまたは構造化された値。
	 * 省略可能。
	 */
	data?: D;
}

export namespace ErrorCodes {
	// JSON RPC で定義されたもの。
	export const ParseError: number = -32700;
	export const InvalidRequest: number = -32600;
	export const MethodNotFound: number = -32601;
	export const InvalidParams: number = -32602;
	export const InternalError: number = -32603;
	export const serverErrorStart: number = -32099;
	export const serverErrorEnd: number = -32000;
	export const ServerNotInitialized: number = -32002;
	export const UnknownErrorCode: number = -32001;

	// このプロトコルで定義されたもの。
	export const RequestCancelled: number = -32800;
	export const ContentModified: number = -32801;
}
```

#### Notification Message
通知メッセージ。処理された通知メッセージはレスポンスを返してはならない。これら
はイベントのように振る舞う

```ts
interface NotificationMessage extends Message {
	/**
	 * 実行されるメソッド。
	 */
	method: string;

	/**
	 * 通知のパラメータ。
	 */
	params?: Array<any> | object;
}
```

#### $ Notifications and Requests
`$/` で始まるメソッドの通知とリクエストはプロトコル実装に依存するメッセージで、
全てのクライアントまたはサーバでは実装できないかもしれない。例えばシングルスレッ
ドで同期的なプログラミング言語を用いてサーバを実装する場合、`$/cancelRequest`
通知に対してサーバができることはほとんどない。サーバまたはクライアントが `$/`
から始まる通知を受けた場合、通知を無視することは自由である。サーバまたはクライ
アントが `$/` から始まるリクエストを受けた場合、エラーコード
`MethodNotFound`(`-32601`) でリクエストをエラーにしなければならない。

#### Cancellation Support
ベースプロトコルはリクエストのキャンセル機能を提供する。リクエストをキャンセルするためには、次のプロパティを通知メッセージとして送信する:

*通知*:
* メソッド: `$/cancelRequest`
* パラメータ: 以下で定義される `CancelParams`:

```ts
interface CancelParams {
	/**
	 * キャンセル対象のリクエストID。
	 */
	id: number | string;
}
```

キャンセルされたリクエストはサーバから戻り、レスポンスを返す必要がある。
open/hanging 状態のままにはできない。これは全てのリクエストはレスポンスを返すと
いう JSON RPC プロトコルに従っている。加えて、これはキャンセル時に部分的な結果
を返すことも許容している。リクエストがキャンセル時にエラーを返す場合、エラーコー
ドは `ErrorCodes.RequestCancelled` を指定することを推奨する。

## Language Server Protocol
LSP は前述のベースプロトコルでやりとりされる JSON-RPC リクエスト、レスポンス、
通知メッセージの集まりとして定義される。このセクションではこのプロトコルで使用
される基本的な JSON 構造の記述から始まる。これらの記述には TypeScript interface
を用いる。基本的な JSON 構造を基に、実際のリクエストとそれに伴うレスポンス、通
知が記述される。

一般的に、LSP は JSON-RPC メッセージをサポートするが、ここで定義されたベースプ
ロトコルはリクエストおよび通知メッセージに渡すパラメータは(渡されるのであれば)
`object` 型であるべきという慣習に従っている。しかしながら、カスタムメッセージ内
で `Array` パラメータを使うことを禁止してはいない。

今はこのプロトコルは一つのサーバが一つのツールを提供することを仮定している。一
つのサーバを異なるツールで共有することをサポートしていない。このような共有をす
るにはプロトコルの追加が必要である。例えば並列に編集することをサポートするため、
ドキュメントのロックを取れるようにする。

### Basic JSON Structures
#### URI
URI は文字列として送信される。URI フォーマットは [https://tools.ietf.org/html/rfc3986](https://tools.ietf.org/html/rfc3986) で定義される。

```
  foo://example.com:8042/over/there?name=ferret#nose
  \_/   \______________/\_________/ \_________/ \__/
   |           |            |            |        |
scheme     authority       path        query   fragment
   |   _____________________|__
  / \ /                        \
  urn:example:animal:ferret:nose
```

文字列を `scheme`、`authority`、`path`、`query`、`fragment` の URI コンポーネン
トにパースするための node module もメンテナンスする。GitHub リポジトリは
[https://github.com/Microsoft/vscode-uri](https://github.com/Microsoft/vscode-uri)
で、npm module は
[https://www.npmjs.com/package/vscode-uri](https://www.npmjs.com/package/vscode-uri)
にある。

多くのインターフェイスはドキュメントの URI に対応するフィールドを持つ。これを明
確にするため、このようなフィールドの型を `DocumentUri` として宣言する。ネット
ワークを通して URI は文字列として転送されるが、これにより、文字列のコンテンツが
有効な URI としてパースされることが保証される。

```ts
type DocumentUri = string;
```

#### Text Documents
現在のプロトコルは文字列として表現できるコンテントからなるテキストドキュメント
用に合わせられている。バイナリドキュメントはサポートされていない。ドキュメント
内の位置(後述の `Position` の定義を参照)は0始まりの行と文字オフセットとして表さ
れる。オフセットは UTF-16 文字表現を基にする。そのため文字列 `a𐐀b` を考えると、
UTF-16 で `𐐀` は2文字単位として表現されるため、文字 `a` のオフセットは0、`𐐀` の
オフセットは1、`b` のオフセットは3となる。クライアントとサーバ双方が文字列を同
じ行表現に分割することを保証するために、LSP では `\n`、`\r\n`、`\r` という EOL
文字列を指定する。

位置は行末文字に依存しない。そのため、`|` が文字オフセットを表すとして、`\r|\n`
や `\n|` のような位置は指定することができない。

```ts
export const EOL: string[] = ['\n', '\r\n', '\r'];
```

#### Position
テキストドキュメント内の位置は0始まりの行と0始まりの文字オフセットで表現される。
位置はエディタの "insert" カーソルのようにふたつの文字の間にある。`-1` で行末を
表わす、などの特別な値はサポートされていない。

```ts
interface Position {
	/**
	 * ドキュメント内の行位置 (0始まり)。
	 */
	line: number;

	/**
	 * ドキュメント内の行中の文字オフセット (0始まり)。行は文字列として表現される
	 * ことを仮定し、`character` の値は `character` と `caracter + 1` 間のすき間
	 * を表現する。
	 * `character` の値が行の長さを越えた場合はデフォルトでは行の長さに戻る。
	 */
	character: number;
}
```

#### Range
テキストドキュメント内のレンジは開始位置(0始まり)と終了位置で表現される。レンジはエディタの選択範囲に相当する。よって終了位置は含まれない。行末文字を含むように行をレンジに指定したい場合は終了位置を次の行の始めに指定する。例えば:

```json
{
    "start": { "line": 5, "character": 23 },
    "end": { "line": 6, "character": 0 }
}
```

```ts
interface Range {
	/**
	 * レンジの開始位置。
	 */
	start: Position;

	/**
	 * レンジの終了位置。
	 */
	end: Position;
}
```

#### Location
テキストファイル内の行などのリソース内の場所を表わす。

```ts
interface Location {
	uri: DocumentUri;
	range: Range;
}
```

#### LocationLink
ソースとターゲット間の位置のリンクを表わす。

```ts
interface LocationLink {

	/**
	 * このリンクのソースのスパン
	 *
	 * マウス操作の下線付きスパンとして使われる。デフォルトではマウス位置の単語の
	 * レンジが使われる。
	 */
	originSelectionRange?: Range;

	/**
	 * このリンクのターゲットリソースの識別子。
	 */
	targetUri: DocumentUri;

	/**
	 * このリンクのターゲットレンジ。ターゲットが例えばシンボルの場合、ターゲット
	 * レンジはこのシンボルを含む範囲で、先頭/末尾の空白は含まれないが、コメント
	 * など他のものは全て含まれる。これは通常エディタでレンジをハイライトするとき
	 * に使われる。
	 */
	targetRange: Range;

	/**
	 * このリンクを辿るときに選択し、表示する範囲。例えば関数名。
	 * `targetRange` に含まれている必要がある。`DocumentSymbol#range` も参照。
	 */
	targetSelectionRange: Range;
}
```

#### Diagnostic
コンパイラのエラーや注意などの診断結果を表わす。`Diagnostic` はリソース内でのみ
有効。

```ts
interface Diagnostic {
	/**
	 * メッセージが適用されるレンジ。
	 */
	range: Range;

	/**
	 * 診断結果の重大度。省略可能。省略した場合、診断結果をエラー、注意、情報、ヒ
	 * ントとして解釈するのはクライアント次第である。
	 */
	severity?: number;

	/**
	 * UI に表示されるであろう診断結果コード。
	 */
	code?: number | string;

	/**
	 * 'typescript' や 'super lint' のようなこの診断結果を提供したソースを表わす
	 * 可読な文字列。
	 */
	source?: string;

	/**
	 * 診断結果のメッセージ。
	 */
	message: string;

	/**
	 * 関連する診断情報の配列、例えばスコープ内のシンボル名が衝突した場合このプロ
	 * パティから全ての定義をマークできる。
	 */
	relatedInformation?: DiagnosticRelatedInformation[];
}
```

LSP は次の重大度をサポートしている:
```ts
namespace DiagnosticSeverity {
	/**
	 * エラーを報告する。
	 */
	export const Error = 1;
	/**
	 * 注意を報告する。
	 */
	export const Warning = 2;
	/**
	 * 情報を報告する。
	 */
	export const Information = 3;
	/**
	 * ヒントを報告する。
	 */
	export const Hint = 4;
}

```

```ts
/**
 * 診断結果の関連メッセージとソースコードの位置を表わす。これはスコープ内のシン
 * ボルが被っているときなど診断結果の原因やそれに関連するコードの位置を指し示す
 * ために使われるべきである。
 */
export interface DiagnosticRelatedInformation {
	/**
	 * この関連診断結果の位置。
	 */
	location: Location;

	/**
	 * この関連診断結果のメッセージ。
	 */
	message: string;
}
```

#### Command
コマンドへの参照を表わす。UI でコマンドを表示するためのタイトルを提供する。コマ
ンドは文字列の識別子によって識別される。コマンドを処理するために推奨される方法
はクライアントとサーバが対応する機能を提供する場合、サーバ側でそれらの実行を実
装することである。その代わりにツール拡張コードはコマンドを処理できる。LSP は現
状 well-known コマンドを指定しない。

```ts
interface Command {
	/**
	 * コマンドのタイトル、`save` など。
	 */
	title: string;
	/**
	 * 実際のコマンドハンドラの識別子。
	 */
	command: string;
	/**
	 * コマンドハンドラに渡すべき引数。
	 */
	arguments?: any[];
}
```

#### TextEdit
テキストドキュメントに適用される編集。

```ts
interface TextEdit {
	/**
	 * 操作されるテキストドキュメントのレンジ。ドキュメントにテキストを挿入するた
	 * めにはレンジを start === end として指定する。
	 */
	range: Range;

	/**
	 * 挿入される文字列。削除操作をするためには空文字列を使う。
	 */
	newText: string;
}
```

#### TextEdit[]
複雑なテキスト操作は、ドキュメントへの1つの変更を表わす `TextEdit` の配列で記述
される。

全てのテキスト編集のレンジは、元のドキュメント内の位置を参照する。テキスト編集
のレンジは重なってはいけない、つまり元のドキュメントのどの部分も1つの編集でしか
操作されない。しかし、複数挿入や単一の削除または置換の後の複数挿入など、複数の
編集が同一の開始位置を持つことは可能である。複数の挿入が同一の位置を持つ場合、
配列の順序が結果のテキストに挿入された文字列が反映される順序を定義する。

#### TextDocumentEdit
単一のテキストドキュメント上の変更を記述する。テキストドドキュメントはこの編集
が適用される前のテキストドキュメントのバージョンを確認できるようにするために
`VersionedTextDocumentIdentifier` として参照される。`TextDocumentEdit` はバー
ジョン `Si` の全ての変更を記述し、バージョン `Si + 1` に移動する。
そのため `TextDocumentEdit` の作成者は配列のソートやその他の順序付けをする必要
はない。ただし編集は重なってはいけない。

```
export interface TextDocumentEdit {
	/**
	 * 変更するテキストドキュメント。
	 */
	textDocument: VersionedTextDocumentIdentifier;

	/**
	 * 適用する変更。
	 */
	edits: TextEdit[];
}
```

### File Resource changes
バージョン 3.13 から:

ファイルリソース変更によりサーバはクライアントを通してファイルおよびフォルダを
作成、リネーム、削除できる。この名前(訳注: ファイルリソース変更)はファイルにつ
いて述べるが、操作はファイルとフォルダに対して動作することに注意する。これは
LSP の他の名付けでも見られる(ファイルとフォルダを監視することができる File
watchers などを参照)。対応する変更リテラルは次のようになる:

```ts
/**
 * ファイル作成時のオプション。
 */
export interface CreateFileOptions {
	/**
	 * 存在するファイルを上書く。`ignoreIfExists` より優先される。
	 */
	overwrite?: boolean;
	/**
	 * 存在する場合は何もしない。
	 */
	ignoreIfExists?: boolean;
}

/**
 * ファイル作成操作
 */
export interface CreateFile {
	/**
	 * 作成
	 */
	kind: 'create';
	/**
	 * 作成するリソース。
	 */
	uri: DocumentUri;
	/**
	 * オプション
	 */
	options?: CreateFileOptions;
}

/**
 * ファイルリネームオプション
 */
export interface RenameFileOptions {
	/**
	 * 存在するファイルを上書く。`ignoreIfExists` より優先される。
	 */
	overwrite?: boolean;
	/**
	 * 存在する場合は何もしない。
	 */
	ignoreIfExists?: boolean;
}

/**
 * リネーム操作
 */
export interface RenameFile {
	/**
	 * リネーム
	 */
	kind: 'rename';
	/**
	 * 古い(存在している)位置。The old (existing) location.
	 */
	oldUri: DocumentUri;
	/**
	 * 新しい位置。
	 */
	newUri: DocumentUri;
	/**
	 * リネームオプション。
	 */
	options?: RenameFileOptions;
}

/**
 * ファイル削除オプション
 */
export interface DeleteFileOptions {
	/**
	 * フォルダが指定されたとき、コンテントを再帰的に削除する。
	 */
	recursive?: boolean;
	/**
	 * ファイルが存在しない場合は何もしない。
	 */
	ignoreIfNotExists?: boolean;
}

/**
 * ファイル削除操作
 */
export interface DeleteFile {
	/**
	 * 削除
	 */
	kind: 'delete';
	/**
	 * 削除するファイル。
	 */
	uri: DocumentUri;
	/**
	 * 削除オプション。
	 */
	options?: DeleteFileOptions;
}
```

#### WorkspaceEdit
ワークスペース編集はワークスペース内で管理される多くのリソースへの変更を表わす。
編集操作は `changes` か `documentChanges` のどちらかを提供すべきである。クライ
アントがバージョン管理されたドキュメントへの編集を操作可能で、`documentChanges`
が指定された場合、`documentChanges` が `changes` よりも優先される。

```ts
export interface WorkspaceEdit {
	/**
	 * 存在するリソースへの変更を保持する。
	 */
	changes?: { [uri: DocumentUri]: TextEdit[]; };

	/**
	 * クライアントが `workspace.workspaceEdit.resourceOperations` を提供している
	 * かに依存してドキュメントへの変更は、それぞれのテキストドキュメントの編集が
	 * 扱う指定したバージョンのテキストドキュメントである n 個の異なるテキストド
	 * キュメントへの変更を表現するための `TextDocumentEdit` の配列または
	 * `TextDocumentEdit` の配列にファイル/フォルダへの作成、リネーム、削除操作を
	 * 混ぜたもののいずれかが使含むことができる。
	 *
	 * クライアントがバージョン管理されたドキュメントを操作できるかどうかはクライ
	 * アントが `workspace.workspaceEdit.documentChanges` を提供するかどうかで表
	 * 現される。
	 *
	 * クライアントが `documentChanges` か
	 * `workspace.workspaceEdit.resourceOperations` のいずれかをサポートしている
	 * 場合、`changes` プロパティにただの `TextEdit` の配列を使うことがサポートさ
	 * れる。
	 */
	documentChanges?: (TextDocumentEdit[] | (TextDocumentEdit | CreateFile | RenameFile | DeleteFile)[]);
}
```

#### TextDocumentIdentifier
テキストドキュメントは URI で識別される。プロトコル上では URI は文字列として渡される。対応する JSON 構造は次のようになる:

```ts
interface TextDocumentIdentifier {
	/**
	 * テキストドキュメントの URI。
	 */
	uri: DocumentUri;
}
```

#### TextDocumentItem
クライアントからサーバへ転送されるテキストドキュメントのためのアイテム。

```ts
interface TextDocumentItem {
	/**
	 * テキストドキュメントの URI。
	 */
	uri: DocumentUri;

	/**
	 * テキストドキュメントの言語識別子。
	 */
	languageId: string;

	/**
	 * このドキュメントのバージョン番号(undo/redo を含む変更後に増加する)。
	 */
	version: number;

	/**
	 * 開かれたテキストドキュメントの中身。
	 */
	text: string;
}
```

テキストドキュメントは再解釈したファイル拡張子を回避するための2つ以上の言語を扱
うときにサーバ側でドキュメントを識別するため言語識別子を持つ。ドキュメントが以
下にリストされるプログラミング言語に関連する場合、これらの ID を使うことを推奨
する。

| 言語 | 識別子 |
| ---- | ------ |
| ABAP | `abap` |
| Windows Bat | `bat` |
| BibTeX | `bibtex` |
| Clojure | `clojure` |
| Coffeescript | `coffeescript` |
| C | `c` |
| C++ | `cpp` |
| C# | `csharp` |
| CSS | `css` |
| Diff | `diff` |
| Dart | `dart` |
| Dockerfile | `dockerfile` |
| F# | `fsharp` |
| Git | `git-commit` and `git-rebase` |
| Go | `go` |
| Groovy | `groovy` |
| Handlebars | `handlebars` |
| HTML | `html` |
| Ini | `ini` |
| Java | `java` |
| JavaScript | `javascript` |
| JavaScript React | `javascriptreact` |
| JSON | `json` |
| LaTeX | `latex` |
| Less | `less` |
| Lua | `lua` |
| Makefile | `makefile` |
| Markdown | `markdown` |
| Objective-C | `objective-c` |
| Objective-C++ | `objective-cpp` |
| Perl | `perl` |
| Perl 6 | `perl6` |
| PHP | `php` |
| Powershell | `powershell` |
| Pug | `jade` |
| Python | `python` |
| R | `r` |
| Razor (cshtml) | `razor` |
| Ruby | `ruby` |
| Rust | `rust` |
| SCSS | `scss` (syntax using curly brackets), `sass` (indented syntax) |
| Scala | `scala` |
| ShaderLab | `shaderlab` |
| Shell Script (Bash) | `shellscript` |
| SQL | `sql` |
| Swift | `swift` |
| TypeScript | `typescript` |
| TypeScript React | `typescriptreact` |
| TeX | `tex` |
| Visual Basic | `vb` |
| XML | `xml` |
| XSL | `xsl` |
| YAML | `yaml` |

#### VersionedTextDocumentIdentifier
指定したバージョンのテキストドキュメントを示す識別子。

```ts
interface VersionedTextDocumentIdentifier extends TextDocumentIdentifier {
	/**
	 * このドキュメントのバージョン番号。バージョン管理されたテキストドキュメント
	 * の識別子がサーバからクライアントに送信され、エディタに開かれていない(サー
	 * バが open 通知をまだ受けていない)場合、サーバはディスクにある中身が真でバー
	 * ジョンは既知であることを示すため `null` を送信することができる(ドキュメン
	 * トコンテントの所有権で指定されている通り)。
	 * ドキュメントのバージョン番号は undo/redo を含むドキュメントへの変更で増加
	 * する。数字は連続である必要はない。
	 */
	version: number | null;
}
```

#### TextDocumentPositionParams
1.0 では `TextDocumentPosition` はインラインパラメータであった。

テキストドキュメントとその中での位置をリクエストに渡すために使われるパラメータ
リテラル。

```ts
interface TextDocumentPositionParams {
	/**
	 * テキストドキュメント。
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * テキストドキュメント内の位置。
	 */
	position: Position;
}
```

#### DocumentFilter
ドキュメントフィルタは `language`、`scheme`、`pattern`を通してドキュメントを記
述する。例としてディスク上の TypeScript ファイルを表わす。もう一つの例は
`package.json` という名前の JSON ファイルを示すフィルタである:

```json
{ "language": "typescript", "scheme": "file" }
{ "language": "json", "pattern": "**/package.json" }
```

```ts
export interface DocumentFilter {
	/**
	 * `typescript` のような言語識別子。
	 */
	language?: string;

	/**
	 * `file` や `untitled` のような URI のスキーマ(#Uri.scheme)。
	 */
	scheme?: string;

	/**
	 * `*.{ts,js}` のような glob パターン。
	 *
	 * glob パターンは次の構文を持つ:
	 * - `*` はパス区切りの中の1文字以上にマッチする
	 * - `?` はパス区切りの中の1文字にマッチする
	 * - `**` はパス区切りが無いことも含む任意の数のパス区切りにマッチする
	 * - `{}` は条件のグループ化(例えば、`**/*.{ts,js}` は全ての TypeScript と JavaScript ファイルにマッチする)
	 * - `[]` はパス区切りの中のマッチする文字の範囲を示す(例えば、`example.[0-9]` は `example.0`、`example.1`、… にマッチする)
	 * - `[!...]` はパス区切りの中のマッチしない文字の範囲を示す(例えば、`example.[!0-9]` は`example.a`、`example.b`、にはマッチするが `example.0` にはマッチしない)
	 */
	pattern?: string;
}
```

ドキュメントセレクタは1つ以上のドキュメントフィルタを組み合わせたものである。

```ts
export type DocumentSelector = DocumentFilter[];
```

#### MarkupContent
`MarkupContent` リテラルは異なるフォーマットで表わすことのできるコンテントの文
字列値を表わす。現在は `plaintext` と `markdown` がサポートされるフォーマットで
ある。`MarkupContent` は大抵 `CompletionItem` か `SignatureInformation` の結果
のプロパティとして使われる。

```ts
/**
 * クライアントがサポートする `Hover` や `ParameterInfo` や `CompletionItem` な
 * どのさまざまな結果のコンテントタイプを表わす。
 *
 * `MarkupKinds` は `$` から始まってはいけないことに注意する。これは内部的な用
 * 途のために予約されている。
 */
export namespace MarkupKind {
	/**
	 * plaintext はコンテントフォーマットとしてサポートされている
	 */
	export const PlainText: 'plaintext' = 'plaintext';

	/**
	 * markdown はコンテントフォーマットとしてサポートされている
	 */
	export const Markdown: 'markdown' = 'markdown';
}
export type MarkupKind = 'plaintext' | 'markdown';

/**
 * `MarkupContent` リテラルはその種別フラグで処理される中身の文字列値を表わす。
 * 現在はマークアップ種別として `plaintext` と `markdown` がサポートされている。
 *
 * 種別が `markdown` の場合値には GitHub issue のようなコードブロックを含めるこ
 * とができる。
 * https://help.github.com/articles/creating-and-highlighting-code-blocks/#syntax-highlighting を参照。
 *
 * JavaScript/TypeScript を用いてこのような文字列を構成する方法の例を示す:
 * ```typescript
 * let markdown: MarkdownContent = {
 *  kind: MarkupKind.Markdown,
 *	value: [
 *		'# Header',
 *		'Some text',
 *		'```typescript',
 *		'someCode();',
 *		'```'
 *	].join('\n')
 * };
 * ```
 *
 * *注意* クライアントは返ってきた markdown をサニタイズするかもしれない。クラ
 * イアントはスクリプト実行を避けるために markdown から HTML を削除することを決
 * 定できる。
 */
export interface MarkupContent {
	/**
	 * マークアップ種別
	 */
	kind: MarkupKind;

	/**
	 * コンテンツ
	 */
	value: string;
}
```

### Actual Protocol
このセクションは実際の LSP のドキュメントである。次のフォーマットに従う:

* リクエストのヘッダ
* *リクエスト:* セクションは送信するリクエストのフォーマットを記述する。メソッドは TypeScript インターフェイスを使用してパラメータが記述されるリクエストを識別する。
* *レスポンス:* セクションはレスポンスのフォーマットを記述する。結果は成功時に返すデータを示す。`error.data` はエラー時に返すデータを示す。失敗時はすでに `error.code` と `error.message` フィールドがすでに含まれていることを覚えておくこと。これらのフィールドは特定のエラーコードまたはメッセージの使用を強制する場合にのみ指定される。サーバが自由に指定することを決定できる場合、ここには提示されない。
* *登録オプション* セクションはリクエストまたは通知が動的な機能登録をサポートする場合の登録オプションを記述する。

#### Request, Notification and Response ordering
リクエストのレスポンスはだいたいサーバまたはクライアント側にリクエストが届いた
順番に送信されるべきである。例えばサーバが `textDocument/completion` リクエスト
を受けてから `textDocument/signatureHelp` を受けた場合、たいてい
`textDocument/completion` のレスポンスから返され、その後
`textDocument/signatureHelp` のレスポンスが返される。

ただし、サーバは並列実行するかもしれないし、リクエストを受けた順と異なる順でレ
スポンスを返したいかもしれない。サーバはレスポンスの正確さに影響しない限りこの
ような順序変更を行うことができる。例えば `textDocument` と
`textDocument/signatureHelp` は大抵出力順序が影響しないため、順序交換は可能であ
る。一方 `textDocument/definition` と `textDocument/rename` は後者の実行が前者
に影響するため、恐らくサーバは順序を交換すべきではない。

#### Server lifetime
現在のプロトコル使用ではサーバのライフサイクルはクライアント(例えば VS Code や
Emacs のようなツール)によって管理されるように定義している。クライアント次第で
サーバ(プロセス)の起動やシャットダウンが決定される。

#### Initialize Request
`Initialize` リクエストはクライアントからサーバへ送られる最初のリクエストであ
る。サーバがリクエストまたは通知を `initialize` リクエストより前に受けた場合は
次のように振る舞うべきである:

* リクエストに対し `code: -32002` であるエラーをレスポンスとして返すべきである。メッセージはサーバが選択することができる。
* 通知は `exit` 通知を除き、無視するべきである。`initialize` リクエストを受けることなくサーバは終了することができる。

サーバが `initialize` リクエストに `InitializeResult` で応答するまで、クライア
ントはサーバに追加のリクエスト及び通知を送信してはならない。加えて、サーバは
`InitializeResult` を応答するまでリクエストまたは通知をクライアントに送信しては
ならない、例外として `initilize` リクエストの処理中はサーバは
`window/showMessage`、`window/logMessage` と `telemetry/event` 通知や
`window/showMessageRequest` リクエストをクライアントに送信することができる。

`initialize` リクエストは一度だけ送信される。

*リクエスト:*

* メソッド: `initialize`
* パラメータ: 次のように定義される `InitializedParams`:

```ts
interface InitializeParams {
	/**
	 * サーバを起動した親プロセスのプロセス ID。プロセスが他のプロセスから起動さ
	 * れていない場合は null となる。
	 * 親プロセスが死んでいる場合は、サーバは自身のプロセスを終了すべきである
	 * (`exit` 通知を参照)。
	 */
	processId: number | null;

	/**
   * ワークスペースの `rootPath`。フォルダを開いていない場合は null となる。
	 *
	 * @deprecated rootUri を用いる
	 */
	rootPath?: string | null;

	/**
	 * ワークスペースの `rootUri`。フォルダを開いていない場合は null となる。
	 * `rootPath` と `rootUri` が指定されている場合は `rootUri` が優先される。
	 */
	rootUri: DocumentUri | null;

	/**
	 * ユーザが提供する初期化オプション。
	 */
	initializationOptions?: any;

	/**
	 * クライアント(エディタまたはツール)から提供される機能
	 */
	capabilities: ClientCapabilities;

	/**
	 * トレースの初期設定。省略した場合、トレースは無効('off')となる。
	 */
	trace?: 'off' | 'messages' | 'verbose';

	/**
	 * サーバ開始時にクライアアントで設定されるワークスペースフォルダ。
	 * このプロパティはクライアントがワークスペースフォルダをサポートするときのみ
	 * 表われる。
	 * クライアントがワークスペースフォルダをサポートしているが設定しない場合は
	 * `null` を指定できる。
	 *
	 * 3.6.0 から
	 */
	workspaceFolders?: WorkspaceFolder[] | null;
}
```

`ClientCapabilities`、`TextDocumentClientCapabilities` と `WorkspaceClientCapabilities` は次のように定義される:


##### `WorkspaceClientCapabilities` define capabilities the editor / tool provides on the workspace:
バージョン 3.13 から追加: `ResourceOperationKind`、`FaulureHandlingKind`、クライアントの機能である `workspace.workspaceEdit.resourceOperations`、`workspace.workspaceEdit.failureHandling`

```ts
/**
 * クライアントがサポートするリソース操作種別。
 */
export type ResourceOperationKind = 'create' | 'rename' | 'delete';

export namespace ResourceOperationKind {

	/**
	 * 新規ファイルやフォルダを作成することをサポートする。
	 */
	export const Create: ResourceOperationKind = 'create';

	/**
	 * 既存ファイルやフォルダをリネームすることをサポートする。
	 */
	export const Rename: ResourceOperationKind = 'rename';

	/**
	 * 既存ファイルやフォルダを削除することをサポートする。
	 */
	export const Delete: ResourceOperationKind = 'delete';
}

export type FailureHandlingKind = 'abort' | 'transactional' | 'undo' | 'textOnlyTransactional';

export namespace FailureHandlingKind {

	/**
	 * ワークスペースへの変更を適用する際、一つでも変更が失敗した場合単純に破棄さ
	 * れる。失敗した操作の前に実行された操作は実行されたままである。
	 */
	export const Abort: FailureHandlingKind = 'abort';

	/**
	 * 全ての操作はトランザクショナルに実行される。つまり、ワークスペースには全て
	 * 成功するか全て変更しないかのどちらかのみが適用される。
	 */
	export const Transactional: FailureHandlingKind = 'transactional';


	/**
   * テキストファイルへの変更のみからなるワークスペースの編集はランザクショナル
   * に実行される。変更に含まれるリソースへの変更(ファイルの作成、リネーム、削
   * 除)には `abort` が適用される。
	 */
	export const TextOnlyTransactional: FailureHandlingKind = 'textOnlyTransactional';

	/**
	 * クライアントはすでに実行した操作を undo しようとする。しかし、その実行が成
	 *  功することは保証されない。
	 */
	export const Undo: FailureHandlingKind = 'undo';
}

/**
 * ワークスペース固有のクライアントの機能。
 */
export interface WorkspaceClientCapabilities {
	/**
	 * クライアントは `workspace/applyEdit` リクエストをサポートすることにより、
	 * ワークスペースへのバッチ編集をサポートする。
	 */
	applyEdit?: boolean;

	/**
	 * `WorkspaceEdit` 固有の機能
	 */
	workspaceEdit?: {
		/**
		 * クライアントは `WorkspaceEdit` でバージョン管理されたドキュメントの変更をサポートする。
		 */
		documentChanges?: boolean;

		/**
		 * クライアントがサポートするリソース操作。クライアントは少なくともファイル
		 * とフォルダへの 'create'、'rename'、'delete' をサポートすべきである。
		 */
		resourceOperations?: ResourceOperationKind[];

		/**
		 * ワークスペースへの編集が失敗した場合のクライアントの処理方法。
		 */
		failureHandling?: FailureHandlingKind;
	};

	/**
	 * `workspace/didChangeConfiguration` 通知固有の機能。
	 */
	didChangeConfiguration?: {
		/**
		 * 設定変更通知の動的な登録をサポートする。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `workspace/didChangeWatchedFiles` 通知固有の機能。
	 */
	didChangeWatchedFiles?: {
		/**
		 * ファイル変更監視通知が動的な登録をサポートする。現在のプロトコルではサー
		 * バ側からファイル変更の静的な設定はサポートされていないことに注意する。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `workspace/symbol` リクエスト固有の機能.
	 */
	symbol?: {
		/**
		 * シンボルリクエストの動的な登録をサポートする
		 */
		dynamicRegistration?: boolean;

		/**
		 * `workspace/symbol` リクエスト内の `SymbolKind` 固有の機能。
		 */
		symbolKind?: {
			/**
			 * クライアントがサポートするシンボル種別の値。このプロパティが存在する場
			 * 合、クライアントは範囲外の値を適切に処理し、不明な場合はデフォルト値に
			 * フォールバックすることも保証する。
			 *
			 * このプロパティが与えられていない場合、クライアントは LSP の初期バージョ
			 * ンで定義されていた `File` から `Array` までのみをサポートする。
			 */
			valueSet?: SymbolKind[];
		}
	};

	/**
	 * `workspace/executeCommand` リクエスト固有の機能。
	 */
	executeCommand?: {
		/**
		 * コマンド実行機能の動的な登録をサポートする。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * クライアントがワークスペースフォルダのサポートをする。
	 *
	 * 3.6.0 から
	 */
	workspaceFolders?: boolean;

	/**
	 * `workspace/configuration` リクエストをサポートする。
	 *
	 * 3.6.0 から
	 */
	configuration?: boolean;
}
```

##### `TextDocumentClientCapabilities` define capabilities the editor / tool provides on text documents.

```ts
/**
 * テキストドキュメント固有のクライアントの機能。
 */
export interface TextDocumentClientCapabilities {

	synchronization?: {
		/**
		 * テキストドキュメント同期機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;

		/**
		 * クライアントは `will save` 通知の送信をサポートする。
		 */
		willSave?: boolean;

		/**
		 * クライアントは `will save` リクエストの送信をサポートし、保存前にドキュ
		 * メントに適用するテキスト編集を提供するレスポンスを待つ。
		 */
		willSaveWaitUntil?: boolean;

		/**
		 * クライアントは `did save` 通知をサポートする。
		 */
		didSave?: boolean;
	}

	/**
	 * `textDocument/completion` 固有の機能。
	 */
	completion?: {
		/**
		 * 補完機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;

		/**
		 * クライアントは次の `CompletionItem` 固有の機能をサポートする。
		 */
		completionItem?: {
			/**
			 * クライアントは挿入テキストとしてスニペットをサポートする。
			 *
			 * スニペットはタブ位置とプレースホルダ `$1`、`$2`、`${3:foo}` で定義でき
			 * る。`$0` は最後のタブ位置を定め、デフォルトでスニペットの最後となる。
			 * 同じ識別子プレースホルダはリンクしている、つまり一方を入力すると他も更
			 * 新される。
			 */
			snippetSupport?: boolean;

			/**
			 * クライアントは補完候補上で文字をコミットすることをサポートする。
			 */
			commitCharactersSupport?: boolean

			/**
			 * クライアントはプロパティの文書化に次のコンテントフォーマットをサポート
			 * する。順序はクライアントで優先されるフォーマットを表わす。
			 */
			documentationFormat?: MarkupKind[];

			/**
			 * クライアントは補完候補の非推奨プロパティをサポートする。
			 */
			deprecatedSupport?: boolean;

			/**
			 * クライアントは補完候補の事前確定をサポートする。
			 */
			preselectSupport?: boolean;
		}

		completionItemKind?: {
			/**
			 * クライアントがサポートする補完候補種別。このプロパティが存在する場合、
			 * クライアントは範囲外の値を適切に処理し、不明な場合はデフォルト値に
			 * フォールバックすることも保証する。
			 *
			 * このプロパティが与えられていない場合、クライアントは LSP の初期バージョ
			 * ンで定義されていた `Text` から `Reference` までのみをサポートする。*/
			valueSet?: CompletionItemKind[];
		},

		/**
		 * クライアントは `textDocument/completion` リクエストに追加のコンテキスト
		 * 情報を送信することをサポートする。
		 * `textDocument/completion` request.
		 */
		contextSupport?: boolean;
	};

	/**
	 * `textDocument/hover` 固有の機能。
	 */
	hover?: {
		/**
		 * ホバー機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;

		/**
		 * クライアントはコンテントプロパティに次のコンテントフォーマットをサポート
		 * する。順序はクライアントでのフォーマットの優先度を表わす。
		 */
		contentFormat?: MarkupKind[];
	};

	/**
	 * `textDocument/signatureHelp` 固有の機能。
	 */
	signatureHelp?: {
		/**
		 * シグネチャヘルプ機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;

		/**
		 * クライアントは次の `SignatureInformation` 固有のプロパティをサポートす
		 * る。
		 */
		signatureInformation?: {
			/**
			 * クライアントはドキュメントプロパティに次のフォーマットをサポートする。
			 * 順序はクライアントでのフォーマットの優先度を表わす。
			 */
			documentationFormat?: MarkupKind[];

			/**
			 * パラメータ情報固有のクライアントの機能。
			 */
			parameterInformation?: {
				/**
				 * クライアントは単純なラベル文字の代わりにラベルオフセットの処理をサ
				 * ポートする。
				 *
				 * 3.14.0 から
				 */
				labelOffsetSupport?: boolean;
			}
		};
	};

	/**
	 * `textDocument/references` 固有の機能。
	 */
	references?: {
		/**
		 * リファレンス機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `textDocument/documentHighlight` 固有の機能。
	 */
	documentHighlight?: {
		/**
		 * ドキュメントハイライト機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `textDocument/documentSymbol` 固有の機能。
	 */
	documentSymbol?: {
		/**
		 * ドキュメントシンボル機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;

		/**
		 * `SymbolKind` 固有の機能。
		 */
		symbolKind?: {
			/**
			 * クライアントがサポートするシンボル種別。このプロパティが存在する場合、
			 * クライアントは範囲外の値を適切に処理し、不明な場合はデフォルト値に
			 * フォールバックすることも保証する。
			 *
			 * このプロパティが与えられていない場合、クライアントは LSP の初期バージョ
			 * ンで定義されていた `File` から `Array` までのみをサポートする。
			 */
			valueSet?: SymbolKind[];
		}

		/**
		 * クライアントは階層的なドキュメントシンボルをサポートする。
		 */
		hierarchicalDocumentSymbolSupport?: boolean;
	};

	/**
	 * `textDocument/formatting` 固有の機能。
	 */
	formatting?: {
		/**
		 * フォーマット機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `textDocument/rangeFormatting` 固有の機能。
	 */
	rangeFormatting?: {
		/**
		 * 範囲フォーマット機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * `textDocument/onTypeFormatting` 固有の機能。
	 */
	onTypeFormatting?: {
		/**
		 * 入力中フォーマット機能の動的な登録をサポートするかどうか。
		 */
		dynamicRegistration?: boolean;
	};

	/**
		* `textDocument/declaration` 固有の機能。
		*/
	declaration?: {
		/**
		 * 宣言元参照機能の動的な登録をサポートするかどうか。`true` がセットされた
		 * 場合、クライアントは対応するサーバ機能の
		 * `(TextDocumentRegistrationOptions & StaticRegistrationOptions)` の返り値
		 * もサポートする。
		 */
		dynamicRegistration?: boolean;

		/**
		 * クライアントは宣言リンク形式の追加メタデータをサポートする。
		 *
		 * 3.14.0 から
		 */
		linkSupport?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/definition`.
	 *
	 * Since 3.14.0
	 */
	definition?: {
		/**
		 * Whether definition supports dynamic registration.
		 */
		dynamicRegistration?: boolean;

		/**
		 * The client supports additional metadata in the form of definition links.
		 */
		linkSupport?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/typeDefinition`
	 *
	 * Since 3.6.0
	 */
	typeDefinition?: {
		/**
		 * Whether typeDefinition supports dynamic registration. If this is set to `true`
		 * the client supports the new `(TextDocumentRegistrationOptions & StaticRegistrationOptions)`
		 * return value for the corresponding server capability as well.
		 */
		dynamicRegistration?: boolean;

		/**
		 * The client supports additional metadata in the form of definition links.
		 *
		 * Since 3.14.0
		 */
		linkSupport?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/implementation`.
	 *
	 * Since 3.6.0
	 */
	implementation?: {
		/**
		 * Whether implementation supports dynamic registration. If this is set to `true`
		 * the client supports the new `(TextDocumentRegistrationOptions & StaticRegistrationOptions)`
		 * return value for the corresponding server capability as well.
		 */
		dynamicRegistration?: boolean;

		/**
		 * The client supports additional metadata in the form of definition links.
		 *
		 * Since 3.14.0
		 */
		linkSupport?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/codeAction`
	 */
	codeAction?: {
		/**
		 * Whether code action supports dynamic registration.
		 */
		dynamicRegistration?: boolean;
		/**
		 * The client support code action literals as a valid
		 * response of the `textDocument/codeAction` request.
		 *
		 * Since 3.8.0
		 */
		codeActionLiteralSupport?: {
			/**
			 * The code action kind is support with the following value
			 * set.
			 */
			codeActionKind: {

				/**
				 * The code action kind values the client supports. When this
				 * property exists the client also guarantees that it will
				 * handle values outside its set gracefully and falls back
				 * to a default value when unknown.
				 */
				valueSet: CodeActionKind[];
			};
		};
	};

	/**
	 * Capabilities specific to the `textDocument/codeLens`
	 */
	codeLens?: {
		/**
		 * Whether code lens supports dynamic registration.
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/documentLink`
	 */
	documentLink?: {
		/**
		 * Whether document link supports dynamic registration.
		 */
		dynamicRegistration?: boolean;
	};

	/**
	 * Capabilities specific to the `textDocument/documentColor` and the
	 * `textDocument/colorPresentation` request.
	 *
	 * Since 3.6.0
	 */
	colorProvider?: {
		/**
		 * Whether colorProvider supports dynamic registration. If this is set to `true`
		 * the client supports the new `(ColorProviderOptions & TextDocumentRegistrationOptions & StaticRegistrationOptions)`
		 * return value for the corresponding server capability as well.
		 */
		dynamicRegistration?: boolean;
	}

	/**
	 * Capabilities specific to the `textDocument/rename`
	 */
	rename?: {
		/**
		 * Whether rename supports dynamic registration.
		 */
		dynamicRegistration?: boolean;
		/**
		 * The client supports testing for validity of rename operations
		 * before execution.
		 */
		prepareSupport?: boolean;
	};

	/**
	 * Capabilities specific to `textDocument/publishDiagnostics`.
	 */
	publishDiagnostics?: {
		/**
		 * Whether the clients accepts diagnostics with related information.
		 */
		relatedInformation?: boolean;
	};
	/**
	 * Capabilities specific to `textDocument/foldingRange` requests.
	 *
	 * Since 3.10.0
	 */
	foldingRange?: {
		/**
		 * Whether implementation supports dynamic registration for folding range providers. If this is set to `true`
		 * the client supports the new `(FoldingRangeProviderOptions & TextDocumentRegistrationOptions & StaticRegistrationOptions)`
		 * return value for the corresponding server capability as well.
		 */
		dynamicRegistration?: boolean;
		/**
		 * The maximum number of folding ranges that the client prefers to receive per document. The value serves as a
		 * hint, servers are free to follow the limit.
		 */
		rangeLimit?: number;
		/**
		 * If set, the client signals that it only supports folding complete lines. If set, client will
		 * ignore specified `startCharacter` and `endCharacter` properties in a FoldingRange.
		 */
		lineFoldingOnly?: boolean;
	};
}
```

`ClientCapabilities` はクライアントがサポートする動的な機能登録、ワークスペー
ス、テキストドキュメント機能を定義する。`experimental` は開発中の実験機能を渡す
ために使われる。将来の互換性のため `ClientCapabilities` オブジェクトは現在定義
されている以上のプロパティを設定できる。サーバが不明なプロパティを持つ
`ClientCapabilities` オブジェクトを受け取った場合、それらを無視すべきである。見
付からないプロパティは提供していない機能であると解釈すべきである。見付からない
プロパティがサブプロパティを定義している場合、全てのサブプロパティは対応する機
能がないと解釈すべきである。

クライアント機能は LSP バージョン 3.0 から導入された。そのため 3.x 以降で導入さ
れた機能のみ記述する。バージョン 2.x から存在した機能は引き続きクライアントで必
須である。クライアントはそれらの提供をオプトアウトできない。そのためクライアン
トが `ClientCapabilities.textDocument.synchronization` を省略した場合であって
も、テキストドキュメント同期は必須である(例えば `open`、`changed`、`close` 通
知)。

```ts
interface ClientCapabilities {
	/**
	 * Workspace specific client capabilities.
	 */
	workspace?: WorkspaceClientCapabilities;

	/**
	 * Text document specific client capabilities.
	 */
	textDocument?: TextDocumentClientCapabilities;

	/**
	 * Experimental client capabilities.
	 */
	experimental?: any;
}
```

*レスポンス:*
* 結果: 次で定義される `InitializeResult`:

```ts
interface InitializeResult {
	/**
	 * The capabilities the language server provides.
	 */
	capabilities: ServerCapabilities;
}
```

* error.code:

```ts
/**
 * Known error codes for an `InitializeError`;
 */
export namespace InitializeError {
	/**
	 * If the protocol version provided by the client can't be handled by the server.
	 * @deprecated This initialize error got replaced by client capabilities. There is
	 * no version handshake in version 3.0x
	 */
	export const unknownProtocolVersion: number = 1;
}
```

* error.data:

```ts
interface InitializeError {
	/**
	 * Indicates whether the client execute the following retry logic:
	 * (1) show the message provided by the ResponseError to the user
	 * (2) user selects retry or cancel
	 * (3) if user selected retry the initialize method is sent again.
	 */
	retry: boolean;
}
```

サーバは次の機能を通知できる:

```ts
/**
 * Defines how the host (editor) should sync document changes to the language server.
 */
export namespace TextDocumentSyncKind {
	/**
	 * Documents should not be synced at all.
	 */
	export const None = 0;

	/**
	 * Documents are synced by always sending the full content
	 * of the document.
	 */
	export const Full = 1;

	/**
	 * Documents are synced by sending the full content on open.
	 * After that only incremental updates to the document are
	 * send.
	 */
	export const Incremental = 2;
}

/**
 * Completion options.
 */
export interface CompletionOptions {
	/**
	 * The server provides support to resolve additional
	 * information for a completion item.
	 */
	resolveProvider?: boolean;

	/**
	 * The characters that trigger completion automatically.
	 */
	triggerCharacters?: string[];
}
/**
 * Signature help options.
 */
export interface SignatureHelpOptions {
	/**
	 * The characters that trigger signature help
	 * automatically.
	 */
	triggerCharacters?: string[];
}

/**
 * Code Action options.
 */
export interface CodeActionOptions {
	/**
	 * CodeActionKinds that this server may return.
	 *
	 * The list of kinds may be generic, such as `CodeActionKind.Refactor`, or the server
	 * may list out every specific kind they provide.
	 */
	codeActionKinds?: CodeActionKind[];
}

/**
 * Code Lens options.
 */
export interface CodeLensOptions {
	/**
	 * Code lens has a resolve provider as well.
	 */
	resolveProvider?: boolean;
}

/**
 * Format document on type options.
 */
export interface DocumentOnTypeFormattingOptions {
	/**
	 * A character on which formatting should be triggered, like `}`.
	 */
	firstTriggerCharacter: string;

	/**
	 * More trigger characters.
	 */
	moreTriggerCharacter?: string[];
}

/**
 * Rename options
 */
export interface RenameOptions {
	/**
	 * Renames should be checked and tested before being executed.
	 */
	prepareProvider?: boolean;
}

/**
 * Document link options.
 */
export interface DocumentLinkOptions {
	/**
	 * Document links have a resolve provider as well.
	 */
	resolveProvider?: boolean;
}

/**
 * Execute command options.
 */
export interface ExecuteCommandOptions {
	/**
	 * The commands to be executed on the server
	 */
	commands: string[]
}

/**
 * Save options.
 */
export interface SaveOptions {
	/**
	 * The client is supposed to include the content on save.
	 */
	includeText?: boolean;
}

/**
 * Color provider options.
 */
export interface ColorProviderOptions {
}

/**
 * Folding range provider options.
 */
export interface FoldingRangeProviderOptions {
}

export interface TextDocumentSyncOptions {
	/**
	 * Open and close notifications are sent to the server. If omitted open close notification should not
	 * be sent.
	 */
	openClose?: boolean;
	/**
	 * Change notifications are sent to the server. See TextDocumentSyncKind.None, TextDocumentSyncKind.Full
	 * and TextDocumentSyncKind.Incremental. If omitted it defaults to TextDocumentSyncKind.None.
	 */
	change?: number;
	/**
	 * If present will save notifications are sent to the server. If omitted the notification should not be
	 * sent.
	 */
	willSave?: boolean;
	/**
	 * If present will save wait until requests are sent to the server. If omitted the request should not be
	 * sent.
	 */
	willSaveWaitUntil?: boolean;
	/**
	 * If present save notifications are sent to the server. If omitted the notification should not be
	 * sent.
	 */
	save?: SaveOptions;
}

/**
 * Static registration options to be returned in the initialize request.
 */
interface StaticRegistrationOptions {
	/**
	 * The id used to register the request. The id can be used to deregister
	 * the request again. See also Registration#id.
	 */
	id?: string;
}

interface ServerCapabilities {
	/**
	 * Defines how text documents are synced. Is either a detailed structure defining each notification or
	 * for backwards compatibility the TextDocumentSyncKind number. If omitted it defaults to `TextDocumentSyncKind.None`.
	 */
	textDocumentSync?: TextDocumentSyncOptions | number;
	/**
	 * The server provides hover support.
	 */
	hoverProvider?: boolean;
	/**
	 * The server provides completion support.
	 */
	completionProvider?: CompletionOptions;
	/**
	 * The server provides signature help support.
	 */
	signatureHelpProvider?: SignatureHelpOptions;
	/**
	 * The server provides goto definition support.
	 */
	definitionProvider?: boolean;
	/**
	 * The server provides Goto Type Definition support.
	 *
	 * Since 3.6.0
	 */
	typeDefinitionProvider?: boolean | (TextDocumentRegistrationOptions & StaticRegistrationOptions);
	/**
	 * The server provides Goto Implementation support.
	 *
	 * Since 3.6.0
	 */
	implementationProvider?: boolean | (TextDocumentRegistrationOptions & StaticRegistrationOptions);
	/**
	 * The server provides find references support.
	 */
	referencesProvider?: boolean;
	/**
	 * The server provides document highlight support.
	 */
	documentHighlightProvider?: boolean;
	/**
	 * The server provides document symbol support.
	 */
	documentSymbolProvider?: boolean;
	/**
	 * The server provides workspace symbol support.
	 */
	workspaceSymbolProvider?: boolean;
	/**
	 * The server provides code actions. The `CodeActionOptions` return type is only
	 * valid if the client signals code action literal support via the property
	 * `textDocument.codeAction.codeActionLiteralSupport`.
	 */
	codeActionProvider?: boolean | CodeActionOptions;
	/**
	 * The server provides code lens.
	 */
	codeLensProvider?: CodeLensOptions;
	/**
	 * The server provides document formatting.
	 */
	documentFormattingProvider?: boolean;
	/**
	 * The server provides document range formatting.
	 */
	documentRangeFormattingProvider?: boolean;
	/**
	 * The server provides document formatting on typing.
	 */
	documentOnTypeFormattingProvider?: DocumentOnTypeFormattingOptions;
	/**
	 * The server provides rename support. RenameOptions may only be
	 * specified if the client states that it supports
	 * `prepareSupport` in its initial `initialize` request.
	 */
	renameProvider?: boolean | RenameOptions;
	/**
	 * The server provides document link support.
	 */
	documentLinkProvider?: DocumentLinkOptions;
	/**
	 * The server provides color provider support.
	 *
	 * Since 3.6.0
	 */
	colorProvider?: boolean | ColorProviderOptions | (ColorProviderOptions & TextDocumentRegistrationOptions & StaticRegistrationOptions);
	/**
	 * The server provides folding provider support.
	 *
	 * Since 3.10.0
	 */
	foldingRangeProvider?: boolean | FoldingRangeProviderOptions | (FoldingRangeProviderOptions & TextDocumentRegistrationOptions & StaticRegistrationOptions);
	/**
	 * The server provides go to declaration support.
	 *
	 * Since 3.14.0
	 */
	declarationProvider?: boolean | (TextDocumentRegistrationOptions & StaticRegistrationOptions);
	/**
	 * The server provides execute command support.
	 */
	executeCommandProvider?: ExecuteCommandOptions;
	/**
	 * Workspace specific server capabilities
	 */
	workspace?: {
		/**
		 * The server supports workspace folder.
		 *
		 * Since 3.6.0
		 */
		workspaceFolders?: {
			/**
			* The server has support for workspace folders
			*/
			supported?: boolean;
			/**
			* Whether the server wants to receive workspace folder
			* change notifications.
			*
			* If a strings is provided the string is treated as a ID
			* under which the notification is registered on the client
			* side. The ID can be used to unregister for these events
			* using the `client/unregisterCapability` request.
			*/
			changeNotifications?: string | boolean;
		}
	}
	/**
	 * Experimental server capabilities.
	 */
	experimental?: any;
}

```

#### Initialized notification
`Initialized` 通知はクライアントが `initialize` リクエストの結果を受け取ってか
らその他のリクエストまたは通知をサーバへ送る前にクライアントからサーバへ送信さ
れる。サーバは `initialized` 通知を例えば動的な機能登録に使用できる。
`initialized` 通知は一度だけ送信される。

*通知:*
* メソッド: `initialized`
* パラメータ: 次で定義される `InitializedParams`:

```ts
interface InitializedParams {
}

```

#### Shutdown Request
`shutdown` リクエストはクライアントからサーバへ送信される。サーバをシャットダウ
ンするよう要求するが、終了はしない(そうしなければレスポンスが正しくクライアント
に送信されない可能性がある)。サーバに終了を要求するための `exit` 通知が別にあ
る。クライアントは `exit` 以外の通知またはリクエストを `shutdown` リクエストを
送ったサーバへ送信してはならない。サーバが `shutdown` リクエストの後にリクエス
トを受信した場合、それらは `InvalidRequest` エラーとなる。

*リクエスト:*
* メソッド: `shutdown`
* パラメータ: 空

*レスポンス:*
* 結果: `null`
* エラー: エラーコードと `shutdown` リクエスト中に発生した例外がセットされたメッセージ。

#### Exit notification
サーバ自身のプロセスの終了を要求するための通知。サーバは事前に `shutdown` リク
エストを受信していた場合は `success` コード0で終了すべきであり、それ以外の場合
は `error` コード1で終了する。

*通知:*
* メソッド: `exit`
* パラメータ: 空

#### ShowMessage Notification
`window/showMessage` 通知はクライアントに UI 上に特定のメッセージの表示を要求す
るためにサーバからクライアントに送信される。

*通知:*
* メソッド: `window/showMessage`
* パラメータ: 次で定義される `ShowMessageParams`:

```ts
interface ShowMessageParams {
	/**
	 * The message type. See {@link MessageType}.
	 */
	type: number;

	/**
	 * The actual message.
	 */
	message: string;
}
```

`type` は次のように定義される:

```ts
export namespace MessageType {
	/**
	 * An error message.
	 */
	export const Error = 1;
	/**
	 * A warning message.
	 */
	export const Warning = 2;
	/**
	 * An information message.
	 */
	export const Info = 3;
	/**
	 * A log message.
	 */
	export const Log = 4;
}
```

#### ShowMessage Request
`window/showMessageRequest` リクエストはクライアントに UI 上に特定のメッセージ
の表示を要求するためにサーバからクライアントに送信される。`window/showMessage`
通知に加えてリクエストにアクションを渡すことができ、クライアントからの応答を待
つことができる。

*リクエスト:*
* メソッド: `window/showMessageRequest`
* パラメータ: 次で定義される `ShowMessageRequestParams`:

*レスポンス:*
* 結果: 選択された `MessageActionItem` または選択されていない場合は `null`
* エラー: エラーコードと `window/showMessageRequest` リクエスト中に発生した例外がセットされたメッセージ。

#### LogMessage Notification
`window/logMessage` 通知はクライアントに特定のメッセージをログに残す要求をする
ためにサーバからクライアントに送信される。

*通知:*
* メソッド: `window/logMessage`
* パラメータ: 次で定義される `LogMessageParams`:

```ts
interface LogMessageParams {
	/**
	 * The message type. See {@link MessageType}
	 */
	type: number;

	/**
	 * The actual message
	 */
	message: string;
}
```

`type` は上記で定義した。

#### Telemetry Notification
`telemetry/event` 通知はクライアントにテレメトリのログ保存を要求するためにサー
バからクライアントに送信される。

*通知:*
* メソッド: `telemetry/event`
* パラメータ: `any`

#### Register Capability
`client/registerCapability` リクエストはクライアント上の新たな機能を登録するた
めにサーバから送信される。全てのクライアントが動的な機能登録をサポートする必要
はない。クライアントは特定のクライアント機能の `dynamicRegistration` プロパティ
によってオプトインする。クライアントは機能 A については動的な登録を提供するが、
機能 B については提供しない、ということも可能である
(`TextDocumentClientCapabilities` を例として参照)。

*リクエスト:*
* メソッド: `client/registerCapability`
* パラメータ: `RegistrationParams`

`RegistrationParams` は次のように定義される:

```ts
/**
 * 機能の登録のための一般的なパラメータ。
 */
export interface Registration {
	/**
	 * リクエストを登録するために使われる ID。この ID は登録の解除に再度使用でき
	 *  る。
	 */
	id: string;

	/**
	 * 登録するメソッド/機能。
	 */
	method: string;

	/**
	 * 登録に必要なオプション。
	 */
	registerOptions?: any;
}

export interface RegistrationParams {
	registrations: Registration[];
}
```

ほとんどの登録オプションはドキュメントセレクタの指定を必要なため、基となるイン
ターフェイスがある。

```ts
export interface TextDocumentRegistrationOptions {
	/**
	 * 登録のスコープを識別するためのドキュメントセレクタ。null の場合クライアン
	 * ト側で提供されるドキュメントセレクタが使用される。
	 */
	documentSelector: DocumentSelector | null;
}
```

クライアント上の `textDocument/willSaveWaitUntil` 機能を動的に登録するための JSON RPC の例は次のようになる(概要のみ):

```json
{
	"method": "client/registerCapability",
	"params": {
		"registrations": [
			{
				"id": "79eee87c-c409-4664-8102-e03263673f6f",
				"method": "textDocument/willSaveWaitUntil",
				"registerOptions": {
					"documentSelector": [
						{ "language": "javascript" }
					]
				}
			}
		]
	}
}
```

このメッセージはサーバからクライアントに送信され、クライアントで正常に実行され
た後、JavaScript テキストドキュメントへの `textDocument/willSaveWaitUntil` リク
エストがクライアントからサーバへ送信される。

*レスポンス:*
* 結果: void
* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

#### Unregister Capability
`client/unregisterCapability` リクエストは以前に登録された機能を解除するために
サーバからクライアントに送信される。

*リクエスト:*
* メソッド: `client/unregisterCapability`
* パラメータ: `UnregistrationParams`

`UnregistrationParams` は次のように定義される:

```ts
/**
 * 機能登録解除の一般的はパラメータ。
 */
export interface Unregistration {
	/**
	 * リクエストまたは通知を登録解除するために使われる ID。ID はたいてい登録リク
	 * エスト中に与えられる。
	 */
	id: string;

	/**
	 * 登録解除するメソッド/機能。
	 */
	method: string;
}

export interface UnregistrationParams {
	unregisterations: Unregistration[];
}
```

上で登録した `textDocument/willServeWaitUntil` 機能を解除するための JSON RPC の
例はこのようになる:

```ts
{
	"method": "client/unregisterCapability",
	"params": {
		"unregisterations": [
			{
				"id": "79eee87c-c409-4664-8102-e03263673f6f",
				"method": "textDocument/willSaveWaitUntil"
			}
		]
	}
}
```

*レスポンス:*
* 結果: void
* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

##### Workspace folders request
バージョン 3.6.0 から

多くのツールがワークスペース毎に一つ以上のルートフォルダをサポートする。例えば
VS Code のマルチルートや Atom のプロジェクトフォルダ Sublime のプロジェクトがあ
る。クライアントのワークスペースが複数のルートからなる場合、大抵サーバはそれに
ついて知っておく必要がある。ここまでのプロトコルはサーバから通知される
`InitializeParams` の `rootUri` プロパティがルートフォルダであると仮定する。ク
ライアントがワークスペースフォルダをサポートし、それをクライアントの機能
`workspaceFolders` で通知している場合、`InitializeParams` にはサーバ起動時のワー
クスペースフォルダの設定値である `workspaceFolders` プロパティが含まれる。

`workspace/workspaceFolders` リクエストは現在開いているワークスペースフォルダの
リストを取得するためにサーバからクライアントへ送信される。ツールが一つのファイ
ルしか開いていない場合 `null` が返される。ワークスペースは開かれているがフォル
ダが設定されていない場合は空配列が返される。

*リクエスト:*
* メソッド: `workspace/workspaceFolders`
* パラメータ: なし

*レスポンス:*
* 結果: 次で定義される `WorkspaceFolder[] | null`

```ts
export interface WorkspaceFolder {
	/**
	 * このワークスペースフォルダの URI。
	 */
	uri: DocumentUri;

	/**
	 * ワークスペースフォルダ名。UI 上でこのワークスペースフォルダと関連付けるた
	 * めに使われる。
	 */
	name: string;
}
```

* エラー: エラーコードと `workspace/workspaceFolders` リクエスト中に発生した例外がセットされたメッセージ。

##### DidChangeWorkspaceFolders Notification
バージョン 3.6.0 から

`workspace/didChangeWorkspaceFolders` 通知はワークスペース設定が変更されたこと
をサーバに通知するためにクライアントから送信される。
`ServerCapabilities/workspace/workspaceFolders` と
`ClientCapablitiies/workspace/workspaceFolders` が双方 `true` の場合、またはサー
バ自身がこの通知の受けることを登録した場合、通知はデフォルトで送信される。
`workspace/didChangeWorkspaceFolders` を登録するためにサーバからクライアントへ
`client/registerCapability` リクエストを送る。登録パラメータは次の形式の
`registrations` アイテムを持つ必要がある。`id` は機能の登録解除に使われるユニー
ク ID である(例では UUID を使用する):

```ts
{
	id: "28c6150c-bd7b-11e7-abc4-cec278b6b50a",
	method: "workspace/didChangeWorkspaceFolders"
}
```

*通知:*
* メソッド: `workspace/didChangeWorkspaceFolders`
* パラメータ: 次で定義される `DidChangeWorkspaceFoldersParams`

```ts
export interface DidChangeWorkspaceFoldersParams {
	/**
	 * 実際のワークスペースフォルダ変更イベント。
	 */
	event: WorkspaceFoldersChangeEvent;
}

/**
 * ワークスペースフォルダ変更イベント
 */
export interface WorkspaceFoldersChangeEvent {
	/**
	 * 追加されるワークスペースフォルダの配列
	 */
	added: WorkspaceFolder[];

	/**
	 * 削除されるワークスペースフォルダの配列
	 */
	removed: WorkspaceFolder[];
}

```

#### DidChangeConfiguration Notification
設定変更を知らせるためにクライアントからサーバに送信される通知。

*通知:*
* メソッド: `workspace/didChangeConfiguration`
* パラメータ: 次で定義される `DidChangeConfigurationParams`:

```ts
interface DidChangeConfigurationParams {
	/**
	 * 実際に変更された設定。
	 */
	settings: any;
}
```

#### Configuration Request
バージョン 3.6.0 から

`workspace/configuration` リクエストはクライアントから設定を取得するためにサー
バからクライアントに送信されるリクエスト。リクエストは一度にいくつかの設定を取
得することもできる。返された設定の順序は渡された `ConfigurationItems` の順序と
一致する(例えば、レスポンスの最初の要素はパラメータの最初の設定値となる)。

`ConfigurationItem` は尋ねたい設定セクション名と追加のスコープ URI から構成され
る。尋ねたい設定セクション名はサーバ側で定義され、クライアント上の設定値と一致
している必要はない。そのため、サーバが設定 `cpp.formatterOptions` を聞こうとす
るが、クライアントは異なるレイアウトの XML に設定を保存している。必要な変換をす
るのはクライアント次第である。スコープ URI が与えられる場合、クライアントは与え
られたリソースのスコープ内の設定を返す必要がある。クライアントが例えば
[EditorConfig](https://editorconfig.org/) で設定を管理していた場合、与えられた
リソース URI の設定が返されるべきである。クライアントは与えられたスコープで設定
が提供できない場合、返される配列には `null` を入れる必要がある。

*リクエスト:*
* メソッド: `workspace/configuration`
* パラメータ: 次で定義される `ConfigurationParams`

```ts
export interface ConfigurationParams {
	items: ConfigurationItem[];
}

export interface ConfigurationItem {
	/**
	 * 設定セクションを取得するためのスコープ。
	 */
	scopeUri?: DocumentUri;

	/**
	 * 尋ねたい設定セクション。
	 */
	section?: string;
}
```

*レスポンス:*
* 結果: `any[]`
* エラー: エラーコードと `workspace/configuration` リクエスト中に発生した例外がセットされたメッセージ。

#### DidChangeWatchedFiles Notification
ファイル監視通知はクライアントにより監視されたファイルへの変更を検出したときに
クライアントからサーバへ送信される通知。サーバはそれらのファイルイベントを登録
メカニズムを用いて登録することが推奨される。以前の実装ではクライアントはサーバ
が要求することなくファイルイベントを通知した。

サーバは自身のファイル監視メカニズムを走らせることができ、クライアントが提供す
るファイルイベントに頼らなくてもよい。ただし、次の理由により推奨はされない:

* 経験上、複数の OS に渡りサポートが必要な場合は特に、ディスク上のファイル監視を正しく行なうことは挑戦的である。
* 実装がなんらかのポーリングとタイムスタンプを比較するためにメモリ上にファイルツリーを保持している場合は特に、ファイル監視はリソースを使う。
* クライアントは大抵一つ以上のサーバを起動する。全てのサーバが自身のファイル監視をするのであれば、CPU またはメモリの問題が発生する。
* 一般的に、サーバのほうがクライアントよりも実装することが多い。なのでこの問題はクライアント側で解決するほうがよい。

*通知:*
* メソッド: `workspace/didChangeWatchedFiles`
* パラメータ: 次で定義される `DidChangeWatchedFilesParams`:

```ts
interface DidChangeWatchedFilesParams {
	/**
	 * 実際のファイルイベント。
	 */
	changes: FileEvent[];
}
```

`FileEvent` は次のように記述される:

```ts
/**
 * ファイル変更を記述するイベント。
 */
interface FileEvent {
	/**
	 * ファイルの URI。
	 */
	uri: DocumentUri;
	/**
	 * 変更種別。
	 */
	type: number;
}

/**
 * ファイルイベント種別
 */
export namespace FileChangeType {
	/**
	 * ファイルが作成された。
	 */
	export const Created = 1;
	/**
	 * ファイルが変更された。
	 */
	export const Changed = 2;
	/**
	 * ファイルが削除された。
	 */
	export const Deleted = 3;
}
```

*登録オプション:* `DidChangeWatchedFilesRegistrationOptions` は次のように定義される

```ts
/**
 * ファイルシステム変更イベントを登録する際に使われるオプションを記述する。
 */
export interface DidChangeWatchedFilesRegistrationOptions {
	/**
	 * 登録する監視。
	 */
	watchers: FileSystemWatcher[];
}

export interface FileSystemWatcher {
	/**
	 * 監視する glob パターン。
	 *
	 * glob パターンは次の構文を持つ:
	 * - `*` はパス区切りの中の1文字以上にマッチする
	 * - `?` はパス区切りの中の1文字にマッチする
	 * - `**` はパス区切りが無いことも含む任意の数のパス区切りにマッチする
	 * - `{}` は条件のグループ化(例えば、`**/*.{ts,js}` は全ての TypeScript と JavaScript ファイルにマッチする)
	 * - `[]` はパス区切りの中のマッチする文字の範囲を示す(例えば、`example.[0-9]` は `example.0`、`example.1`、… にマッチする)
	 * - `[!...]` はパス区切りの中のマッチしない文字の範囲を示す(例えば、`example.[!0-9]` は`example.a`、`example.b`、にはマッチするが `example.0` にはマッチしない)
	 */
	globPattern: string;

	/**
	 * 関心のあるイベント種別。省略した場合はデフォルトで 7 つまり
	 * WatchKind.Create | WatchKind.Change | WatchKind.Delete が指定される。
	 */
	kind?: number;
}

export namespace WatchKind {
	/**
	 * 作成イベントに関心がある。
	 */
	export const Create = 1;

	/**
	 * 変更イベントに関心がある。
	 */
	export const Change = 2;

	/**
	 * 削除イベントに関心がある。
	 */
	export const Delete = 4;
}

```

#### Workspace Symbols Request
ワークスペースシンボルリクエストはクエリ文字列に該当するプロジェクト全体のシン
ボルを一覧するためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `workspace/symbol`
* パラメータ: 次のように定義される `WorkspaceSymbolParams`:

```ts
/**
 * ワークスペースシンボルリクエストのパラメータ。
 */
interface WorkspaceSymbolParams {
	/**
	 * 空でないクエリ文字列
	 */
	query: string;
}
```

*レスポンス:*
* 結果: 上で定義された `SymbolInformation[]` | `null`
* エラー: エラーコードと `workspace/symbol` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 空

#### Execute a command
`workspace/executeCommand` リクエストはサーバ上でコマンドを実行するためにクライ
アントからサーバへ送信される。大抵の場合サーバは `WorkspaceEdit` 構造体を作成
し、サーバからクライアントに送られる `workspace/applyEdit` を用いてワークスペー
スへの変更を適用する。

*リクエスト:*
* メソッド: `workspace/executeCommand`
* パラメータ: 次のように定義される `ExecuteCommandParams`:

```ts
export interface ExecuteCommandParams {

	/**
	 * コマンドハンドラの識別子。
	 */
	command: string;
	/**
	 * コマンド実行時に渡すべき引数。
	 */
	arguments?: any[];
}
```

*レスポンス:*
* 結果: `any` | `null`
* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `ExecuteCommandRegistrationOptions`

```ts
/**
 * 実行コマンド登録オプション
 */
export interface ExecuteCommandRegistrationOptions {
	/**
	 * サーバ上で実行されるコマンド。
	 */
	commands: string[]
}

```

#### Applies a WrokspaceEdit
`workspace/applyEdit` リクエストはクライアント側のリソースを変更するためにサー
バからクライアントに送信される。

*リクエスト:*
* メソッド: `workspace/applyEdit`
* パラメータ: 次で定義される `ApplyWorkspaceEditPrams`:

```ts
export interface ApplyWorkspaceEditParams {
	/**
	 * ワークスペース編集のラベル。このラベルは UI で、ワークスペースへの編集を
	 * undo するための undo スタック上などで表示される。
	 */
	label?: string;

	/**
	 * 適用する編集。
	 */
	edit: WorkspaceEdit;
}
```

*レスポンス:*
* 結果: 次のように定義される `ApplyWorkspaceEditResponse`:

```ts
export interface ApplyWorkspaceEditResponse {
	/**
	 * 編集が適用済みかどうかを指す。
	 */
	applied: boolean;

	/**
	 * 編集が適用されなかった理由を示す省略可能な概要。
	 * これはサーバが診断ログや編集を起こしたリクエストに適切なエラーを提供するた
	 * めに使われる場合もある。
	 */
	failureReason?: string;
}
```

* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

#### DidOpenTextDocument Notification
ドキュメントオープン通知は新しいテキストドキュメントを開いたことを知らせるため
にクライアントからサーバへ送信される。ドキュメントの実体はクライアントに管理さ
れ、サーバはドキュメントの URI から実体を読もうとしてはならない。この意味で
`open` はクライアントによって管理されることを意味する。中身がエディタで表示され
ることを必ずしも表わさない。オープン通知は対応するクローズ通知を送信する前に再
度送信してはならない。これはオープンとクローズ通知は同数でなければならず、特定
のテキストドキュメントを開いている数は最大で1でなかればならないことを意味する。
サーバがリクエストを満たす能力はテキストドキュメントが開いているか閉じているか
に依らないことを注意する。

`DidOpenTextDocumentParams` はドキュメントが関連する言語識別子を含む。ドキュメ
ントの言語識別子が変更された場合、クライアントは `textDocument/didClose` をサー
バに送信し、その後新たな言語識別子をサーバが処理できる場合は新しい言語識別子を
`textDocument/didOpen` で送信する必要がある。

*通知:*
* メソッド: `textDocument/didOpen`
* パラメータ: 次で定義される `DidOpenTextDocumentParams`:

```ts
interface DidOpenTextDocumentParams {
	/**
	 * 開かれたドキュメント。
	 */
	textDocument: TextDocumentItem;
}
```

*登録オプション:* `TextDocumentRegisterationOptions`

#### DidChangeTextDocument Notification
ドキュメント変更通知はテキストドキュメントへの変更を伝えるためにクライアントか
らサーバへ送信される。2.0 でパラメータが適切なバージョン番号と言語識別子を持つ
ように変更された。

*通知:*
* メソッド: `textDocument/didChange`
* パラメータ: 次で定義される `DidChangeTextDocumentParams`:

```ts
interface DidChangeTextDocumentParams {
	/**
	 * 変更されたドキュメント。バージョン番号は与えられた変更が全て適用された後の
	 * バージョンを指す。
	 */
	textDocument: VersionedTextDocumentIdentifier;

	/**
	 * コンテントの変更。コンテントの変更はドキュメントへの単一の状態変更として記
	 * 述される。つまり状態 S にあるドキュメントの中身の変更 c1 と c2 がある場合、
	 * c1 はドキュメントを S' にし、c2 は S'' にする。
	 */
	contentChanges: TextDocumentContentChangeEvent[];
}

/**
 * テキストドキュメントへの変更を記述するイベント。`range` と `rangeLength` が
 * 省略された場合、新しいテキストがドキュメントの中身全体であると解釈される。
 */
interface TextDocumentContentChangeEvent {
	/**
	 * 変更するドキュメントの範囲。
	 */
	range?: Range;

	/**
	 * 置換される範囲の長さ。
	 */
	rangeLength?: number;

	/**
	 * 範囲/ドキュメントの新しいテキスト。
	 */
	text: string;
}
```

*登録オプション:* 次で定義される `TextDocumentChangeRegisterationOptions`:

```ts
/**
 * テキストドキュメント変更イベントを登録する際使用されるオプションを記述する。
 */
export interface TextDocumentChangeRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * どのようにドキュメントをサーバへ同期するか。TextDocumentSyncKind.Full と
	 * TextDocumentSyncKind.Incremental を参照。
	 */
	syncKind: number;
}

```

#### WillSaveTextDocument Notification
`WillSaveTextDocument` 通知はドキュメントが保存される前にクライアントからサーバ
へ送信される。

*通知:*
* メソッド: `textDocument/willSave`
* パラメータ: 次で定義される `WillSaveTextDocumentParams`

```ts
/**
 * `WillSaveTextDocument` 通知で送信されるパラメータ。
 */
export interface WillSaveTextDocumentParams {
	/**
	 * 保存されるドキュメント。
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * 'TextDocumentSaveReason'。
	 */
	reason: number;
}

/**
 * テキストドキュメントが保存される理由を表わす。
 */
export namespace TextDocumentSaveReason {

	/**
	 * 手動、例えばユーザが保存ボタンを押した、やデバッグを開始した、または API
	 * が叩かれた。
	 */
	export const Manual = 1;

	/**
	 * 遅延後自動で。
	 */
	export const AfterDelay = 2;

	/**
	 * エディタがフォーカスを失なったとき。
	 */
	export const FocusOut = 3;
}
```

*登録オプション:* `TextDocumentRegisterOptions`

#### WillSaveWaitUntilTextDocument Request
`WillSaveWaitUntilTextDocument` リクエストはドキュメントが保存される前にクライ
アントからサーバに送信される。リクエストは保存されるまでにテキストドキュメント
に適用された `TextEdit` の配列を返すことができる。テキスト編集の計算に非常に時
間がかかった場合やサーバがこのリクエストで常に失敗する場合、クライアントは結果
を落とすかもしれないことに注意する。これは保存を早く、信頼性のあるものにするた
めに行なっている。

*リクエスト:*
* メソッド: `textDocument/willSaveWaitUntil`
* パラメータ: `WillSaveTextDocumentParams`

*レスポンス:*
* 結果: `TextEdit[]` | `null`
* エラー: エラーコードと `willSaveWaitUntil` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### DidSaveTextDocument Notification
`DidSaveTextDocument` 通知はクライアントでドキュメントを保存したときにクライア
ントからサーバへ送信される。

* メソッド: `textDocument/didSave`
* パラメータ: 次で定義される `DidSaveTextDocumentParams`:

```ts
interface DidSaveTextDocumentParams {
	/**
	 * 保存されたドキュメント。
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * 保存された省略可能なコンテント。保存通知がリクエストされたとき、
	 * includeText の値に依存する。
	 */
	text?: string;
}
```

*登録オプション:* 次で定義される `TextDocumentSaveRegistrationOptions`:

```ts
export interface TextDocumentSaveRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * クライアントが保存時のコンテントを含むことをサポートする。
	 */
	includeText?: boolean;
}

```

#### DidCloseTextDocument Notification
`DidCloseTextDocument` 通知はクライアントでドキュメントが閉じられたときにクライ
アントからサーバへ送信される。ドキュメントの実体はドキュメント URI の指す先に存
在する(例えばドキュメント URI がファイル URI の場合実体はディスクに存在する)。
オープン通知と同様にクローズ通知はドキュメントの中身の管理についてのものである。
クローズ通知を受けることはそれまでにドキュメントがエディタで開かれていたことを
意味するわけではない。クローズ通知の前にオープン通知が送信されている必要がある。
サーバがリクエストを満たす能力はテキストドキュメントが開いているか閉じているか
に依らないことを注意する。

*通知:*
* メソッド: `textDocument/didClose`
* パラメータ: 次で定義される `DidCloseTextDocumentParams`:

```ts
interface DidCloseTextDocumentParams {
	/**
	 * 閉じられたドキュメント。
	 */
	textDocument: TextDocumentIdentifier;
}
```

*登録オプション:* `TextDocumentRegistrationOptions`

#### PublishDiagnostics Notification
`PublishDiagnostics` 通知は検証結果を伝えるためにサーバからクライアントに送信さ
れる。

診断結果はサーバのものなので、必要であればクリアすることはサーバの責任である。VS Code では診断結果を生成するサーバは次のルールに従う:

* 単一ファイルからなる言語 (例えば HTML) の場合、ファイルを閉じた際に診断結果をクリアする。
* プロジェクトシステムを持つ言語 (例えば C#) の場合、ファイルを閉じた際には診断結果をクリアしない。プロジェクトが開かれたとき、全てのファイルの全ての診断結果は再計算される(かキャッシュから読まれる)。

ファイルが変更されたとき、診断結果を再計算しクライアントに送信するのはサーバの
責任である。計算結果が空の場合、以前の診断結果をクリアするために空配列を送信す
る必要がある。新たに送信された診断結果は常に以前に送信された診断結果を置き換え
る。クライアント側でマージされることはない。

*通知:*
* メソッド: `textDocument/publishDiagnostics`
* パラメータ: 次で定義される `PublishDiagnosticsParams`:

```ts
interface PublishDiagnosticsParams {
	/**
	 * 診断結果を通知する URI。
	 */
	uri: DocumentUri;

	/**
	 * 診断結果の配列。
	 */
	diagnostics: Diagnostic[];
}

```

#### Completion Request
`Completion` リクエストは与えられたカーソル位置の補完候補を計算するためにクライ
アントからサーバへ送信される。補完候補は
[IntelliSense](https://code.visualstudio.com/docs/editor/editingevolved#_intellisense)
UI で表示される。補完候補を全て計算するコストが高い場合、サーバは
`completionItem/resolve` リクエストへのハンドラを提供することができる。このリク
エストは UI 上で補完候補が選択されたときに送信される。典型的なユースケースとし
て: 計算コストが高いため `textDocument/completion` リクエストは `documentation`
プロパティを埋めずに補完候補を返す。UI 上でその候補が選択されたとき、パラメータ
に選択された補完候補を入れた `completionItem/resolve` リクエストが送信される。
このとき返される補完候補には `documentation` プロパティは入っているべきである。
リクエストは `detail` と `documentation` プロパティの計算を遅延することができ
る。しかし `sortText`、`filterText`、`insertText`、`textEdit` のような最初のソー
トとフィルタに必要なプロパティは `textDocument/cmpletion` レスポンスに含まれて
いる必要があり、`completionItem/resolve` 中に変更してはならない。

*リクエスト:*
* メソッド: `textDocumentcompletion`
* パラメータ: 次で定義される `CompletionParams`

```ts
export interface CompletionParams extends TextDocumentPositionParams {

	/**
	 * 補完コンテキスト。クライアントが
	 * `ClientCapabilities.textDocument.completion.contextSupport === true` を用
	 * いて送信する場合のみ表われる。
	 */
	context?: CompletionContext;
}

/**
 * 補完がどのように引き起こされたか。
 */
export namespace CompletionTriggerKind {
	/**
	 * 補完は識別子(24x7)の入力、手動起動(例えば Ctrl+Space) または API 経由で起
	 * 動された。
	 */
	export const Invoked: 1 = 1;

	/**
	 * 補完は `CompletionRegistrationOptions` の `triggerCharacters` プロパティで
	 * 指定された文字のより起動された。
	 */
	export const TriggerCharacter: 2 = 2;

	/**
	 * 補完は現在の補完候補が不完全なので再度起動された。
	 */
	export const TriggerForIncompleteCompletions: 3 = 3;
}
export type CompletionTriggerKind = 1 | 2 | 3;


/**
 * 補完リクエストが起動したコンテキスト情報を追加で含む。
 */
export interface CompletionContext {
	/**
	 * どのように補完が起動したか。
	 */
	triggerKind: CompletionTriggerKind;

	/**
	 * コード補完を起動した文字(単一の文字)。`triggerKind !==
	 * CompletionTriggerKind.TriggerCharacter` のとき `undefined` となる
	 */
	triggerCharacter?: string;
}
```

*レスポンス:*
* 結果: `CompletionItem[]` | `CompletionList` | `null`。もし `CompletionItem[]` が与えられた場合、補完が完了したことを表わす。つまり `{ isIncomplete: false, items }` と同一である。

```ts
/**
 * エディタ上で表示するための [completion items](#CompletionItem) の集まりを表
 * わす。
 */
interface CompletionList {
	/**
	 * リストは完成していない。さらに入力するとリストが再計算されるべきである。
	 */
	isIncomplete: boolean;

	/**
	 * 補完候補。
	 */
	items: CompletionItem[];
}

/**
 * 補完候補から入力されるテキストがプレインテキスト、スニペットどちらとして処理
 * されるかを定める。
 */
namespace InsertTextFormat {
	/**
	 * 挿入されるテキストはただの文字列として扱われる。
	 */
	export const PlainText = 1;

	/**
	 * 挿入されるテキストはスニペットとして扱われる。
	 *
	 * スニペットはタブ位置とプレースホルダ `$1`、`$2`、`${3:foo}` で定義できる。
	 * `$0` は最後のタブ位置を定め、デフォルトでスニペットの最後となる。同じ識別
	 * 子プレースホルダはリンクしている、つまり一方を入力すると他も更新される。
	 */
	export const Snippet = 2;
}

type InsertTextFormat = 1 | 2;

interface CompletionItem {
	/**
	 * 補完候補のラベル。デフォルトではこの候補を選択したときに挿入されるテキス
	 * ト。
	 */
	label: string;

	/**
	 * 補完候補の種別。種別に基づき、エディタがアイコンを選択する。標準的に利用で
	 * きる値は `CompletionItemKind` で定義される。
	 */
	kind?: number;

	/**
	 * 型やシンボル情報のような、この候補の追加情報を与える可読な文字列。
	 */
	detail?: string;

	/**
	 * コメントを表わす可読な文字列。
	 */
	documentation?: string | MarkupContent;

	/**
	 * この候補が非推奨かどうかを指す。
	 */
	deprecated?: boolean;

	/**
	 * 表示するときにこの候補を選択する。
	 *
	 * *注意* 選択できる補完候補は唯一つであり、候補はツール/クライアントが決定す
	 * る。ルールは最適な *最初の* 候補が選択されることである。
	 */
	preselect?: boolean;

	/**
	 * 他の候補と比較するときに使われるべき文字列。`falsy` な場合は `label` が用
	 * いられる。
	 */
	sortText?: string;

	/**
	 * 補完候補をフィルタするときに使われるべき文字列。`falsy` な場合は `label`
	 * が用いられる。
	 */
	filterText?: string;

	/**
	 * この補完候補が選択されたときに挿入されるべき文字列。`falsy` な場合は
	 * `label` が用いられる。
	 *
	 * `insertText` はクライアント側で解釈される対象である。一部のツールは文字列
	 * をそのまま使わないかもしれない。例えば VS Code は `con<カーソル位置>` で
	 * コード補完がリクエストされ、補完候補の `insertText` が `console` で与えら
	 * れた場合、`sole` のみを挿入する。よってクライアント側での解釈を避けるには
	 * `textEdit` を用いることが推奨される。
	 */
	insertText?: string;

	/**
	 * 挿入するテキストのフォーマット。このフォーマットは `textEdit` プロパティで
	 * 与えられる `insertText` プロパティと `newText` プロパティ双方に適用される。
	 * 省略した場合はデフォルトで `InsertTextFormat.PlainText` が使われる。
	 */
	insertTextFormat?: InsertTextFormat;

	/**
	 * この補完候補が選択されたときにドキュメントに適用される編集。`textEdit` が
	 * 与えられたとき、`insertText` の値は無視される。
	 *
	 * *注意:* 編集範囲は1行で、補完リクエストが発生した位置を含んでいる必要があ
	 * る。
	 */
	textEdit?: TextEdit;

	/**
	 * この補完候補が選択されたときに適用される追加編集の列。編集は `textEdit` と
	 * も配列の他の要素とも被ってはならない(同一位置への挿入を含む)。
	 *
	 * `additionalTextEdits` は現在のカーソル位置とは関係の無いテキスト編集に使用
	 * されるべきである(例えば補完候補が未宣言の型を挿入するときに、ファイル上部
	 * の import 文に追加する)。
	 */
	additionalTextEdits?: TextEdit[];

	/**
	 * この補完候補が有効なときに押されると、許容され、入力される文字の集まり。
	 * *注意* `commitCharacters` の各要素は `length=1` であるべきである、余計な文
	 * 字は無視される。
	 */
	commitCharacters?: string[];

	/**
	 * この補完候補が挿入された *後に* 実行されるコマンド。*注意* この現在のドキュ
	 * メントへの追加の編集は `additionalTextEdits` プロパティに記述されるべきで
	 * ある。
	 */
	command?: Command;

	/**
	 * `textDocument/completion` リクエストと `completionItem/resolve` リクエスト
	 * の間で補完候補に保存されるデータ。
	 */
	data?: any
}

/**
 * 補完候補の種別。
 */
namespace CompletionItemKind {
	export const Text = 1;
	export const Method = 2;
	export const Function = 3;
	export const Constructor = 4;
	export const Field = 5;
	export const Variable = 6;
	export const Class = 7;
	export const Interface = 8;
	export const Module = 9;
	export const Property = 10;
	export const Unit = 11;
	export const Value = 12;
	export const Enum = 13;
	export const Keyword = 14;
	export const Snippet = 15;
	export const Color = 16;
	export const File = 17;
	export const Reference = 18;
	export const Folder = 19;
	export const EnumMember = 20;
	export const Constant = 21;
	export const Struct = 22;
	export const Event = 23;
	export const Operator = 24;
	export const TypeParameter = 25;
}
```

* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `CompletionRegistrationOptions`:

```ts
export interface CompletionRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * ほとんどのツールはキーボードショートカット(例えば Ctrl+Space)を用いて明示
	 * 的にリクエストすることなく自動的に補完リクエストを発火する。典型的にはユー
	 * ザが識別子の入力を始めたときに補完は開始される。例えばユーザが `c` を
	 * JavaScript ファイルで入力した場合、コード補完は他の候補と共に `console` を
	 * 候補として自動的にポップアップする。識別子を構成する文字をここにリストする
	 * 必要はない。
	 *
	 * コード補完が識別子内で無効な文字により自動的に発火されるべき場合(例えば
	 * JavaScript での `.`)、`triggerCharacters` に列挙する。
	 */
	triggerCharacters?: string[];

	/**
	 * 補完を確定することのできる全ての文字。クライアントが補完候補毎の確定文字を
	 * サポートしていない場合に使われる。
	 * `ClientCapabilities.textDocument.completion.completionItem.commitCharactesSupport`
	 * を参照。
	 *
	 * サーバが `allCommitCharacters` と各補完候補の確定文字を双方提供している場
	 * 合、補完候補に設定されたものが優先される。
	 *
	 * 3.2.0 から
	 */
	allCommitCharacters?: string[];

	/**
	 * サーバが補完候補の追加情報を解決することをサポートする。
	 */
	resolveProvider?: boolean;
}
```

補完候補はスニペットをサポートする(`InsertTextFormat.Snippet` を参照)。スニペッ
トのフォーマットは次のように定義される。

##### Snippet Syntax
スニペットの `body` はカーソルの操作とテキスト入力のための特別な構成要素を使う
ことができる。次はそれらの構文と機能である:

##### Tab stops
タブ位置により、エディタのカーソルをスニペット内で移動させることができる。`$1`、
`$2` でカーソル位置を指定する。数字はタブ位置に移動する順序であり、`$0` は最後
のカーソル位置を記述する。複数のタブ位置はリンクしており、更新は同期される。

##### Placeholders
プレースホルダは `${1:foo}` のような、値付きのタブ位置である。プレースホルダの
テキストは挿入され、簡単に変更できるように選択される。プレースホルダは
`${1:another ${2:placeholder}}` のように、ネストすることができる。

##### Choice
プレースホルダは値として選択肢を持つ。構文は、例えば `${1|one,two,three|} のよ
うに、カンマ区切りで列挙された、パイプで囲まれた値である。スニペットが挿入され、
プレースホルダが選択されたとき、選択肢はユーザに値を選択するよう促す。

##### Variables
`$name` または `${name:default}` で変数の値を挿入することができる。変数の中身が
セットされていない場合、`default` または空文字列が挿入される。不明な変数(つまり
名前が宣言されていない)の場合、変数名が挿入され、プレースホルダに変換される。

次の変数が使用できる:
* `TM_SELECTED_TEXT` 現在選択中のテキストまたは空文字列
* `TM_CURRENT_LINE` 現在行の中身
* `TM_CURRENT_WORD` カーソル上の単語の中身または空文字列
* `TM_LINE_INDEX` 0始まりの行番号
* `TM_LINE_NUMBER` 1始まりの行番号
* `TM_FILENAME` 現在のドキュメントのファイル名
* `TM_FILENAME_BASE` 現在のドキュメントの拡張子を除くファイル名
* `TM_DIRECTORY` 現在のドキュメントのディレクトリ
* `TM_FILEPATH` 現在のドキュメントの絶対パス

##### Variable Transforms
変換により入力前の変数の値を編集することができる。変換の定義は3つの部分からな
る。

1. 変数の値と照合される正規表現、または変数を解決できない場合は空文字列。
2. 正規表現のマッチグループへの参照を持つフォーマット文字列。フォーマット文字列により条件付きの挿入や単純な編集ができる。
3. 正規表現に渡されるオプション。

次の例は現在のファイル名から最後を除いて挿入する例である、つまり `foo.txt` から
`foo` を作る。

```
${TM_FILENAME/(.*)\..+$/$1/}
  |           |         | |
  |           |         | |-> オプションなし
  |           |         |
  |           |         |-> 最初のキャプチャグループへの参照
  |           |
  |           |-> 最後の `.suffix` の前全てを捕えるための正規表現
  |
  |-> ファイル名の解決
```

##### Grammar
スニペットの EBNF([extended Backus-Naur
form](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form)) を以下
に示す。`\`(バックスラッシュ) により、`$`、`}`、`\` をエスケープできる。選択肢
ではバックスラッシュで更にカンマとパイプ文字をエスケープできる。

```
any         ::= tabstop | placeholder | choice | variable | text
tabstop     ::= '$' int | '${' int '}'
placeholder ::= '${' int ':' any '}'
choice      ::= '${' int '|' text (',' text)* '|}'
variable    ::= '$' var | '${' var }'
                | '${' var ':' any '}'
                | '${' var '/' regex '/' (format | text)+ '/' options '}'
format      ::= '$' int | '${' int '}'
                | '${' int ':' '/upcase' | '/downcase' | '/capitalize' '}'
                | '${' int ':+' if '}'
                | '${' int ':?' if ':' else '}'
                | '${' int ':-' else '}' | '${' int ':' else '}'
regex       ::= JavaScript Regular Expression value (ctor-string)
options     ::= JavaScript Regular Expression option (ctor-options)
var         ::= [_a-zA-Z] [_a-zA-Z0-9]*
int         ::= [0-9]+
text        ::= .*

```

#### Completion Item Resolve Request
このリクエストは与えられた補完候補の追加情報を解決するためにクライアントからサー
バへ送信される。

*リクエスト:*
* メソッド: `completionItem/resolve`
* パラメータ: `CompletionItem`

*レスポンス:*
* 結果: `CompletionItem`
* エラー: エラーコードと `completionItem/resolve` リクエスト中に発生した例外がセットされたメッセージ。


#### Hover Request
`Hover` リクエストは与えられたテキスト位置でのホバー情報を要求するためにクライ
アントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/hover`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: 次で定義される `Hover` | `null`:

```ts
/**
 * ホバーリクエストの結果
 */
interface Hover {
	/**
	 * ホバーの中身
	 */
	contents: MarkedString | MarkedString[] | MarkupContent;

	/**
	 * オプションの `range` は、例えば背景色を変更するような、ホバーを表示するた
	 * めに使われるテキストドキュメント内の範囲。
	 */
	range?: Range;
}
```

`MarkedString` は次のように定義される:

```ts
/**
 * MarkedString は人間の読めるテキストを表示するために使うことができる。これは
 * markdown 文字列または言語とコードスニペットが与えられたコードブロックである。
 * 言語識別子は GitHub issue の fenced code block の言語識別子と意味的に同じで
 * ある。
 * https://help.github.com/articles/creating-and-highlighting-code-blocks/#syntax-highlighting
 * を参照。
 *
 * 言語と値のペアは markdown と同一である:
 * ```${language}
 * ${value}
 * ```
 *
 * マークダウン文字列はサニタイズされる。- つまり、HTML はエスケープされる。
* @deprecated MarkupContent を代わりに用いる。
*/
type MarkedString = string | { language: string; value: string };
```

* エラー: エラーコードと `textDocument/hover` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Signature Help Request
`Signnature Help` リクエストは与えられたカーソル位置でのシグネチャ情報を要求す
るためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/signatureHelp`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: 次で定義される `SignatureHelp` | `null`:

```ts
/**
 * Signature help represents the signature of something
 * callable. There can be multiple signature but only one
 * active and only one active parameter.
 */
interface SignatureHelp {
	/**
	 * One or more signatures.
	 */
	signatures: SignatureInformation[];

	/**
	 * The active signature. If omitted or the value lies outside the
	 * range of `signatures` the value defaults to zero or is ignored if
	 * `signatures.length === 0`. Whenever possible implementors should
	 * make an active decision about the active signature and shouldn't
	 * rely on a default value.
	 * In future version of the protocol this property might become
	 * mandatory to better express this.
	 */
	activeSignature?: number;

	/**
	 * The active parameter of the active signature. If omitted or the value
	 * lies outside the range of `signatures[activeSignature].parameters`
	 * defaults to 0 if the active signature has parameters. If
	 * the active signature has no parameters it is ignored.
	 * In future version of the protocol this property might become
	 * mandatory to better express the active parameter if the
	 * active signature does have any.
	 */
	activeParameter?: number;
}

/**
 * Represents the signature of something callable. A signature
 * can have a label, like a function-name, a doc-comment, and
 * a set of parameters.
 */
interface SignatureInformation {
	/**
	 * The label of this signature. Will be shown in
	 * the UI.
	 */
	label: string;

	/**
	 * The human-readable doc-comment of this signature. Will be shown
	 * in the UI but can be omitted.
	 */
	documentation?: string | MarkupContent;

	/**
	 * The parameters of this signature.
	 */
	parameters?: ParameterInformation[];
}

/**
 * Represents a parameter of a callable-signature. A parameter can
 * have a label and a doc-comment.
 */
interface ParameterInformation {

	/**
	 * The label of this parameter information.
	 *
	 * Either a string or an inclusive start and exclusive end offsets within its containing
	 * signature label. (see SignatureInformation.label). The offsets are based on a UTF-16
	 * string representation as `Position` and `Range` does.
	 *
	 * *Note*: a label of type string should be a substring of its containing signature label.
	 * Its intended use case is to highlight the parameter label part in the `SignatureInformation.label`.
	 */
	label: string | [number, number];

	/**
	 * The human-readable doc-comment of this parameter. Will be shown
	 * in the UI but can be omitted.
	 */
	documentation?: string | MarkupContent;
}
```

* エラー: エラーコードと `textDocument/hover` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `SignatureHelpRegistrationOptions`:

```ts
export interface SignatureHelpRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * The characters that trigger signature help
	 * automatically.
	 */
	triggerCharacters?: string[];
}

```

#### Goto Declaration Request
バージョン 3.14.0 から

`Goto Declaration` リクエストは与えられたテキストドキュメント位置のシンボルの宣
言位置を解決するためにクライアントからサーバへ送信される。

結果の
[`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[]
型はバージョン 3.14.0 で導入され、一致するクライアント機能
`clientCapabilities.textDocument.declaration.linkSupport` に依存する。

*リクエスト:*
* メソッド: `textDocument/declaration`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location) | [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location)[] | [`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[] | `null`
* エラー: エラーコードと `textDocument/declaration` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Goto Definition Request
バージョン 3.14.0 から

`Goto Definition` リクエストは与えられたテキストドキュメント位置のシンボルの定
義位置を解決するためにクライアントからサーバへ送信される。

結果の
[`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[]
型はバージョン 3.14.0 で導入され、一致するクライアント機能
`clientCapabilities.textDocument.declaration.linkSupport` に依存する。

*リクエスト:*
* メソッド: `textDocument/definition`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location) | [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location)[] | [`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[] | `null`
* エラー: エラーコードと `textDocument/definition` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Goto Type Definition Request
バージョン 3.6.0 から

`Goto Type Definition` リクエストは与えられたテキストドキュメント位置のシンボル
の型定義位置を解決するためにクライアントからサーバへ送信される。

結果の
[`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[]
型はバージョン 3.14.0 で導入され、一致するクライアント機能
`clientCapabilities.textDocument.declaration.linkSupport` に依存する。

*リクエスト:*
* メソッド: `textDocument/typeDefinition`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location) | [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location)[] | [`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[] | `null`
* エラー: エラーコードと `textDocument/typeDefinition` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Goto Implementation Request
バージョン 3.6.0 から

`Goto Implementation` リクエストは与えられたテキストドキュメント位置のシンボル
の実装位置を解決するためにクライアントからサーバへ送信される。

結果の
[`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[]
型はバージョン 3.14.0 で導入され、一致するクライアント機能
`clientCapabilities.textDocument.declaration.linkSupport` に依存する。

*リクエスト:*
* メソッド: `textDocument/implementation`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location) | [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location)[] | [`LocationLink`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#locationlink)[] | `null`
* エラー: エラーコードと `textDocument/implementation` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Find References Request
`Find References` リクエストは与えられたテキスト位置で記述されているシンボルの
プロジェクト全体での参照を解決するためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/references`
* パラメータ: 次で定義される `ReferenceParams`:

```ts
interface ReferenceParams extends TextDocumentPositionParams {
	context: ReferenceContext
}

interface ReferenceContext {
	/**
	 * Include the declaration of the current symbol.
	 */
	includeDeclaration: boolean;
}
```

*レスポンス:*
* 結果: [`Location`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#location) | `null`
* エラー: エラーコードと `textDocument/references` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Document Highlights Request
`Document Hightlights` リクエストは与えられたテキスト位置のドキュメントハイライ
トを解決するためにクライアントからサーバへ送信される。プログラミング言語の場合、
通常、このファイルをスコープとするシンボルへの全ての参照をハイライトする。しか
し、 `textDocument/documentHighlight` と `textDocument/references` を別のリクエ
ストとしたのは最初のリクエストをより曖昧にできるようにするためである。シンボル
はたいてい `DocumentHighlightKind` の `Read` または `Write` にマッチする、一方
曖昧または文字的な一致には `Text` が用いられる。

*リクエスト:*
* メソッド: `textDocument/documentHighlight`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: 次で定義される `DocumentHighlight[]` | `null`

```ts
/**
 * A document highlight is a range inside a text document which deserves
 * special attention. Usually a document highlight is visualized by changing
 * the background color of its range.
 *
 */
interface DocumentHighlight {
	/**
	 * The range this highlight applies to.
	 */
	range: Range;

	/**
	 * The highlight kind, default is DocumentHighlightKind.Text.
	 */
	kind?: number;
}

/**
 * A document highlight kind.
 */
export namespace DocumentHighlightKind {
	/**
	 * A textual occurrence.
	 */
	export const Text = 1;

	/**
	 * Read-access of a symbol, like reading a variable.
	 */
	export const Read = 2;

	/**
	 * Write-access of a symbol, like writing to a variable.
	 */
	export const Write = 3;
}
```

* エラー: エラーコードと `textDocument/documentHighlight` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Document Symbols Request
`Document Symbols` リクエストはクライアントからサーバへ送信される。結果はいずれ
かである

* `SymbolInformation[]` 与えられたテキストドキュメント内で見付かった全てのシンボルのリスト。シンボルの場所の範囲もシンボルのコンテナ名も階層構造の推測に用いるべきではない。
* `DocumentSymbol[]` 与えられたテキストドキュメント内で見付かったシンボルの階層構造。

*リクエスト:*
* メソッド: `textDocument/documentSymbol`
* パラメータ: 次で定義される `DocumentSymbolParams`:

```ts
interface DocumentSymbolParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;
}
```

*レスポンス:*
* 結果: 次で定義される `DocumentSymbol[]` | `SymbolInformation[]` | `null`:

```ts
/**
 * A symbol kind.
 */
export namespace SymbolKind {
	export const File = 1;
	export const Module = 2;
	export const Namespace = 3;
	export const Package = 4;
	export const Class = 5;
	export const Method = 6;
	export const Property = 7;
	export const Field = 8;
	export const Constructor = 9;
	export const Enum = 10;
	export const Interface = 11;
	export const Function = 12;
	export const Variable = 13;
	export const Constant = 14;
	export const String = 15;
	export const Number = 16;
	export const Boolean = 17;
	export const Array = 18;
	export const Object = 19;
	export const Key = 20;
	export const Null = 21;
	export const EnumMember = 22;
	export const Struct = 23;
	export const Event = 24;
	export const Operator = 25;
	export const TypeParameter = 26;
}

/**
 * Represents programming constructs like variables, classes, interfaces etc. that appear in a document. Document symbols can be
 * hierarchical and they have two ranges: one that encloses its definition and one that points to its most interesting range,
 * e.g. the range of an identifier.
 */
export class DocumentSymbol {

	/**
	 * The name of this symbol. Will be displayed in the user interface and therefore must not be
	 * an empty string or a string only consisting of white spaces.
	 */
	name: string;

	/**
	 * More detail for this symbol, e.g the signature of a function.
	 */
	detail?: string;

	/**
	 * The kind of this symbol.
	 */
	kind: SymbolKind;

	/**
	 * Indicates if this symbol is deprecated.
	 */
	deprecated?: boolean;

	/**
	 * The range enclosing this symbol not including leading/trailing whitespace but everything else
	 * like comments. This information is typically used to determine if the clients cursor is
	 * inside the symbol to reveal in the symbol in the UI.
	 */
	range: Range;

	/**
	 * The range that should be selected and revealed when this symbol is being picked, e.g the name of a function.
	 * Must be contained by the `range`.
	 */
	selectionRange: Range;

	/**
	 * Children of this symbol, e.g. properties of a class.
	 */
	children?: DocumentSymbol[];
}

/**
 * Represents information about programming constructs like variables, classes,
 * interfaces etc.
 */
interface SymbolInformation {
	/**
	 * The name of this symbol.
	 */
	name: string;

	/**
	 * The kind of this symbol.
	 */
	kind: number;

	/**
	 * Indicates if this symbol is deprecated.
	 */
	deprecated?: boolean;

	/**
	 * The location of this symbol. The location's range is used by a tool
	 * to reveal the location in the editor. If the symbol is selected in the
	 * tool the range's start information is used to position the cursor. So
	 * the range usually spans more then the actual symbol's name and does
	 * normally include things like visibility modifiers.
	 *
	 * The range doesn't have to denote a node range in the sense of a abstract
	 * syntax tree. It can therefore not be used to re-construct a hierarchy of
	 * the symbols.
	 */
	location: Location;

	/**
	 * The name of the symbol containing this symbol. This information is for
	 * user interface purposes (e.g. to render a qualifier in the user interface
	 * if necessary). It can't be used to re-infer a hierarchy for the document
	 * symbols.
	 */
	containerName?: string;
}
```

* エラー: エラーコードと `textDocument/documentSymbol` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Code Action Request
`Code Action` リクエストは与えられたテキストドキュメントと範囲上でコマンドを実
行するためにクライアントからサーバへ送信される。コマンドは典型的には問題を修正
するかコードを綺麗/リファクタリングするものである。`textDocument/codeAction` リ
クエストの結果は、通常 UI 上で表示される `Command` リテラルの配列である。サーバ
を多くのクライアントから使いやすくするために、指定されたコマンドはクライアント
側ではなくサーバ側で処理すべきである(`workspace/executeCommand` と
`ServerCapabilities.executeCommandPrivider` を参照)。クライアントがコードアク
ションによる編集を提供する場合、モードを使用するべきである。

コマンドがサーバにより選択された場合、コマンドを実行するために再度
(`workspace/executeCommand`)リクエストを送信するべきである。

*バージョン 3.8.0 から:* `CodeAction` リテラルがサポートされることで、次のよう
なシナリオが可能になる:

* コードアクションリクエストから直接ワークスペース編集に戻る機能。これにより実際のコードアクション実行のための別のサーバへの往復を避けることができる。しかし、サーバの提供者はコードアクションの計算が重かったり、編集が膨大な場合、結果がシンプルなコマンドであり必要な場合のみ実際の編集が実行されることが有用なサーバ実装であることを知っておくべきである。
* 種別を用いたコードアクションのグループ化機能。クライアントはこの情報を無視できる。しかし、これにより例えば対応するメニューにコードアクションをグループ化できる(例えばリファクタリングメニュー内に全てのリファクタリング用コードアクションを表示する)。

クライアントは対応するクライアント機能
`textDocument.codeAction.codeActionLiteralSupport` により `CodeAction` リテラル
とコードアクション種別をサポートすることを知らせる必要がある。

*リクエスト:*
* メソッド: `textDocument/codeAction`
* パラメータ: 次で定義される `CodeActionParams`:

```ts
/**
 * Params for the CodeActionRequest
 */
interface CodeActionParams {
	/**
	 * The document in which the command was invoked.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The range for which the command was invoked.
	 */
	range: Range;

	/**
	 * Context carrying additional information.
	 */
	context: CodeActionContext;
}

/**
 * The kind of a code action.
 *
 * Kinds are a hierarchical list of identifiers separated by `.`, e.g. `"refactor.extract.function"`.
 *
 * The set of kinds is open and client needs to announce the kinds it supports to the server during
 * initialization.
 */
export type CodeActionKind = string;

/**
 * A set of predefined code action kinds
 */
export namespace CodeActionKind {

	/**
	 * Empty kind.
	 */
	export const Empty: CodeActionKind = '';

	/**
	 * Base kind for quickfix actions: 'quickfix'
	 */
	export const QuickFix: CodeActionKind = 'quickfix';

	/**
	 * Base kind for refactoring actions: 'refactor'
	 */
	export const Refactor: CodeActionKind = 'refactor';

	/**
	 * Base kind for refactoring extraction actions: 'refactor.extract'
	 *
	 * Example extract actions:
	 *
	 * - Extract method
	 * - Extract function
	 * - Extract variable
	 * - Extract interface from class
	 * - ...
	 */
	export const RefactorExtract: CodeActionKind = 'refactor.extract';

	/**
	 * Base kind for refactoring inline actions: 'refactor.inline'
	 *
	 * Example inline actions:
	 *
	 * - Inline function
	 * - Inline variable
	 * - Inline constant
	 * - ...
	 */
	export const RefactorInline: CodeActionKind = 'refactor.inline';

	/**
	 * Base kind for refactoring rewrite actions: 'refactor.rewrite'
	 *
	 * Example rewrite actions:
	 *
	 * - Convert JavaScript function to class
	 * - Add or remove parameter
	 * - Encapsulate field
	 * - Make method static
	 * - Move method to base class
	 * - ...
	 */
	export const RefactorRewrite: CodeActionKind = 'refactor.rewrite';

	/**
	 * Base kind for source actions: `source`
	 *
	 * Source code actions apply to the entire file.
	 */
	export const Source: CodeActionKind = 'source';

	/**
	 * Base kind for an organize imports source action: `source.organizeImports`
	 */
	export const SourceOrganizeImports: CodeActionKind = 'source.organizeImports';
}

/**
 * Contains additional diagnostic information about the context in which
 * a code action is run.
 */
interface CodeActionContext {
	/**
	 * An array of diagnostics.
	 */
	diagnostics: Diagnostic[];

	/**
	 * Requested kind of actions to return.
	 *
	 * Actions not of this kind are filtered out by the client before being shown. So servers
	 * can omit computing them.
	 */
	only?: CodeActionKind[];
}
```

*レスポンス:*
* 結果: `(Command | CodeAction)[]` | `null`、`CodeAction` は次のように定義される。

```ts
/**
 * A code action represents a change that can be performed in code, e.g. to fix a problem or
 * to refactor code.
 *
 * A CodeAction must set either `edit` and/or a `command`. If both are supplied, the `edit` is applied first, then the `command` is executed.
 */
export interface CodeAction {

	/**
	 * A short, human-readable, title for this code action.
	 */
	title: string;

	/**
	 * The kind of the code action.
	 *
	 * Used to filter code actions.
	 */
	kind?: CodeActionKind;

	/**
	 * The diagnostics that this code action resolves.
	 */
	diagnostics?: Diagnostic[];

	/**
	 * The workspace edit this code action performs.
	 */
	edit?: WorkspaceEdit;

	/**
	 * A command this code action executes. If a code action
	 * provides an edit and a command, first the edit is
	 * executed and then the command.
	 */
	command?: Command;
}
```

* エラー: エラーコードと `textDocument/codeAction` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `CodeActionRegisterationOptions`:

```ts
export interface CodeActionRegistrationOptions extends TextDocumentRegistrationOptions, CodeActionOptions {
}

```

#### Code Lens Request
`Code Lens` リクエストは与えられたテキストドキュメントのコードレンズを計算するためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/codeLens`
* パラメータ: 次で定義される `CodeLensParams`:

```ts
interface CodeLensParams {
	/**
	 * The document to request code lens for.
	 */
	textDocument: TextDocumentIdentifier;
}
```

*レスポンス:*
* 結果: 次で定義される `CodeLens[]` | `null`:

```ts
/**
 * A code lens represents a command that should be shown along with
 * source text, like the number of references, a way to run tests, etc.
 *
 * A code lens is _unresolved_ when no command is associated to it. For performance
 * reasons the creation of a code lens and resolving should be done in two stages.
 */
interface CodeLens {
	/**
	 * The range in which this code lens is valid. Should only span a single line.
	 */
	range: Range;

	/**
	 * The command this code lens represents.
	 */
	command?: Command;

	/**
	 * A data entry field that is preserved on a code lens item between
	 * a code lens and a code lens resolve request.
	 */
	data?: any
}
```

* エラー: エラーコードとリクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `CodeLensRegistrationOptions`:

```ts
export interface CodeLensRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * Code lens has a resolve provider as well.
	 */
	resolveProvider?: boolean;
}

```

#### Code Lens Resolve Request
`Code Lens Resolve` リクエストは与えられたコードレンズアイテムのコマンドを解決
するためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `codeLens/resolve`
* パラメータ: `CodeLens`

*レスポンス:*
* 結果: `CodeLens`
* エラー: エラーコードと `codeLens/resolve` リクエスト中に発生した例外がセットされたメッセージ。

#### Document Link Request
`Document Link` リクエストはドキュメントへのリンクを要求するためにクライアント
からサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/documentLink`
* パラメータ: 次で定義される `DocumentLinkParams`:

```ts
interface DocumentLinkParams {
	/**
	 * The document to provide document links for.
	 */
	textDocument: TextDocumentIdentifier;
}
```

*レスポンス:*
* 結果: `DocumentLink` | `null` の配列

```ts
/**
 * A document link is a range in a text document that links to an internal or external resource, like another
 * text document or a web site.
 */
interface DocumentLink {
	/**
	 * The range this link applies to.
	 */
	range: Range;
	/**
	 * The uri this link points to. If missing a resolve request is sent later.
	 */
	target?: DocumentUri;
	/**
	 * A data entry field that is preserved on a document link between a
	 * DocumentLinkRequest and a DocumentLinkResolveRequest.
	 */
	data?: any;
}
```

* エラー: エラーコードと `textDocument/documentLink` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `DocumentLinkRegsitrationOptions`:

```ts
export interface DocumentLinkRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * Document links have a resolve provider as well.
	 */
	resolveProvider?: boolean;
}

```

#### Document Link Resolve Request
`Document Link Resolve` リクエストは与えられた `DocumentLink` を解決するために
クライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `documentLink/resolve`
* パラメータ: `DocumentLink`

*レスポンス:*
* 結果: `DocumentLink`
* エラー: エラーコードと `documentLink/resolve` リクエスト中に発生した例外がセットされたメッセージ。

#### Document Color Request
バージョン 3.6.0 から

`Document Color` リクエストは与えられたテキストドキュメントで見付かった全ての色
参照を列挙するためにクライアントからサーバへ送信される。範囲に加えて、色の RGB
値が返される。

クライアントはエディタで色参照を装飾するために結果を使うことができる。例えば:

* 参照の横に実際の色のカラーボックスを表示する
* 色参照の編集時にカラーピッカーを表示する

*リクエスト:*
* メソッド: `textDocument/documentColor`
* パラメータ: 次で定義される `DocumentColorParams`

```ts
interface DocumentColorParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;
}
```

*レスポンス:*
* 結果: 次で定義される `ColorInformation[]`

```ts
interface ColorInformation {
	/**
	 * The range in the document where this color appears.
	 */
	range: Range;

	/**
	 * The actual color value for this color range.
	 */
	color: Color;
}

/**
 * Represents a color in RGBA space.
 */
interface Color {

	/**
	 * The red component of this color in the range [0-1].
	 */
	readonly red: number;

	/**
	 * The green component of this color in the range [0-1].
	 */
	readonly green: number;

	/**
	 * The blue component of this color in the range [0-1].
	 */
	readonly blue: number;

	/**
	 * The alpha component of this color in the range [0-1].
	 */
	readonly alpha: number;
}
```

* エラー: エラーコードと `textDocument/documentColor` リクエスト中に発生した例外がセットされたメッセージ。

#### Color Presentation Request
バージョン 3.6.0 から

`Color Presentation` リクエストは与えられた位置の色値の表現一覧を取得するために
クライアントからサーバへ送信される。クライアントはこの結果を

* 色参照を編集する
* カラーピッカーの中で表示し、表現をユーザに選択させる

ことに使用できる。

*リクエスト:*
* メソッド: `textDocument/colorPresentation`
* パラメータ: 次で定義される `ColorPresentationParams`

```ts
interface ColorPresentationParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The color information to request presentations for.
	 */
	color: Color;

	/**
	 * The range where the color would be inserted. Serves as a context.
	 */
	range: Range;
}
```

*レスポンス:*
* 結果: 次で定義される `ColorPresentation[]`

```ts
interface ColorPresentation {
	/**
	 * The label of this color presentation. It will be shown on the color
	 * picker header. By default this is also the text that is inserted when selecting
	 * this color presentation.
	 */
	label: string;
	/**
	 * An [edit](#TextEdit) which is applied to a document when selecting
	 * this presentation for the color.  When `falsy` the [label](#ColorPresentation.label)
	 * is used.
	 */
	textEdit?: TextEdit;
	/**
	 * An optional array of additional [text edits](#TextEdit) that are applied when
	 * selecting this color presentation. Edits must not overlap with the main [edit](#ColorPresentation.textEdit) nor with themselves.
	 */
	additionalTextEdits?: TextEdit[];
}
```

* エラー: エラーコードと `textDocument/colorPresentation` リクエスト中に発生した例外がセットされたメッセージ。

#### Document Formatting Request
`Document Formatting` リクエストはドキュメント全体をフォーマットするためにクラ
イアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/formatting`
* パラメータ: 次で定義される `DocumentFormattingParams`

```ts
interface DocumentFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The format options.
	 */
	options: FormattingOptions;
}

/**
 * Value-object describing what options formatting should use.
 */
interface FormattingOptions {
	/**
	 * Size of a tab in spaces.
	 */
	tabSize: number;

	/**
	 * Prefer spaces over tabs.
	 */
	insertSpaces: boolean;

	/**
	 * Signature for further properties.
	 */
	[key: string]: boolean | number | string;
}
```

*レスポンス:*
* 結果: [`TextEdit`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textedit)[] | `null` フォーマットされたドキュメントへの編集が記述される。
* エラー: エラーコードと `textDocument/formatting` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Document Range Formatting Request
`Document Range Formatting` リクエストは与えられたドキュメントの範囲をフォーマッ
トするためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/rangeFormatting`
* パラメータ: 次で定義される `DocumentRangeFormattingParams`

```ts
interface DocumentRangeFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The range to format
	 */
	range: Range;

	/**
	 * The format options
	 */
	options: FormattingOptions;
}
```

*レスポンス:*
* 結果: [`TextEdit`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textedit)[] | `null` フォーマットされたドキュメントへの編集が記述される。
* エラー: エラーコードと `textDocument/rangeFormatting` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

#### Document on Type Formatting Request
`Document on Type Formatting` リクエストは入力中のドキュメントをフォーマットす
るためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/onTypeFormatting`
* パラメータ: 次で定義される `DocumentOnTypeFormattingParams`:

```ts
interface DocumentOnTypeFormattingParams {
	/**
	 * The document to format.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position at which this request was sent.
	 */
	position: Position;

	/**
	 * The character that has been typed.
	 */
	ch: string;

	/**
	 * The format options.
	 */
	options: FormattingOptions;
}
```

*レスポンス:*
* 結果: [`TextEdit`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textedit)[] | `null` フォーマットされたドキュメントへの編集が記述される。
* エラー: エラーコードと `textDocument/onTypeFormatting` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `DocumentOnTypeFormattingRegistrationOptions`:

```ts
export interface DocumentOnTypeFormattingRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * A character on which formatting should be triggered, like `}`.
	 */
	firstTriggerCharacter: string;
	/**
	 * More trigger characters.
	 */
	moreTriggerCharacter?: string[]
}

```

#### Rename Request
`Rename` リクエストは、クライアントがシンボルのワークスペース全体でのリネームを
行なうために、サーバがワークスペースへの変更を計算するために、クライアントから
サーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/rename`
* パラメータ: 次で定義される `RenameParams`

```ts
interface RenameParams {
	/**
	 * The document to rename.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position at which this request was sent.
	 */
	position: Position;

	/**
	 * The new name of the symbol. If the given name is not valid the
	 * request must return a [ResponseError](#ResponseError) with an
	 * appropriate message set.
	 */
	newName: string;
}
```

*レスポンス:*
* 結果: [`WorkspaceEdit`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#workspaceedit) | `null` ワークスペースへの変更を記述する
* エラー: エラーコードと `textDocument/rename` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* 次で定義される `RenameRegistrationOptions`:

```ts
export interface RenameRegistrationOptions extends TextDocumentRegistrationOptions {
	/**
	 * Renames should be checked and tested for validity before being executed.
	 */
	prepareProvider?: boolean;
}

```

#### Prepare Rename Request
バージョン 3.12.0 から

`Prepare Rename` リクエストは与えられた場所でのリネーム処理の妥当性をテストする
ためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/prepareRename`
* パラメータ: [`TextDocumentPositionParams`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#textdocumentpositionparams)

*レスポンス:*
* 結果: [`Range`](https://microsoft.github.io/language-server-protocol/specifications/specification-3-14/#range) | `{ range: Range, placeholder: string }` | `null` リネームする文字列の範囲とオプションでリネームする文字列のプレースホルダを記述する。`null` が返された場合、与えられた位置での `textDocument/rename` リクエストは無効であるとみなす。
* エラー: エラーコードと `textDocument/prepareRename` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:*

#### Folding Range Request
バージョン 3.10.0 から

`Folding Range` リクエストは与えられたテキストドキュメントの折り畳まれた全て範囲を返すためにクライアントからサーバへ送信される。

*リクエスト:*
* メソッド: `textDocument/foldingRange`
* パラメータ: 次で定義される `FoldingRangeParams`

```ts
export interface FoldingRangeParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;
}
```

*レスポンス:*
* 結果: 次で定義される `FoldingRange[]` | `null`

```ts
/**
 * Enum of known range kinds
 */
export enum FoldingRangeKind {
	/**
	 * Folding range for a comment
	 */
	Comment = 'comment',
	/**
	 * Folding range for a imports or includes
	 */
	Imports = 'imports',
	/**
	 * Folding range for a region (e.g. `#region`)
	 */
	Region = 'region'
}

/**
 * Represents a folding range.
 */
export interface FoldingRange {

	/**
	 * The zero-based line number from where the folded range starts.
	 */
	startLine: number;

	/**
	 * The zero-based character offset from where the folded range starts. If not defined, defaults to the length of the start line.
	 */
	startCharacter?: number;

	/**
	 * The zero-based line number where the folded range ends.
	 */
	endLine: number;

	/**
	 * The zero-based character offset before the folded range ends. If not defined, defaults to the length of the end line.
	 */
	endCharacter?: number;

	/**
	 * Describes the kind of the folding range such as `comment` or `region`. The kind
	 * is used to categorize folding ranges and used by commands like 'Fold all comments'. See
	 * [FoldingRangeKind](#FoldingRangeKind) for an enumeration of standardized kinds.
	 */
	kind?: string;
}
```

* エラー: エラーコードと `textDocument/foldingRange` リクエスト中に発生した例外がセットされたメッセージ。

*登録オプション:* `TextDocumentRegistrationOptions`

### Implementation considerations
言語サーバは大抵別のプロセスとして動作し、クライアントとは非同期に通信する。加
えてクライアントは大抵リクエストがペンディング状態であってもソースコードへの変
更が許容される。クライアントに古いレスポンスを適用させないために次の実装パター
ンを推奨する。

* クライアントがサーバにリクエストを送信し、結果が無効になるようにクライアントの状態が変更する場合、サーバリクエストをキャンセルし、結果を無視すべきである。必要であれば、最新の結果を受信するためにリクエストを再送できる。
* サーバが実行中のリクエストの結果を無効にする状態変更を検知した場合、そのリクエストに `ContentModified` エラーを返すことができる。クライアントが `ContentModified` エラーを受信した場合、一般的に UI 上でエンドユーザに見せるべきではない。クライアントは適切であればリクエストを再送できる。
* サーバが矛盾した状態に陥った場合、`window/logMessage` リクエストでクライアントにそのことをログに残させるべきである。その状態から回復できない場合、最善は自身を終了されることである。現在クライアントからサーバを再起動させるための[プロトコル拡張](https://github.com/Microsoft/language-server-protocol/issues/646)を考えているところである。
* クライアントが予期しないサーバの存在を検知した場合、サーバの再起動を試みるべきである。ただし、クライアントはクラッシュしたサーバを無限に再起動しないように注意すべきである。VS Code では例えば、180秒で5回クラッシュした場合はサーバを再起動しない。
