---
tags: [Sentry]
intro: Sentry 如何自定义上报
type: [知识收集]
---
# 自行上报信息
```ts
Sentry.withScope((scope) => {
	scope.setExtra('info', info); Sentry.captureException(error);
});
```