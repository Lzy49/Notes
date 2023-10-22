---
tags: [Vue, 面试]
intro: Vue2 和 Vue3 的重要差异
type: [知识]
source: [https://time.geekbang.org/column/article/428904]
---
# 源代码层面
## 由面向对象改为函数式编程
代码风格由 面向对象转变为函数式编程.
## 模块重新划分管理
- Vue3 重新拆分模块并使用 monorepo 进行单独管理.
- Vue3 将模块拆分为  reactivity, runtime 和 compiler 三部分
- 模块可以单独使用 , 便于打包
## 由 TypeScript 重写
所有模块由 TypeScript 重写, 全面支持 TypeScript 
## 自定义渲染器
- Vue3 将与 浏览器相关的操作进行依赖倒置处理 . 由外部传入, 增加自定义渲染器机制.
- 通过 `createRenderer` 函数可以创建出不同渲染平台的项目
## 响应系统更改
- Vue3 使用 `Proxy` 代替 Vue2 的 `Object.defineProperty` 对对象进行代理.
- 支持 Set , Map 等多个新的数据类型响应
- 修复 数组更新问题.
	- 数组修改长度不更新
	- 需要使用 $set 更新
## diff 算法的优化
- flag 系统加入 
	- 动态节点收集 -> 知道改了那个
	- 动态值类型打 flag -> 知道改了什么
- diff 算法重写 -> 提升对比两个数组的能力
# 使用
## Composition API 组合语法
- 新增 setup 语法糖 , 和 hooks 钩子函数 . 
- 同一逻辑可以写在一起,避免 vue2 data methed 反复横跳
- 同逻辑更容易抽离, 剃除 mixin .
- 更有利于 Tree-shaking 清理代码
- TypeScript 支持
## 新组件加入
- Fragment: 不再要求有一个唯一的根节点，清除了很多无用的占位 div。
- Teleport: 允许组件渲染在别的元素内
- Suspense: 异步组件，更方便开发有异步请求的组件。

# 生态
## 新增打包工具 Vite 
## 更换状态管理工具 vuex 到 pinia

## 更换测试工具 jest 到 vitest
- [[Vitest vs jest]]
## RFC 机制
关于 Vue 的新语法或者新功能的讨论，都会先在 [GitHub](https://github.com/vuejs/rfcs) 上公开征求意见，邀请社区所有的人一起讨论，任何人都可以围观、参与讨论和尝试实现。

# 总结
- 源码
	- 整体
		- 由面向对象改为函数式
		- 模块重新划分为 reactivity, runtime 和 compiler 并使用 monorepo 管理
		- 由 TypeScript 重写
	- 重要功能点
		- 增加自定义渲染器功能
		- 更改响应式系统为 `Proxy` 代替 `Object.definProperty`
		- diff 算法进行优化
			- flag 系统
			- diff 算法重写
- 使用
	- Composition API 加入 setup 语法糖
		- 由 mixin -> use
		- 由 选项 -> 模块化
	- 新组件加入
		- Fragment
		- Teleport
		- Suspense
- 生态
	- vite
	- pinia
	- vitest
	- RFC 讨论机制