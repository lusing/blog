# vscode插件快餐教程(2) - 编程语言扩展

上一节我们学习了如何写一个控制光标的vscode命令插件。
对于一个编辑器来说，编辑命令是非常重要的部分。不过vscode更主要的作用不是写文本，而是写代码。所以我们第二讲就直入辅助编写代码的部分。

## 可以做哪些编程语言相关的扩展

我们先看一张图，看看vscode支持我们做哪些编程语言的扩展。

![vscode语言扩展](https://upload-images.jianshu.io/upload_images/1638145-52502847b4949267.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们以Bill Gates起家的BASIC语言的一个小子集为例来展示下如何使进行编程语言的扩展。

首先，我们在package.json下的contributes下增加对于语言配置的支持：
```json
        "languages": [{
            "id": "basic",
            "extensions": [
                ".bas"
            ],
            "configuration": "./language-configuration.json"
        }
```

### 注释

BASIC语言中使用\'来表示单行注释，用/' '/来表示多行注释。我们这样来写language-configuation.json:
```json
    "comments": {
        "lineComment": "'",
        "blockComment": [
            "/'",
            "'/"
        ]
    }
```

在传统Basic语言中，使用REM语句来表示注释，我们可以写成下面这样：
```json
    "comments": {
        "lineComment": "REM ",
        "blockComment": [
            "/'",
            "'/"
        ]
    },
```

定义之后，我们就可以用Ctrl+K(Windows)或者Cmd-K(Mac)来触发打开或关闭注释了

### 括号匹配

我们对小括号和中括号进行配对：
```json
    "brackets": [
        [
            "[",
            "]"
        ],
        [
            "(",
            ")"
        ],
    ],
```

### 括号的自动补齐

可以通过括号的自动补齐功能来防止少写一半括号：
```json
    "autoClosingPairs": [
        {
            "open": "\"",
            "close": "\""
        },
        {
            "open": "[",
            "close": "]"
        },
        {
            "open": "(",
            "close": ")"
        },
        {
            "open": "Sub",
            "close": "End Sub"
        }
    ]
```
在上例中，输入一个"，就会补上另一半"。对于其他括号也是如此。

### 选中区域加括号

在选中一个区域之后，再输入一半括号，就可以自动用一对完整括号将其包围起来，称为auto surrounding功能。

例：
```json
    "surroundingPairs": [
        [
            "[",
            "]"
        ],
        [
            "(",
            ")"
        ],
        [
            "\"",
            "\""
        ],
        [
            "'",
            "'",
        ]
    ],
```


### 代码折叠

函数和代码块多了以后，给代码阅读带来一定困难。我们可以选择将一个代码块折叠起来。这也是Vim和emacs时代就有的老功能了。

我们以折叠Sub/End Sub为例，看看代码折叠的写法：

```json
    "folding": {
        "markers": {
            "start": "^\\s*Sub.*",
            "end": "^\\s*End\\s*Sub.*"
        }
    }
```

我们来看下Sub折叠后的效果：
![sub](https://upload-images.jianshu.io/upload_images/1638145-90e420d1087e2cd2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

