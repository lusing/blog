---
title: "操作系统形式化验证实践教程(1) - 证明第一个定理"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

# vscode插件快餐教程(8) - LSP文本同步

这一节开始我们介绍下通过LSP进行文本同步的方法。

## 文件打开

我们先从简单的做起，先监听文件的打开。
我们看一下LSP协议中对此部分的支持，参数是DidChangeTextDocumentParams结构。
![LSP文本同步.png](https://upload-images.jianshu.io/upload_images/1638145-744206c3197cf363.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

微软的SDK在LSP的基础上是做了封装的，我们看下封装后的接口：
![文本改变API.png](https://upload-images.jianshu.io/upload_images/1638145-bd1ffa166187329e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当前，TextDocument提供了4个属性：
- uri: 文件的URI
- version: 文件的版本号
- languageId: 编程语言
- lineCount: 有多少行
另外还有3个函数：
- getText(): 获取文本
- positionAt和offsetAt用于Position和offset的转换

我们来看个例子：
```ts
documents.onDidOpen(
	(event: TextDocumentChangeEvent) => {
		logger.debug(`on open:${event.document.uri}`);
		logger.debug(`file version:${event.document.version}`);
		logger.debug(`file content:${event.document.getText()}`);
		logger.debug(`language id:${event.document.languageId}`);
		logger.debug(`line count:${event.document.lineCount}`);
	}
);
```
我们来看一个运行的例子：
```
[2019-06-04T18:11:31.999] [DEBUG] lsp_demo - on open:file:///Users/ziyingliuziying/test.vb
[2019-06-04T18:11:31.999] [DEBUG] lsp_demo - file version:1
[2019-06-04T18:11:31.999] [DEBUG] lsp_demo - file content:dim a as integer;
TextView1
Javascript
Button3
Test2

[2019-06-04T18:11:31.999] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:11:32.000] [DEBUG] lsp_demo - line count:6
```

## 监听文件变化

监听文件变化与监听打开文件基本上是一模一样的，代码如下：
```ts
documents.onDidChangeContent(
	(e: TextDocumentChangeEvent) => {
		logger.debug('document change received.');
		logger.debug(`document version:${e.document.version}`);
		logger.debug(`text:${e.document.getText()}`);
		logger.debug(`language id:${e.document.languageId}`);
		logger.debug(`line count:${e.document.lineCount}`);
	}
);
```

```
[2019-06-04T18:30:34.329] [DEBUG] lsp_demo - document change received.
[2019-06-04T18:30:34.329] [DEBUG] lsp_demo - document version:1
[2019-06-04T18:30:34.329] [DEBUG] lsp_demo - text:dim a as integer;
TextView1
Javascript
Button3
Test2

[2019-06-04T18:30:34.329] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:30:34.329] [DEBUG] lsp_demo - line count:6

[2019-06-04T18:30:39.457] [DEBUG] lsp_demo - document change received.
[2019-06-04T18:30:39.457] [DEBUG] lsp_demo - document version:2
[2019-06-04T18:30:39.457] [DEBUG] lsp_demo - text:

[2019-06-04T18:30:39.457] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:30:39.458] [DEBUG] lsp_demo - line count:2

[2019-06-04T18:30:41.576] [DEBUG] lsp_demo - document change received.
[2019-06-04T18:30:41.576] [DEBUG] lsp_demo - document version:3
[2019-06-04T18:30:41.577] [DEBUG] lsp_demo - text:b
[2019-06-04T18:30:41.577] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:30:41.577] [DEBUG] lsp_demo - line count:1

[2019-06-04T18:30:41.949] [DEBUG] lsp_demo - document change received.
[2019-06-04T18:30:41.949] [DEBUG] lsp_demo - document version:4
[2019-06-04T18:30:41.949] [DEBUG] lsp_demo - text:u
[2019-06-04T18:30:41.949] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:30:41.949] [DEBUG] lsp_demo - line count:1

[2019-06-04T18:30:42.447] [DEBUG] lsp_demo - document change received.
[2019-06-04T18:30:42.447] [DEBUG] lsp_demo - document version:5
[2019-06-04T18:30:42.447] [DEBUG] lsp_demo - text:Button5
[2019-06-04T18:30:42.447] [DEBUG] lsp_demo - language id:vb
[2019-06-04T18:30:42.447] [DEBUG] lsp_demo - line count:1
```

## 文本监听模式

上面的监听方式是增量监听，使用TextDocumentSyncKind.Incremental模式，代码如下：
```ts
connection.onInitialize((params: InitializeParams) => {
	return {
		capabilities: {
			textDocumentSync: {
				openClose: true,
				change: TextDocumentSyncKind.Incremental
			},
			completionProvider: {
				resolveProvider: true
			}
		}
	};
});
```
增量模式是每次只传变化的部分。
下面我们可以看看传全量模式与其的区别：

```ts
connection.onInitialize((params: InitializeParams) => {
	return {
		capabilities: {
			textDocumentSync: {
				openClose: true,
				change: TextDocumentSyncKind.Full
			},
			completionProvider: {
				resolveProvider: true
			}
		}
	};
});
```

全量模式下，每次变化后的全量都会通过消息传递过来，我们看个例子：

```
[2019-06-04T19:52:12.305] [DEBUG] lsp_demo - document change received.
[2019-06-04T19:52:12.305] [DEBUG] lsp_demo - document version:1
[2019-06-04T19:52:12.305] [DEBUG] lsp_demo - text:dim a as integer;
TextView1
Javascript
Button3
Test2
Button5

[2019-06-04T19:52:12.305] [DEBUG] lsp_demo - language id:vb
[2019-06-04T19:52:12.305] [DEBUG] lsp_demo - line count:7
[2019-06-04T19:52:19.442] [DEBUG] lsp_demo - document change received.
[2019-06-04T19:52:19.442] [DEBUG] lsp_demo - document version:2
[2019-06-04T19:52:19.442] [DEBUG] lsp_demo - text:dim a as integer;
TextView1
Javascript
Button3
Test2
Button5
T
[2019-06-04T19:52:19.443] [DEBUG] lsp_demo - language id:vb
[2019-06-04T19:52:19.443] [DEBUG] lsp_demo - line count:7
[2019-06-04T19:52:19.443] [DEBUG] lsp_demo - onCompletion
[2019-06-04T19:52:19.787] [DEBUG] lsp_demo - document change received.
[2019-06-04T19:52:19.787] [DEBUG] lsp_demo - document version:5
[2019-06-04T19:52:19.787] [DEBUG] lsp_demo - text:dim a as integer;
TextView1
Javascript
Button3
Test2
Button5
Test
[2019-06-04T19:52:19.787] [DEBUG] lsp_demo - language id:vb
[2019-06-04T19:52:19.787] [DEBUG] lsp_demo - line count:7
[2019-06-04T19:52:19.788] [DEBUG] lsp_demo - onCompletion
[2019-06-04T19:52:21.877] [DEBUG] lsp_demo - document change received.
[2019-06-04T19:52:21.877] [DEBUG] lsp_demo - document version:6
[2019-06-04T19:52:21.877] [DEBUG] lsp_demo - text:dim a as integer;
TextView1
Javascript
Button3
Test2
Button5
Test

[2019-06-04T19:52:21.877] [DEBUG] lsp_demo - language id:vb
[2019-06-04T19:52:21.877] [DEBUG] lsp_demo - line count:8
```

还可以选择TextDocumentSyncKind.None模式，这时候不同步文本信息。
