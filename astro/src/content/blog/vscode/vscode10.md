---
title: "vscode插件快餐教程(10) - 设置"
description: "本文讨论如何在插件中读取和写入配置选项，以根据用户的环境和个性化需求进行设置。作者解释了如何使用`vscode.workspace.getConfiguration()`方法来获取所有设置，以及如何通过指定前缀来获取特定类别的设置。本文还涵盖了如何使用`contributes`部分在`package.json`文件中添加自定义配置选项。作者提供了处理缺失配置选项的解决方案，并解释了配置值的优先级，包括默认值、全局值、工作区值和工作区目录值。最后，本文演示了如何使用`inspect()`方法来检查配置选项的值。"
pubDate: "Jul 01 2022"
heroImage: "https://upload-images.jianshu.io/upload_images/1638145-8e4275af60afb16e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240"
---


在插件中，根据用户的环境和个性化的不同，需要增加一些配置项。

## 读写配置项

可以通过vscode.workspace.getConfiguration()方法来获取所有的设置项。

```typescript
let config: vscode.WorkspaceConfiguration = vscode.workspace.getConfiguration();
```

设置项可以分类，可以指定某一类前缀，来获取这一类的所有属性，例：
```js
let config2: vscode.WorkspaceConfiguration = vscode.workspace.getConfiguration('launch');
```

获取的配置对象将包括：launch.configurations:Array和launch.compounds等子项。

取得到全部或者分类之后，就可以根据名称来通过get<T>方法来读取配置项的值了：

例，我们读取一个tab等于多少个空格的选项：
```typescript
	let editorConfig: vscode.WorkspaceConfiguration = vscode.workspace.getConfiguration('editor');
	let tabsize = editorConfig.get<number>('tabSize');
```

同样，我们可以通过WorkspaceConfiguration的update方法来更新配置项，例：

```typescript
	editorConfig.update('tabSize',4);
```

## 添加自己的配置项

在package.json中，contributes下通过configuration来添加设置项，例：

```json
	"contributes": {
		"configuration": {
			"title": "BanmaOS CodeComplete Configuration",
			"properties": {
				"banmaos.enableCodeCompletion": {
					"type": "boolean",
					"default": false,
					"description": "Whether to enable AliOS code completion"
				},
				"banmaos.codeCompletionServer": {
					"type": "string",
					"default": "",
					"description": "Server list for code completion"
				}
			}
		}
	}
```

运行后打开settings，显示出来是这个样子的：
![settings](https://upload-images.jianshu.io/upload_images/1638145-63e5fb79fa1ff641.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 指定缺省值

自己设置的配置项可能都设了默认值，但是如果是依赖别人的配置项，可能因为环境不同就没有这个项目。这时有三个解决方案：

1. 假设存在，然后判断undefined
2. 先用has方法判断下存不存在
3. 指定一个默认值, get方法的第二个函数就是干这个的：

```typescript
let whatisthis = editorConfig.get<string>('notExisted',"Unknown value");
```

因为这个项目就是没定义，所以返回值为Unknown value。

## 值的优先级

在目前的版本中，配置项的值有4种：默值值、全局值、工作区值、工作区目录值。
除了默认值不能改之外，其他三个值的类型叫做ConfigurationTarget，分为Global，Workspace和WorkspaceFolder三种。
前面讲到的update方法的第三个参数就是ConfigurationTarget值。可以直接指定ConfigurationTarget，也可以给一个布尔值，true表示Global，false表示Workspace。如果为空或者undefined，则优先选WorkspaceFolder，如果不适用则自动适配为Workspace. 

例：带有ConfigurationTarget的更新：
```typescript
	editorConfig.update('tabSize',4, vscode.ConfigurationTarget.Global);
	editorConfig.update('tabSize',4, true);
```

值这么多有点乱哈，所以vscode为我们提供了一个方法去检查这个配置项的所有值，即inspect方法。我们看个例子：
```js
console.log(editorConfig.inspect<string>('tabSize'));
```

输出的对象类似这样：
```
key:"editor.tabSize"
defaultValue:4
globalValue:4
workspaceValue:4
```
