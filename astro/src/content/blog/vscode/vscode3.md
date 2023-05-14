---
title: "vscode插件快餐教程(3) - Diagnostic"
description: "波澜壮阔的操作系统级的验证全景，我们后面会徐徐展开。做为一个落地的教程，我们千里之行始于足下，先从Isabelle/HOL工具的使用开始说起。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---

上一节我们介绍了语言扩展的大致情况，这一节我们开始深入一些细节。

## 诊断信息

语言扩展中一个重要的功能是代码扫描的诊断信息。这个诊断信息是以vscode.Diagnostic为载体呈现的。
我们来看一下vscode.Diagnostic类的成员和与相关类的关系：

![image.png](https://upload-images.jianshu.io/upload_images/1638145-5acbc100be034e9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

以小到大，这些类为：
- Position: 定位到一行上的一个字符的坐标
- Range: 由起点和终点两个Position决定
- Location: 一个Range配上一个URI
- DiagnosticRelatedInformation: 一个Location配一个message
- Diagnostic: 主体是一个message字符串，一个Range和一个DiagnosticRelatedInformation.

### 构造一个诊断信息

下面我们来构造一个诊断信息。
我们随便造一个BASIC语言的例子吧，保存为test.bas:
```basic
dim i as integer

for i = 1 to 10 step 1
    for i = 1 to 10 step 1
        print "*";
    next i
next i
```
这个例子中，循环控制变量在外循环和内循环中被重用，导致外循环失效。
出现问题的Range是第4行的第9字符到第10字符。位置是以0开始的，所以我们构造(3,8)到(3,9)这样两个Position为首尾的Range. 
```typescript
        new vscode.Range(
            new vscode.Position(3, 8), new vscode.Position(3, 9),
        )
```
有了Range，加上问题描述字符串，和问题的严重程序三项，就可以构造一个Diagnostic来。

```typescript
    let diag1: vscode.Diagnostic = new vscode.Diagnostic(
        new vscode.Range(
            new vscode.Position(3, 8), new vscode.Position(3, 9),
        ),
        '循环变量重复赋值',
        vscode.DiagnosticSeverity.Hint,
    )
```

### 诊断相关信息

上一节提到，有Range，有message，有严重程度这三项，就可以构造一个Diagnostic信息出来。

除此之外，还可以设置一些高级信息。
第一个是来源，比如来自eslint某版本，使用了某某规则之类的。这个可以写到Diagnostic的source属性中。
```typescript
diag1.source = 'basic-lint';
```
第二个是错误码，有助于分类和查询。这个是code属性来表示的，既可以是一个数字，也可以是一个字符串。
```typescript
diag1.code = 102;
```
第三个是相关信息。以上节例子来说，我们说i已经被赋值过了，那么可以进一步告诉开发者是在哪里被赋值过了。所以要有一个uri，能找到代码的地址。还要有一个Range，告诉在uri中的具体位置。前面介绍过了，这是一个vscode.Location结构。
```typescript
    diag1.relatedInformation = [new vscode.DiagnosticRelatedInformation(
        new vscode.Location(document.uri,
            new vscode.Range(new vscode.Position(2, 4), new vscode.Position(2, 5))),
        '第一次赋值')];
```

下面我们把它们集合起来，针对上面的test.bas进行错误提示。主要就是将上面的提示信息写到传参进来的DiagnosticCollection中。
```typescript
import * as vscode from 'vscode';
import * as path from 'path';

export function updateDiags(document: vscode.TextDocument,
    collection: vscode.DiagnosticCollection): void {
    let diag1: vscode.Diagnostic = new vscode.Diagnostic(
        new vscode.Range(
            new vscode.Position(3, 8), new vscode.Position(3, 9),
        ),
        '循环变量重复赋值',
        vscode.DiagnosticSeverity.Hint,
    );
    diag1.source = 'basic-lint';
    diag1.relatedInformation = [new vscode.DiagnosticRelatedInformation(
        new vscode.Location(document.uri,
            new vscode.Range(new vscode.Position(2, 4), new vscode.Position(2, 5))),
        '第一次赋值')];
    diag1.code = 102;

    if (document && path.basename(document.uri.fsPath) === 'test.bas') {
        collection.set(document.uri, [diag1]);
    } else {
        collection.clear();
    }
}
```

### 触发诊断信息的事件

下面我们在plugin的activate函数中增加到于刚才写的updateDiags函数的调用。

```typescript
	const diag_coll = vscode.languages.createDiagnosticCollection('basic-lint-1');

	if (vscode.window.activeTextEditor) {
		diag.updateDiags(vscode.window.activeTextEditor.document, diag_coll);
	}

	context.subscriptions.push(vscode.window.onDidChangeActiveTextEditor(
		(e: vscode.TextEditor | undefined) => {
			if (e !== undefined) {
				diag.updateDiags(e.document, diag_coll);
			}
		}));
```

运行一下，在新启动的vscode中打开test.bas，然后在最后任意编辑一下代码，激活事情就可以触发。运行界面如下：
![诊断信息](https://upload-images.jianshu.io/upload_images/1638145-e612db62b73ae5d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从中可以看到，第4行的i变量下面有一个提示，错误码102，source是basic-lint。第二行是DiagnosticRelatedInformation的信息。
