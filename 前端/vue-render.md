---
tags: [vue]
intro: vue 源码解析
type: [知识]
source: [《Vue.js设计与实现》 ]
---
# 渲染器的创建
- createRenderer(option) -> 创建 一个渲染器 , option 自定义配制对象 是为了实现自定义行为. 调用平台能力 即 浏览器 操作 DOM. 
  - renderer 
    - render 
      - 当有 new vnode -> patch
      - 无 new vnode -> unmount
      - 记录上一次的 vnode
    - patch( old Vnode, new Vnode , container)
      - !old -> 首次挂载 -> mountElement
      - old -> 更新 ->  
        - newVnode.type !== oldVnode.type -> unmount(oldVnode) , oldVnode = null
        - 判断 newVnode.type 与 old.type 是否相同
          - 不同 -> 卸载旧的  -> 存放新的
          - 相同
            - Symbol.Fragment -> 代码片断 -> 无 old patch插入  ? 有 old Diff 
            - Symbol.text -> 文本 -> 无 old 创建 ? 有 old 更新
            - Symbol.Comment -> 注释 -> 无 old 创建 ? 有 old 更新
            - string -> html vnode -> oldVnode ? patchElement : mountElement 
            - object , function -> 组件(render) -> 
              - 是否存在 oldVnode 
                -  是: patchComponent 
                -  否: 
                   - 组件是否是有 keptAlive 
                     - 有 : keepAlive 子组件 且 已被挂载过 -> 调用 keepAlive.active
                     - 无 : mountComponent
            - xx -> TODO
        - 判断 newVnode.children -> patchChildren
    - patchChildren
      - string -> 更新文案即可
      - array -> 
        - diff 算法
          - 取 min(old.length, new.length) -> 进行 path 比较
          - 判断新增 (old.length < new.length) -> 挂载 new 中剩余vnode
          - 判断删除 (old.length > new.length) -> 卸载 old 中剩余vnode
        - key 判断 (想要复用 item 通过 type 判定有点草率 , 所以给 item 定义了 ID_Card)
	        - 动态与静态比较法
		        - 在 vnode 中 层面 , 只有存在响应值的 vnode 才会更新. 所以我们只需要关注 node
		        - 循环处理 vnode 保存的 修改点.
          - 普通比较法
            - 循环旧vnode -> 如果在新vnode中找到相同 key -> 
               - 有 -> patch(该vnode) -> 移动vnode
                 - 循环新旧vnode时 , 记录最大 index.
                 - 当前vnode(旧vnode)的 index < 最大 index
                   - true : 需要移动的vnode -> 需要移动的vnode 移动到 其 prev vnode的后面 . 即 prev vnode的 nextSibling 前面.
                   - false : 更新 最大 index
               - 无 -> 说明vnode是新vnode -> 需要挂载 -> 创建新vnode -> 判断是否有 prev vnode
                 - 有 -> 挂载到容器的,创建vnode的 prev vnode的 nextSibling 的前面. 
                 - 无 -> 挂载到容器的第一个vnode的前面
            - 循环旧vnode -> 判断是否在新vnode中出现过 -> 无 -> unmount(old).
          - 双端比较法
            - 操作
             - 循环判断 新旧vnode队列 . 当 两个队列中有一个结束后,跳出
                - 处理 旧的队列中的 undefined 
                  1. old-start === undefined -> old-start ++ 
                  2. old-end === undefined -> old-end --
                - 处理双端
                  1. 比较 new-start 和 old-start -> 不动 , 更改 new-start++ , old-start++ (因为 new-start 是正确的顺序,所以只移动指针即可.下个操作会在这个 item 之下操作)
                  2. 比较 new-start 和 old-end -> 移动 old-end 到 old-start 上面 -> new-start ++  ,  old-end -- (将 old-end 移动到 old-start 上面, 将 item 移动到了 index 范围外,后续不会对其产生影响)
                  3. 比较 new-end 和 old-end -> 不动 , 更改 new-end -- , old-end --(因为 new-end 是正确的顺序,所以只移动指针即可.下个操作会在这个 item 之下操作)
                  4. 比较 new-end 和 old-start -> 移动 old-start 到 old-end 下面  , new-end -- , old-start ++ (将 old-start 移动到 old-end 下面, 将 item 移动到了 index 范围外,后续不会对其产生影响)
                - 处理双端没有办法确定元素
                  1. 用 new-start 寻找 old list 中 对应的vnode 
                    - 找到: 
                      - 将该vnode移动到 old-start 的 前面
                      - 将 old list 中的 该vnode 设置为 undefined .
                    - 找不到:
                      - 创建新vnode -> 挂载到 old-start 的上面.
             - 如果新的队列中存在未使用 item -> 循环vnode新增到 old-start 之前 ( old-start 会在操作完后++, 所以 old-start 占用的位置是需排序的内容.)
             - 如果就的队列中存在未使用 item -> 循环旧vnode old-start ~ old-end 之间的vnode , unmount 它们 👍🏻.
          - 快速比较法
            - 快速比较法的原理是: 比较两个数据, 从头 从尾比较,找出差异. 使用场景是在一个有序内容中插入一个内容,寻找该内容.
            - 操作:
              - 寻找需要更改的列
                - 比较开始位置 -> 创建 j -> 当 old-list[j] !== new-list[j] -> 跳出
                - 比较结束位置 -> 创建 o-end , n-end ( 因为两个数组长度可能不同) -> old-list[o-end] !== new-list[n-end]  -> 跳出
              - 判定
                - 删除vnode : 当 j > n-end && j < o-end -> new-list 遍历所有vnode 且 仍有old vnode存在 -> 删除 j ~ o-end 之间的 vnode.
                - 新增vnode : 当 j > o-end && J < n-end -> old-list 遍历所有vnode 且 仍有new vnode存在 -> 新增 j ~ n-end 之间的 vnode 到 new-end + 1 为锚点的 vnode 之前.
                - 新老vnode中都有要处理的vnode ( 删 , 增 , 移 -> 找到这些vnode )
                  - 收集要移动的vnode -> 创建 source 保存新旧 list 的映射关系 source = new Array(n-end - j + 1),
                  - [优化] 减少循环找值成本:
                    - 循环 new list - 转环映射表 -> (key -> index)
                    - 创建 count 记录在 old-list 中找到的可复用 vnode 数量
                  - 判定那些点需移动
                    - 创建 lastItem = 0;
                    - 循环 old list -> 
                      - [优化] count >= source.length -> 
                        - true : 卸载 vnode ( 因为 count > length 时新vnode已被用完) 
                        - break
                      - new List 映射表中找-> 
                        - 找到 -> 
                          - source 中 记录 需要移动 
                          - index < lastItem -> 记录 item-index
                          - index > lastItem -> lastItem = index; // 更新最后一个vnode值
                          - [优化] count + 1 ;
                        - 没找到 -> 卸载 vnode.
                  - 在 source 中找出 最小子序列 ,移动其之外的 值
                    - 循环 source 
                      - 当 index 在 最小子序列时 . index = index - 最小子序列.length + 1;
                      - 当 index 不在 最小子序列时
                        - source[index]=== -1 -> 新增到 index + j + 1 的位置.
                        - source[index]!== -1 -> 移动该节点 到 index + j + 1 的vnode前面
    - mountElement(vnode , container) -> 用来完成挂载
      -  把 vnode 解析成真实 dom 
        - 解析 自己的 props -> dom 属性 -> patchProps 
        - 解析 自己的 children
          - string -> 直接 setText
          - Array -> 循环 - patch(null , item ,父vnode)
      -  把解析好的dom 挂载到容器上
    - mountComponent
      - 执行 vnode.type 得到
        - function -> 函数式组件 -> 转为 option.render + option.props
        - object -> 组件 option
      - [生命周期] option.beforeCreate()
      - 处理 props 和 attrs 
        - 循环 vnode.props 判断 instance.props 中是否存在 || 以 on 开头 
          - true : props.push(item) 
          - false : attrs.push( item)
      - 创建 instance 实例 -> 绑定 vnode.component
        - state : reactive 包裹 option.data() 值 为 state
        - isMounted : false 判定是否被挂载
        - subTree : 旧的 vnode 树
        - props : shallowReactive(props)
        - slots : vnode.children -> function[]
        - mounted: [] -> 保存所有的 mounted 生命周期 ( update , unmounted 都如此实现)
      - keepAlive : 判断 vnode.component.__isKeepAlive === true -> 挂载 keepAliveCtx
        - move : insert
        - createElement -> 创建一个元素
      - 设 currentInstance 当前组件为 instance -> 用来服务 setup 中的生命周期钩子
      - setup 生命周期钩子 API
        - onMounted -> 判断当前是否有 currentInstance -> 有 -> 在 setup 中, 收集 fn 到 currentInstance.mounted 中. 
      - 处理 setup 函数
        - props 传入 函数
        - setupContext 传入函数
          - attrs
          - emit -> emit 函数 -> 
            - 参数
              - eventName -> props['on' + eventName]
              - option -> 对应函数需要的参数
            - 做事
              - if props['on' + eventName] -> event(...option) 
        - 执行 option.setup 
        - 判断返回值:
          - function -> 判断 render (无) -> 以 render 方式运行
          - object -> 创建 setup 返回 响应值. 在 context 中代理 
      - 创建 renderContext 代理 instance -> 为 this 取值服务
        - 代理 data 中的值 和 props 中的值. 使 this.xx -> data.xx || props.xx
        - 代理 setupContext 运行后的响应值.
        - 代理 slots 的值 -> 在 render 函数中 会执行到 slot 节点,直接调用 该代理值
      - [生命周期] created.call(renderContext)
      - 增加副作用
        - effect 为 render 添加副作用 & 执行生命周期
          - isMounted :
            - false : 挂载 
              - [生命周期] beforeMount.call(renderContext)
              - path(null, render 产生的vnode)
              - isMounted = true 
              - [生命周期] mounted.foreach(item => item.call(renderContext))
            - true : 更新
              - [生命周期] beforeUpdate.call(renderContext) 
              - patch(instance.subTree , render 产生的vnode)
              - [生命周期] updated.call(renderContext)
          - instance.subTree = subTree
        - 使用 scheduler 拦截 修改 同步到 下一个 事件循环中进行更新.
      - vnode 传入 patch 进行继续渲染.
    - patchComponent
      - new vnode.component = old vnode.component (防止设置新 vnode 后 没有 组件)
      - 判断 props 和 attrs 是否有变换
        - 有: 更改 props 和 attrs -> props 的 响应性质 -> 更新组件
        - 无: 无操作
    - unmount()
      - 判断 vnode.type === object
        - 判断 vnode.shouldKeepAlive 
          - 是: 缓存组件 -> 不卸载 -> 使用 其父 deactive 
          - 否: unmount(vnode.subTree)
      - 判断 vnode.type === Fragment -> vnode.children.foreach(item => unmount(item))
      - 其他 -> js 移除旧vnode ( node.el.parent.remove(node.el))
  - option
    - patchProps -> 
      - 处理属性
        - 判断 props 是 class or style -> 指令正常化处理 -> 使用对应高效方案处理
        - 判断 props 是否只能通过 setAttribute 设置 ( condition : 只读属性 映射表 ) 
        - 判断 props 是否在 el 上存在
          - 存在 : [typeof el.xx  === boolean && '' => true] -> 使用 el.xx = xx 来处理
          - 不存在: 使用 setAttribute 来处理
      - 处理事件
        - 以 on 开头的是事件.
        - 以 伪造事件处理函数(invoker) 执行事件
          - 记录 invoker 绑定 el 高精时间
          - 事件传入
            - 如果 invoker 存在 -> 替换
            - 如果 invoker 不存在 -> 安装
          - 无事件传入
            - 如果 invoker 存在 -> removeEventLister
          - 妙: 
            - 以中间函数调用 props 中的事件, 可以节省 addEventLister 消耗
            - 以中间函数调用 props 中的事件, 方便 模板修饰符.
            - 以数组 + 字符串形势 管理 props 中重复事件 , 以编译时节省运行时.
            - 无事件卸载 中间函数 , 有事件使用中间函数 ,冲突更改事件不会调用 addEventLister
        - 事件执行
          - 获取当前时间和绑定事件比较 如果 绑定时间 > 当前 -> return (为了处理 绑定事件发生在事件冒泡之前.)
          - 执行事件.
- defineAsyncComponent (将异步组件转为同步组件)
  - 将异步组件转为同步组件
    - 利用响应值属性 -> 抛出一个组件 -> 组件 render 中包含一个响应值 -> 响应值为 false 展示加载中 -> 响应值 为 true 返回 该组件. (响应值在 组件加载完成后改变状态)
- keepAlive 
  - props : include : 要缓存  , excludes : 不缓存 , cache : 用来保存所有的 缓存组件. 
    - cache : 定义了一套接口, 传入值必须符合该接口定义.且该值为了服务缓存太多问题. 故在 值中应自定义 修剪策略 常用策略为 队列 即 先入先嘎.
  - 定义标识 (__isKeepAlive): 标识其子组件为被缓存组件
  - 缓存组件: 创建 cache 用来保存其子缓存组件的实例.
  - 缓存DOM
    - 创建 storageContainer -> 不挂载到 body 只 , 只存在 js 里
    - 定义 active(move 元素 -> 其父内) & deactive ( move 元素 -> storageContainer)
  - 返回 render
    - 判断 子组件.name 在 include 中 || 子组件.name 不在 excludes 中 
      - 是: 
        - 给子组件设置不卸载标识(shouldKeepAlive)
        - 给子组件设置父组件为 keepAlive
        - 对子组件进行选择处理:
          - 第一次进入 : 只缓存 -> 返回子组件
          - 第二次进入 : 查缓存 -> 标记已缓存(keptAlive) -> 返回子组件
      - 否 -> 返回子组件
- Teleport : 传送门
  - __isTeleport : 标识是传送门组件 
  - process: 提供处理子节点 的方法
    - 接收 : n1, n2, container, anchor, internals
      - internals : 
        - patch
        - patchChildren
        - move
        - unmount
    - 做事: 
      - Teleport.props.to -> 获取真是DOM
        - !n1 ? n2.foreach(item => patch(null,n2,Teleport.props.to,...)) : patchChildren
- Transition 
  - 原理 : 在将 el 挂载到 parent 之前给 该 el 增加 class 保证其效果. 在 效果执行完毕后 使用 transitionend 检测动画结束 执行下个生命周期 -> 当要 卸载时先执行动画 再调用卸载. 
  - 底层组件:因为牵涉阻塞挂载与阻塞卸载两个无法控制的行为.故其是地层组件

## 知识
1. getAttribute 对于一些属性只会取其初始值 例如 input.value 
2. HTML Attributes 可能关联多个 DOM Properties。
3. HTML Attributes 的作用是设置与之对应的 DOM Properties 的初始值。
# renderer 和 reactivity 如何联系
在响应式的副作用函数中 执行渲染器函数 . 当渲染器函数中引用了响应数据 .就可以触发响应.
