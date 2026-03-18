# 图床 · Pictures Bed

> 个人图文件仓库，以 Mermaid 格式为主，GitHub 原生渲染。

## 目录结构

```
uml/          UML 架构图（Mermaid）
  goodsop/    GoodSOP 宠物医疗 SOP 平台
    v1.5-recording/  v1.5 录音架构系列
```

## 索引

| 系列 | 文件 | 描述 |
|------|------|------|
| GoodSOP v1.5 录音架构 | [总览图](./uml/goodsop/v1.5-recording/overview.md) | 整体分层、Session ID 双层设计、Strategy 类图、PAD vs Web 对比 |
| GoodSOP v1.5 录音架构 | [PAD 前台录音](./uml/goodsop/v1.5-recording/pad-recording.md) | 组件关系、完整时序（6 阶段）、断线异常、状态机 |
| GoodSOP v1.5 录音架构 | [Web 工作站录音](./uml/goodsop/v1.5-recording/web-recording.md) | 多医生并发时序、声纹校验、超时收尾、断线重连、内存快照 |
