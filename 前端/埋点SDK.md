---
tags: [业务设计]
intro: 埋点SDK
type: [思考]
---
# 设计初衷与针对用户
## 设计出发点
### 初衷

- 为客户提供有效数据支持. 
- 分析数据优化产品. 
- 提供个性化数据报表.
- 提供用户动线数据.
### 针对3D项目
- 记录用户动线 , 优化场景设计动线 , 优化场景设计,动线设计.
- 记录各场景停留时间 , 为售前提供数据支持和客户引导
- 记录 npc 与用户交互 , 分析用户选择偏好. 优化美术设计
- 针对 用户角色选择 分析 美术偏好. 优化美术设计
### 为什么不使用3方
1. 数据保密.
2. 报表个性化.
3. 个性化埋点.
4. 随项目积累,降低成本.
## 设计
### 整体设计
1. 上报机制
2. 埋点类型 和 接入方式
### 上报机制
#### 数据分类
1. 持续上报 : 记录用户在项目中的活跃性, 使用心跳来处理 .
2. 紧要上报 : 重要数据 马上上报
3. 次要上报 : 不重要的数据 等心跳一起上报.
#### 设计
1. 上报内容: 采用队列设计 , 每一个上报都会将一条数据塞入队列中.
2. 上报时机: 
	1. 有心跳 : 紧要上报 马上上报 , 心跳隔断配制时间上报 心跳 + 次要上报
	2. 无心跳 : 紧要上报 马上上报 , 次要上报 积累到 配制数量 上报
### 埋点类型
#### 心跳
1. 难点:
	1. 判断失活 : 以 hasFocus visibilityState 两个 API 判断失活. 
2. 上传间隔 : 
	1. 以 settimeOut 来处理 心跳 , 依次递归运行
	2. 当 紧急上报执行时 , 并发心跳重置 settimeout .
3. 上报内容:
	1. 顺序: 心跳顺序值, 从上次叠加
	2. 时间: 心跳时间
	3. 位置: 心跳所在路由 , 3D 空间场景值 .
4. 接入方式:
	1. 引入 SDK 初始化后自动运行
5. 价值:
	1. 在线时间相关:
		1. 用户在一次使用中的时间
		2. 用户二次 ,三次 使用时间 增减情况
		3. 用户在路由,场景中的在线时间.
	2. 路由场景相关:
		1. 用户动线
		2. 在某一场景中的用户行为.
		3. 分析场景时间原因
		4. 场景 PV UV 停留时间
#### 事件相关
1. 事件类型: 点击, 聚焦, 曝光, 失焦, 鼠标移入, 长按
2. 上报: 紧要上报
3. 上报内容: 类型 , 路由场景 , 相关必要值 , 上次上报值
4. 接入方式:
	1. 声明式 : vue 指令
	2. 命令式 : sdk function 调用.
5. 价值
	1. 记录用户行为动线 
	2. 分析单一核心业务
#### 路由场景切换
1. 上报: 次要上报
2. 上报内容: 路由场景 , 时间, 上次上报值 ( 原因:没有心跳或行为是依次无效的场景切换,不能作为有效数据)
3. 接入方式:
	1. 自动在 Router 钩子记录
	2. 手动记录
4. 价值
	1. 场景动线路
	2. 场景行为记录
	3. 处理bug依据
#### 接口信息
1. 上报: 次要上报
2. 上报内容: 接口调用,调用情况,返回值
3. 接入方式:初始化自动接入 
4. 价值:
	1. bug 解决
	2. 逻辑 review
	3. 动线记录
#### 长页面热区分析
1. 上报: 次要上报
2. 上报内容: 长页面内容相关值, 当前滚动条位置 和所在时间 .
3. 接入方式:
	1. 声明式 : vue 指令
	2. 命令式 : 函数调用
4. 价值:
	1. 分析长页面热区
	2. 分析长页面内容并优化
	3. 更精确的的给出事件上报位置.
#### 分享裂变数据
1. 上报: 紧要上报
2. 上报内容: 上级,自己,时间,渠道
3. 接入方式: 命令式
4. 价值:
	1. 产生列表数据报表
	2. bug 纠错
## 报表设计
1. 心跳: 
	1. 页面心跳: 
		1. 路由:
			1. 柱状图 : 分析数据多少
			2. 折线图 : 分析页面趋势
		2. 用户
			1. 数据列表: 分析活性
			2. 折线图: 分析用户 2次激活概率,停留时间
2. 事件相关:
	1. 单一事件: 
		1. PV UV  核心数据 
3. 用户动线
	1. 流程图
4. 分享
	1. 分享关系:力导图
	2. 用户为统计点: 柱状图
