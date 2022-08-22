# vscode插件快餐教程(1) - 从写命令开始

大致从2017年开始，vscode就越来越流行。vscode能够流行起来，除了功能强大、微软不断升级给力之外，优秀的插件机制也是非常重要的一环。vscode中相当多的功能也是通过自身的插件机制实现的。
比起使用coffeescript为主要开发语言的atom IDE，vscode使用越来越有王者气质的typescript做为主要的开发语言，这也为vscode插件开发提供了良好的助力。
随着插件机制的不断完善，文档、示例与脚手架工具等也日渐成熟。相关的文章与教程也非常丰富，现在写vscode的plugin已经是件比较容易的事情了。
自己的业务开发，只有自己最了解，作为程序员，写自己的plugin来加速自己的开发效率，现在正是好时机。
vscode的plugin种类非常丰富，我们先从最传统的定义新命令说起吧。

## 使用脚手架生成骨架

与其他主流前端工程一样，我们通过脚手架来生成插件的骨架。微软提供了基于脚手架生成工具yeoman的脚本。
![yeoman](https://upload-images.jianshu.io/upload_images/1638145-46f844dc3d4e3d56.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们通过npm来安装vscode plugin脚手架：
```
npm install -g yo generator-code
```

然后我们就可以通过yo code命令来生成vscode plugin骨架代码了。

```
yo code
```

脚手架会提示选择生成的plugin的类型：

```
     _-----_     ╭──────────────────────────╮
    |       |    │   Welcome to the Visual  │
    |--(o)--|    │   Studio Code Extension  │
   `---------´   │        generator!        │
    ( _´U`_ )    ╰──────────────────────────╯
    /___A___\   /
     |  ~  |     
   __'.___.'__   
 ´   `  |° ´ Y ` 

? What type of extension do you want to create? (Use arrow keys)
❯ New Extension (TypeScript) 
  New Extension (JavaScript) 
  New Color Theme 
  New Language Support 
  New Code Snippets 
  New Keymap 
  New Extension Pack 
```

我们就选择New Extension (Typescript)。关于其他选择我们后面的文章会继续介绍。

我们从最简单的移动光标开始做起吧。比如写一个将光标移动到编辑区首，比如一篇文章或代码的首部，用emacs的命令叫做move-beginning-of-buffer。

## 光标移动命令实践

同是编辑器，都有相似的功能。这种移动到文章首的功能，肯定不用我们自己开发，vscode早就为我们做好了。在vscode中定义了一大堆光标控制的命令，比如这条就叫做cursorTop。

* cursorTop: 移动到文章首
* cursorBottom: 移动到文章尾
* cursorRight: 光标向右移，相当于emacs的forward-char
* cursorLeft: 光标左移，相当于emacs的backward-char
* cursorDown: 向下移一行，相当于emacs的next-line
* cursorUp: 向上移一行，相当于emacs的previous-line
* cursorLineStart: 移至行首，相当于emacs的move-beginning-of-line
* cursorLineEnd: 移至行尾，相当于emacs的move-end-of-line

我们新建一个move.ts，用于实现光标移动的功能。我们先以移动到文章首为例。我们通过vscode.commands.executeCommand函数来执行命令。

例:
```typescript
import * as vscode from 'vscode';

export function moveBeginningOfBuffer(): void {
    vscode.commands.executeCommand('cursorTop');
}
```

在主文件extension.ts中，我们先把move.ts这个包引入进来：

```ts
import * as move from './move';
```

然后在activate函数中，注册这个命令：

```ts
	let disposable_begin_buffer = vscode.commands.registerCommand('extension.littleemacs.moveBeginningOfBuffer',
		move.moveBeginningOfBuffer);

	context.subscriptions.push(disposable_begin_buffer);
```

最后我们在package.json中给其绑定一个快捷键。'<'和'>'在键盘中是上排键，我们就不用它们了，比如绑定到alt-[上。
修改contributes部分如下：
```json
    "contributes": {
        "commands": [{
            "command": "extension.littleemacs.moveBeginningOfBuffer",
            "title": "move-beginning-of-buffer"
        }],
        "keybindings": [{
            "command": "extension.littleemacs.moveBeginningOfBuffer",
            "key": "alt+["
        }]
    }
```

大功告成。我们用F5来启动调试，就会启动一个新的vscode界面。
我们在新开的vscode界面，打开命令窗口（F1或shift-cmd-p），输入move-beginning-of-buffer就可以看到这个命令了：
![move-beginning-of-buffer](https://upload-images.jianshu.io/upload_images/1638145-664063ea6b1af4b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以移到文件中间的位置，按alt+[就会回到文件头。

目前完整的extension.ts的代码为：
```ts
// The module 'vscode' contains the VS Code extensibility API
// Import the module and reference it with the alias vscode in your code below
import * as vscode from 'vscode';
import * as move from './move';

// this method is called when your extension is activated
// your extension is activated the very first time the command is executed
export function activate(context: vscode.ExtensionContext) {

	// Use the console to output diagnostic information (console.log) and errors (console.error)
	// This line of code will only be executed once when your extension is activated
	console.log('Congratulations, your extension "littleemacs" is now active!');

	// The command has been defined in the package.json file
	// Now provide the implementation of the command with registerCommand
	// The commandId parameter must match the command field in package.json
	let disposable_begin_buffer = vscode.commands.registerCommand('extension.littleemacs.moveBeginningOfBuffer',
		move.moveBeginningOfBuffer);

	context.subscriptions.push(disposable_begin_buffer);
}

// this method is called when your extension is deactivated
export function deactivate() {
	console.log('Plugin deactivated');
}
```

对应的完整package.json为：
```json
{
    "name": "littleemacs",
    "displayName": "littleemacs",
    "description": "Some operations just like emacs",
    "version": "0.0.1",
    "engines": {
        "vscode": "^1.33.0"
    },
    "categories": [
        "Other"
    ],
    "activationEvents": [
        "onCommand:extension.littleemacs.moveBeginningOfBuffer"
    ],
    "main": "./out/extension.js",
    "contributes": {
        "commands": [{
            "command": "extension.littleemacs.moveBeginningOfBuffer",
            "title": "move-beginning-of-buffer"
        }],
        "keybindings": [{
            "command": "extension.littleemacs.moveBeginningOfBuffer",
            "key": "alt+["
        }]
    },
    "scripts": {
        "vscode:prepublish": "yarn run compile",
        "compile": "tsc -p ./",
        "watch": "tsc -watch -p ./",
        "postinstall": "node ./node_modules/vscode/bin/install",
        "test": "yarn run compile && node ./node_modules/vscode/bin/test"
    },
    "devDependencies": {
        "typescript": "^3.3.1",
        "vscode": "^1.1.28",
        "tslint": "^5.12.1",
        "@types/node": "^10.12.21",
        "@types/mocha": "^2.2.42"
    }
}
```

## 一个插件实现多条命令

一个不够，我们再写一个：
activate函数：
```ts
	let disposable_begin_buffer = vscode.commands.registerCommand('extension.littleemacs.beginningOfBuffer',
		move.beginningOfBuffer);

	let disposable_end_buffer = vscode.commands.registerCommand('extension.littleemacs.endOfBuffer',
		move.endOfBuffer);

	context.subscriptions.push(disposable_begin_buffer);
	context.subscriptions.push(disposable_end_buffer);
```

功能实现函数：
```ts
import * as vscode from 'vscode';

export function beginningOfBuffer(): void {
    vscode.commands.executeCommand('cursorTop');
}

export function endOfBuffer() {
    vscode.commands.executeCommand('cursorBottom');
}
```

package.json:
```json
    "activationEvents": [
        "onCommand:extension.littleemacs.beginningOfBuffer",
        "onCommand:extension.littleemacs.endOfBuffer"
    ],
    "main": "./out/extension.js",
    "contributes": {
        "commands": [{
                "command": "extension.littleemacs.beginningOfBuffer",
                "title": "beginning-of-buffer"
            },
            {
                "command": "extension.littleemacs.endOfBuffer",
                "title": "end-of-buffer"
            }
        ],
        "keybindings": [{
                "command": "extension.littleemacs.beginningOfBuffer",
                "key": "alt+["
            },
            {
                "command": "extension.littleemacs.endOfBuffer",
                "key": "alt+]"
            }
        ]
    },
```
