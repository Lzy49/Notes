---
tags: [vue]
intro: vue 源码解析
type: [笔记]
source: [《Vue.js设计与实现》 ]
---

# 编译器
- 作用 ：将 源代码 转换为 目标代码的一个工具
## 概念
- 源码 ：要被转换的代码
- 目标代码：源码转换成的代码
- 编译：源码 -> 目标代码的过程。
- 编译常见流程：
	- 源代码 -> 
		- 编译前端：词法分析 -> 语法分析 -> 语义分析
		- 编译后端：中间代码生成-> 优化-> 目标代码生成 
	- 目标代码
# vue compiler
- 作用： 将 template -> JSDOM 操作
- 转换过程： 
template 模板 -> parse （词法分析 + 语法分析） -> 模板 ast -> transform(语义分析 + 转换 JavaScript AST) ->  JavaScript AST -> generate -> 渲染函数
![[Pasted image 20231005210808.png]]
## [扩展] parse - token 解决版
### 1、parse - tokenize
#### 行为
将 Template 字符串 截取为 Token 数组 。
即将 源代码
```html
<p> vue </p>
```
转换为 目标代码
```js
[
	{type:"tag" , name :'p'},
 	{type:"text" , content :'vue'},
  	{type:"tagEnd" , name :'p'}
]
```
#### 操作
##### 有限状态自动机
1. 有限
2. 在不同的条件下切换状态
##### vue 有限状机 将 template -> token 数组
- 设计 状态
	- 初始状态
	- 标签开始状态:
		- <
		- / -> 结束标签状态 
		- string -> 标签名称状态 [ 收集 token 字符串]
	- 标签名称状态
		- string -> [ 收集 token 字符串]
	   - > -> 标签结束状态 [ 保存当前 token 字符串]  
	- 文本状态
		- string -> [ 收集 token 字符串]
		- < -> 切换到 标签开始状态 [ 保存当前 token 字符串]  
	- 结束标签状态
		- string -> 结束标签名称状态 [ 收集 token 字符串]
	- 结束标签名称状态
		- string -> [ 收集 token 字符串]
		- > -> 切换状态到 初始化状态 [ 保存当前 token 字符串] 
- 每个状态主要做：
	1. 切换状态
	2. 消费传入模板
	3. 收集 token 内容
	4. 收集 token 为 token 数组集和
- 状态机使用 -> 循环template -> 匹配状态收集 -> 返回 token 数组
### 2. parse - other
#### 行为
将 tokens -> JavaScript AST
将
```js
[
	{type:"tag" , name :'div'},
	{type:"tag" , name :'p'},
 	{type:"text" , content :'vue'},
  	{type:"tagEnd" , name :'p'},
	{type:"tagEnd" , name :'div'},
]
```
转换为
```js
const ast = {
  // 逻辑根节点
  type: 'Root',
  children: [
    // div 标签节点
    {
      type: 'Element',
      tag: 'div',
      children: [
        // h1 标签节点
        {
          type: 'Element',
          tag: 'p',
          children:[
            'vue'
          ]
        }
      ]
    }
  ]
}
```
#### 操作
- 建立 一个 elementStack 压入 elemnt
- 建立 Root 根节点 ，并压入 elementStack 中
- 循环 tokens 
	- tag
		- 压入 elementStack 中
		- parent.children.push(it)
	- text
		- parent.children.push(it)
	- tagEnd
		- 弹出 elementStack
## 1. parse 
#### 行为
将 Template 字符串 截取为 Token 数组 。
即将 源代码
```html
<p> vue </p>
```
转换为 目标代码
```js
[
	{type:"tag" , name :'p'},
 	{type:"text" , content :'vue'},
  	{type:"tagEnd" , name :'p'}
]
```
### 操作
- 定义状态表，区分当前状态
- 定义 context 上下文 和 ancestors 记录层级关系
	- source : 当前整个 template 模板 (`\<div>\</div>`)
	- advanceBy : 消费 source
	- advanceSpaces : 消费 source 的 空白字符
- 定义 Root 节点 ：
	- 守卫打平模板的开始和结束
	- 为后续 render 函数拼接做准备
- root.children = parseChildren
- parseChildren： 解析子节点 [有限状态机]
	- 状态机跳出条件：
		- source.length <= 0 : 模板结束
		- source 以 ancestors 最后一个 开始
			- 守卫 source 以 ancestors 的其他 item 开始 -> source 少写了 闭合标签
	- 状态机状态
		- < :
			- \<xxx : html 标签 -> parseElement
		   - \<!-- : 注释 -> parseComment
	   - {{ : vue 模板内容 -> parseInterpolation
		- 其他：
			- 文本 ：parseText
- parseElement: 
	- 使用 parseTag 获取一个 el 节点
		- 如果是自闭合直接返回 el
	- 通过 tag 切换模式
		- `tag === title or tag === textarea` -> RCDATA
		- `/style|xmp|iframe|noembed|noframes|noscript/` -> RAWTEXT
		- other -> DATA
	- 保存 该 el 到 ancestors 中
	- 解析其子节点 ：el.children = parseChildren()
	- 弹出 ancestors 栈顶
	- 使用 parseTag 消费 source
   - 返回这个 el
- parseTag:
	- 参数
		- context
		- type : end & start
	- 操作
		- 处理 tagName
			- 获匹配正则 ：type -> 活得正则 ： 消费 < & </
			- 拿到 tag 
			- 消费 source
		- 处理 attrs
			- 消费 空白
			- 循环匹配 字符串
				- 收集 attrs-tag
					- 判断 vue 标签和普通标签 -> 以 v- , @ ,: 开头是 vue 标签
						- [优化点] 
							- 记录 patchFlag -> 如果有 `:` 说明是有 attrs 动态值. 需要在响应值修改时进行更新.
							- 收集该 DOM 到 该组件的 vnode 上. 
				- 以 = 分隔
				- 收集 attrs-value
			- 得到 {tag,value} array
		- 处理 结束标签
			- 判断自闭合
			- 消费 > & \>
		- 返回 tag 相关 el 对象
			- tagName
			- type
			- isSelfClosing
			- props
			- children
- parseText: 处理文本的
	- 确定 文本长度
		- start : 0
		- end : < , {{ 为结束符 -> < , {{ 先出现的位置 
	- 创建 text 节点
	- 消费 source (end)
- parseComment: 处理 注释
	- 确定 文本长度
		- start: 0
		- end : --> 
	- 创建 comment 节点
	- 消费 source
- parseInterpolation 
	- [优化点] 
		- 当内容是 {{}} 时节点会记录 patchFlag -> 表示内容是动态的.
		- 收集该 DOM 到 该组件的 vnode 上. 
	- 确定 文本长度
		- start: 0
		- end : }} 
	- 创建 interpolation 节点
	- 消费 source
#### 状态表
| 模式    | 是否能解析标签 | 是否支持 HTML 实在体 | 进入标签                                                           |
| ------- | -------------- | -------------------- | ------------------------------------------------------------------ |
| DATA    | 能             | 是                   | 其他                                                               |
| RCDATA  | 否             | 是                   | \<title\> 、\<textarea\> 标签                                      |
| RAWTEXT | 否             | 否                   | \<style>、\<xmp>、\<iframe>、\<noembed>、\<noframes>、 \<noscript> |
| CDATA   | 否             | 否                   | \<!\[CDATA\[                                                       |
## 2. transfrom
### 行为
给 JavaScript AST 打补丁:
1. 生成 AST 的 jsNode 即 描述 JavaScript AST 的各项属性
```js
const FunctionDeclNode = {
  type: 'FunctionDecl', // 代表该节点是函数声明
  // 函数的名称是一个标识符，标识符本身也是一个节点
  id: {
    type: 'Identifier',
    name: 'render' // name 用来存储标识符的名称，在这里它就是渲染函数的名称 render
  },
  params: [], // 参数，目前渲染函数还不需要参数，所以这里是一个空数组
  // 渲染函数的函数体只有一个语句，即 return 语句
  body: [
    {
      type: 'ReturnStatement',
      return: null // 暂时留空，在后续讲解中补全
    }
  ]
}
```
例如 ：
- 将 AST 中的 p element 的 class 修改为 'p'
- 将 AST 中
### 操作
- contexnt 创建上下文保存相关信息 ，为了方便后续使用 
	- parant 
	- currentNode 
	- childrenIndex 
	- transforms  插件
- traverseNode 用来递归 JavaScript AST tree 
	- 修改 当前环境 context.currentNode
	- 运行 transforms 中的插件 并收集副作用
	- 递归 子节点 并更改 context.childrenIndex 和 context.parent
	- 倒序运行 transforms 插件产生的副作用
- 注意点： 插件顺序是 a -> b -> c 副作用顺序 ：c -> b -> a
- transforms 插件：
	- 设置 item.jsNode -> 描述其 js 代码的数组
		- p -> h('p', null ,children)
## 3. generate 生成器
#### 行为
循环 AST 生成 JavaScript AST
```html
<p> vue </p>
```
->
```js
function render () {
  return h('div', [h('p', 'Vue'), h('p', 'Template')])
}
```
#### 操作
- 创建 context 上下文
	- genXX 定义 处理 jsNode 类型接口 
	- index 记录当前层次 
	- code 记录整个 Javascript AST
	- push 拼接 Javascript AST 的接口
- 递归 上面生成的 AST
	- 获取 节点的 jsNode 
	- 通过 context 的 方法产生 对应内容
	- 利用 context.push 拼接 JavaScript AST 树
- 生成 ast 树有缩进 -> 通过 index 来处理缩进关系 ，每有 {} 即可缩进2格
# 总结
Vue compiler 是 将 template 转为 JavaScript 函数表达式 。
经过 3 个步骤 ：
- parse : 将 template 解析为 模板 AST 树 .
- tranfrom : 给 模板 AST 树打补丁 ， 并 绑上 JavaScript Node .
- generate : 利用 JavaScript AST 的 JsNode 生成 函数代码 .



# 临时
1. block -> 收集 组件收集 动态内容 ,  v-if 来控制 , -> 变来变去 -> 




