---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# vscode插件快餐教程(9) - LSP补全与本地补全

我们接续第5讲未介绍完的LSP的onCompletion补全的部分。

## TextDocumentPositionParams

在第5讲，我们曾经介绍过LSP处理onCompletion的例子，我们再复习一下：
```js
connection.onCompletion(
	(_textDocumentPosition: TextDocumentPositionParams): CompletionItem[] => {
		return [
			{
				label: 'TextView',
				kind: CompletionItemKind.Text,
				data: 1
			},
			{
				label: 'Button',
				kind: CompletionItemKind.Text,
				data: 2
			},
			{
				label: 'ListView',
				kind: CompletionItemKind.Text,
				data: 3
			}
		];
	}
)
```

这其中的TextDocumentPositionParams其实非常简单，只有文档uri，行，列三个参数。
我们来看下其定义：

```js
export interface TextDocumentPositionParams {
	/**
	 * The text document.
	 */
	textDocument: TextDocumentIdentifier;

	/**
	 * The position inside the text document.
	 */
	position: Position;
}
```

### TextDocumentIdentifier

TextDocumentIdentifier封装了两层，本质上就是一个URI的字符串。

```js
/**
 * A literal to identify a text document in the client.
 */
export interface TextDocumentIdentifier {
	/**
	 * The text document's uri.
	 */
	uri: DocumentUri;
}
```

DocumentUri其实就是string的马甲，请看定义：

```js
/**
 * A tagging type for string properties that are actually URIs.
 */
export type DocumentUri = string;
```

这个URI地址，一般是所编辑文件地址，以Windows上的地址为例：

```
file:///c%3A/working/temp/completions-sample/test.bas
```

### Position

Position由行号line和列号character组成：

```js
export interface Position {
	/**
	 * Line position in a document (zero-based).
	 * If a line number is greater than the number of lines in a document, it defaults back to the number of lines in the document.
	 * If a line number is negative, it defaults to 0.
	 */
	line: number;

	/**
	 * Character offset on a line in a document (zero-based). Assuming that the line is
	 * represented as a string, the `character` value represents the gap between the
	 * `character` and `character + 1`.
	 *
	 * If the character value is greater than the line length it defaults back to the
	 * line length.
	 * If a line number is negative, it defaults to 0.
	 */
	character: number;
}
```

## LSP与本地CompleteProvider的对照

LSP毕竟是一套完整的协议，可以多条消息或命令配合执行。而本地Provider提供的功能相对更全面一些。

上面我们介绍了onComplete的参数是一个URI字符串，而在CompleteProvider中，则直接获取到完整的TextDocument的内容：
```
provideCompletionItems(document: vscode.TextDocument, position: vscode.Position, token: vscode.CancellationToken, context: vscode.CompletionContext)
```

通过TextDocument对象，我们就可以获取到文本的内容，版本号，所对应的语言等等：
```js
	let provider1 = vscode.languages.registerCompletionItemProvider('plaintext', {

		provideCompletionItems(document: vscode.TextDocument, position: vscode.Position, token: vscode.CancellationToken, context: vscode.CompletionContext) {
			console.log('document version=' + document.version);
			console.log('text is:' + document.getText());
			console.log('URI is:' + document.uri);
			console.log('Language ID=' + document.languageId);
			console.log('Line Count=' + document.lineCount);
```

## CompleteItem

说完参数，我们再说说返回值中的CompleteItem。

### 最简单的CompleteItem类型 - 字符串补全

最简单的就是直接给一个字符串，例：
```ts
const simpleCompletion = new vscode.CompletionItem('console.log');
```

这样，当用户输入c的时候，就会提示是否要补全console.log。

### Code Snippets补全

另外高级一点的补全，是允许用户进行选择和替换的补全，类似于Code Snippets功能。
比如我们可以提供log, warn, error三个选项给console做补全：

```ts
			const snippetCompletion = new vscode.CompletionItem('console');
			snippetCompletion.insertText = new vscode.SnippetString('console.${1|log,warn,error|}. Is it console.${1}?');
			snippetCompletion.documentation = new vscode.MarkdownString("Code snippet for console");
```

也就是说，除了默认的label属性，这个例子中还指定了insertText和documentation属性。

### 指定commit键的补全

这一节我们增加commitCharacters，文档也选用更强大的MarkdownString:

```ts
			const commitCharacterCompletion = new vscode.CompletionItem('console');
			commitCharacterCompletion.commitCharacters = ['.'];
			commitCharacterCompletion.documentation = new vscode.MarkdownString('Press `.` to get `console.`');
```

然后，如第5节中所述一样，我们还需要为二段补全提供一个新的provider:
```ts
	const provider2 = vscode.languages.registerCompletionItemProvider(
		'plaintext',
		{
			provideCompletionItems(document: vscode.TextDocument, position: vscode.Position) {

				// get all text until the `position` and check if it reads `console.`
				// and if so then complete if `log`, `warn`, and `error`
				let linePrefix = document.lineAt(position).text.substr(0, position.character);
				if (!linePrefix.endsWith('console.')) {
					return undefined;
				}

				return [
					new vscode.CompletionItem('log', vscode.CompletionItemKind.Method),
					new vscode.CompletionItem('warn', vscode.CompletionItemKind.Method),
					new vscode.CompletionItem('error', vscode.CompletionItemKind.Method),
				];
			}
		},
		'.' // triggered whenever a '.' is being typed
	);
```

### 终极大招：调用其它命令进行补全

最后，我们如果自己搞不定了，还可以通过指定command属性来调用其它命令来进行补全，比如本例中我们调用editor.action.triggerSuggest命令来进行进一步的处理：

```ts
			const commandCompletion = new vscode.CompletionItem('new');
			commandCompletion.kind = vscode.CompletionItemKind.Keyword;
			commandCompletion.insertText = 'new ';
			commandCompletion.command = { command: 'editor.action.triggerSuggest', title: 'Re-trigger completions...' };
```

## 实现异步补全

vscode的CompletionProvider另外强大的一点是，provideCompletionItems是可以async的，这样就可以去等待另一个费时的线程甚至是远程的服务返回来进行补全计算了，只要await真正计算的线程就好了。
我们来个需要服务器返回的例子看下：

```js
	let provider1 = vscode.languages.registerCompletionItemProvider('javascript', {

		async provideCompletionItems(document: vscode.TextDocument, position: vscode.Position, token: vscode.CancellationToken, context: vscode.CompletionContext) {
			let item: vscode.CompletionItem = await instance.post('/complete', { code: getLine(document, position) })
				.then(function (response: any) {
					console.log('complete: ' + response.data);
					return new vscode.CompletionItem(response.data);
				})
				.catch(function (error: Error) {
					console.log(error);
					return new vscode.CompletionItem('No suggestion');
				});

			return [item];
```
