---
title: "vscode插件快餐教程(6) - LSP协议的初始化参数"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-0a60a8245a35988d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

学习了lsp的代码补全之后，我们可以尝试搭建一套可以运行的lsp的系统。
在此之前，我们再将一些细节夯实一下。

我们在第4节曾经介绍过LSP的初始化的握手过程。
我们可以在connection的onInitialize函数中来接收客户端的初始化参数，比如客户端的能力。

```js
connection.onInitialize((params: InitializeParams) => {
	let capabilities = params.capabilities;

	return {
		capabilities: {
			textDocumentSync: documents.syncKind,
			// Tell the client that the server supports code completion
			completionProvider: {
				resolveProvider: true
			}
		}
	};
})
```

我们先用一张图来看一下lsp初始化参数所包含的内容：
![lsp初始化参数](https://upload-images.jianshu.io/upload_images/1638145-0a60a8245a35988d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下面我们看下这些定义：

```ts
interface InitializeParams {
	/**
	 * The process Id of the parent process that started
	 * the server. Is null if the process has not been started by another process.
	 * If the parent process is not alive then the server should exit (see exit notification) its process.
	 */
	processId: number | null;

	/**
	 * The rootPath of the workspace. Is null
	 * if no folder is open.
	 *
	 * @deprecated in favour of rootUri.
	 */
	rootPath?: string | null;

	/**
	 * The rootUri of the workspace. Is null if no
	 * folder is open. If both `rootPath` and `rootUri` are set
	 * `rootUri` wins.
	 */
	rootUri: DocumentUri | null;

	/**
	 * User provided initialization options.
	 */
	initializationOptions?: any;

	/**
	 * The capabilities provided by the client (editor or tool)
	 */
	capabilities: ClientCapabilities;

	/**
	 * The initial trace setting. If omitted trace is disabled ('off').
	 */
	trace?: 'off' | 'messages' | 'verbose';

	/**
	 * The workspace folders configured in the client when the server starts.
	 * This property is only available if the client supports workspace folders.
	 * It can be `null` if the client supports workspace folders but none are
	 * configured.
	 *
	 * Since 3.6.0
	 */
	workspaceFolders?: WorkspaceFolder[] | null;
}
```

我们将上面的信息分下类，如下图所示：
![InitializeParams](https://upload-images.jianshu.io/upload_images/1638145-c93cae654e26a0ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要有三种信息：
- 一是运行环境相关的信息，如路径相关信息和进程相关信息。
- 二是能力信息。
- 三是一些辅助信息，包括trace设置和用户自定义信息。

路径信息是只有在插件运行的vscode中已经打开了一个目录时才可以获取到。
假设我打开的目录是：/Users/ziyingliuziying/working/gitlab/cafmode/server，那么rootPath返回的结果是：/Users/ziyingliuziying/working/gitlab/cafmode/server。而rootUri返回的结果是：file:///Users/ziyingliuziying/working/gitlab/cafmode/server。
workspaceFolders是一个由WorkspaceFolder组成的数组。每一个WorkspaceFolder是由name和uri组成的对象。以上面的例子为例：name为server, uri是file:///Users/ziyingliuziying/working/gitlab/cafmode/server

## 能力参数

从前面的WorkspaceClientCapabilities和TextDocumentClientCapabilities两大族能力，我们可以大致了解其中的功能。

下面我们来看个实际的能力返回值的例子：
```json
{ workspace:
   { applyEdit: true,
     workspaceEdit: { documentChanges: true },
     didChangeConfiguration: { dynamicRegistration: true },
     didChangeWatchedFiles: { dynamicRegistration: true },
     symbol: { dynamicRegistration: true, symbolKind: [Object] },
     executeCommand: { dynamicRegistration: true },
     configuration: true,
     workspaceFolders: true },
  textDocument:
   { publishDiagnostics: { relatedInformation: true },
     synchronization:
      { dynamicRegistration: true,
        willSave: true,
        willSaveWaitUntil: true,
        didSave: true },
     completion:
      { dynamicRegistration: true,
        contextSupport: true,
        completionItem: [Object],
        completionItemKind: [Object] },
     hover: { dynamicRegistration: true, contentFormat: [Array] },
     signatureHelp:
      { dynamicRegistration: true, signatureInformation: [Object] },
     definition: { dynamicRegistration: true },
     references: { dynamicRegistration: true },
     documentHighlight: { dynamicRegistration: true },
     documentSymbol:
      { dynamicRegistration: true,
        symbolKind: [Object],
        hierarchicalDocumentSymbolSupport: true },
     codeAction:
      { dynamicRegistration: true,
        codeActionLiteralSupport: [Object] },
     codeLens: { dynamicRegistration: true },
     formatting: { dynamicRegistration: true },
     rangeFormatting: { dynamicRegistration: true },
     onTypeFormatting: { dynamicRegistration: true },
     rename: { dynamicRegistration: true },
     documentLink: { dynamicRegistration: true },
     typeDefinition: { dynamicRegistration: true },
     implementation: { dynamicRegistration: true },
     colorProvider: { dynamicRegistration: true },
     foldingRange:
      { dynamicRegistration: true,
        rangeLimit: 5000,
        lineFoldingOnly: true } } }
```
