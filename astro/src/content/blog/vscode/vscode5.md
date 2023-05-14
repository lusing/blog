---
title: "vscode插件快餐教程(5) - 代码补全"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---


上节我们介绍了lsp的基本框架和协议的三次握手。
下面我们先学习一个最简单的功能协议：给vscode发送一条通知。

## LSP窗口消息

在LSP协议中，跟窗口相关的协议有三条：
- window/ShowMessage Notification
- window/showMessage Request
- window/logMessage Notification

我们可以使用Connection.window.sendxxxMessage函数来向客户端发送消息。
根据消息程度的不同，分为Information, Warning和Error三个级别。

举个例子，我们可以在onInitialized，也就是客户端与服务端三次握手一切就绪之后，向客户端发一个消息。
```js
connection.onInitialized(() => {
	connection.window.showInformationMessage('Hello World! form server side');
});
```
显示结果如下：
![showMessage](https://upload-images.jianshu.io/upload_images/1638145-1a5d6f01ffde54ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 代码补全

我们用窗口通知热热身，测试一下链路通不通。下面我们就直奔我们最感兴趣的主题之一：代码补全。

代码补全的形式其实也很简单，输入是一个TextDocumentPositionParams，输出是一个CompletionItem的数组，这个函数注册到connection.onCompletion中：
```js
connection.onCompletion(
	(_textDocumentPosition: TextDocumentPositionParams): CompletionItem[] => {});
```

代码补全中用到的主要数据结构如下图所示：
![代码补全.png](https://upload-images.jianshu.io/upload_images/1638145-5640f2f3960ef3d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中kind属性由一个枚举定义：
![CompletionItemKind.png](https://upload-images.jianshu.io/upload_images/1638145-46d72ff3b279f6ed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大家不要被吓到，我们通过一个简单的例子看一下，其实基本实现方法还是很简单的：
```ts
connection.onCompletion(
	(_textDocumentPosition: TextDocumentPositionParams): CompletionItem[] => {
		connection.console.log('[xulun]Position:' + _textDocumentPosition.textDocument);

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

### 补全的详细信息

除了补全信息textDocument/completion之外，lsp还支持completionItem/resolve请求，输入和输出都是CompletionItem，返回进一步的信息。
通过connection.onCompletionResolve方法可以注册对于completionItem/resolve请求的支持：

```ts
connection.onCompletionResolve(
	(item: CompletionItem): CompletionItem => {
		if (item.data === 1) {
			item.detail = 'TextView';
			item.documentation = 'TextView documentation';
		} else if (item.data === 2) {
			item.detail = 'Button';
			item.documentation = 'JavaScript documentation';
		} else if (item.data === 3) {
			item.detail = 'ListView';
			item.documentation = 'ListView documentation';
		}
		return item;
	}
)
```

运行效果如下：
![ListView](https://upload-images.jianshu.io/upload_images/1638145-422885960d885fcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 使用参数中的补全位置信息

输入参数中会带有发出补全申请的位置信息，我们可以根据这个信息来控制补全的信息。
我们以一个例子来说明下：
```ts
connection.onCompletion(
	(_textDocumentPosition: TextDocumentPositionParams): CompletionItem[] => {
		
		return [
			{
				label: 'TextView' + _textDocumentPosition.position.character,
				kind: CompletionItemKind.Text,
				data: 1
			},
			{
				label: 'Button' + _textDocumentPosition.position.line,
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
我们此时不光补全一个控件名，还将当前的行号或列号增加其中。
下面是补全Button的运行情况，会增加当前的行号到补全信息中，我们在934行触发补全，于是补全提示的信息变成Button933：
![补全带列号](https://upload-images.jianshu.io/upload_images/1638145-3b9add3b0bc132f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

