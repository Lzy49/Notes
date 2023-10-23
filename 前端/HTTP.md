---
tags: [HTTP,面试,浏览器,TCP]
intro: HTTP
type: [知识]
---
# HTTP 请求发送过程
![[Pasted image 20231021112721.png]]
## 浏览器发起HTTP请求流程
1. 构建请求行
2. 查找缓存
	3. DNS 缓存: 在浏览器本地把对应的 IP 和域名关联起来，这里就不做过多分析了。
	4. 资源缓存: [HTTP 缓存 - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching)
3. 准备 IP 地址 和 端口号
4. 等待 TCP 队列
5. 建立 TCP 连接
6. 发送 HTTP 请求
	1. 向服务器发送 请求行 : 请求方式, 请求 URL , HTTP 协议版本
	2. 向服务器发送 请求头 : 浏览器的基础信息(操作系统、浏览器内核，当前请求的域名信息、浏览器端的 Cookie 信息)
## 服务器发起HTTP响应流程
- 发送响应报文
	1. 向浏览器发送 响应行: HTTP 协议版本, 状态码
	2. 向浏览器发送 响应头: 
		1. 200 成功头: 服务器自身的一些信息(生成返回数据的时间,返回的数据类型（JSON、HTML、流媒体等类型），Cookie 信息)
		2. 301 重定向: Location 重定向地址 
	3. 发送响应体的数据
- 断开连接
	1. 判断 浏览器响应头中是否存在 `Connection:Keep-Alive` 
	2. 存在 -> 保持 TCP 连接
	3. 不存在 -> 关闭 TCP 连接
### Connection:Keep-Alive
- 发送: 由浏览器向服务器发出
- 用途: TCP 连接在发送后将仍然保持打开状态，这样浏览器就可以继续通过同一个 TCP 连接发送请求。保持 TCP 连接可以省去下次请求时需要建立连接的时间，提升资源加载速度
- 使用场景: 一个 Web 页面中内嵌的图片就都来自同一个 Web 站点，如果初始化了一个持久连接，你就可以复用该连接，以请求其他资源，而不需要重新再建立新的 TCP 连接。
# 报文

## HTTP 请求报文
![[Pasted image 20231021110637.png]]
## HTTP 响应报文
![[Pasted image 20231021111100.png]]

## 缓存报文
[HTTP 缓存 - HTTP | MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Caching)