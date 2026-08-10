# CabPad-Refactor-Doc（架构升级设计文档）

> 本仓库为**设计文档集合**，不包含任何业务源码，用于说明某工程机械车载HMI（CabPad类项目）
> 从单体Qt应用向4节点微服务架构升级的整体设计思路。配套通用开源中间件见
> [VehicleIpcBus](https://github.com/yourname/VehicleIpcBus)，车云网关见
> [SomeIpGateway](https://github.com/yourname/SomeIpGateway)。

---

## 📑 文档目录

1. [01_现状分析.md](./01_现状分析.md) — 单体架构痛点
2. [02_目标架构.md](./02_目标架构.md) — 4节点微服务拓扑
3. [03_IPC协议设计.md](./03_IPC协议设计.md) — 二进制Header + ServiceID清单
4. [04_CAN_Gateway解耦.md](./04_CAN_Gateway解耦.md) — RawCan + CanManagerQt分层
5. [05_状态机.md](./05_状态机.md) — 车辆连接/建图/运行/故障 状态图
6. [06_故障隔离方案.md](./06_故障隔离方案.md) — 心跳超时 + 降级策略

## 🔗 相关仓库

| 仓库 | 说明 |
|---|---|
| [VehicleIpcBus](https://github.com/yourname/VehicleIpcBus) | 开源UDS+epoll IPC中间件实现 |
| [SomeIpGateway](https://github.com/yourname/SomeIpGateway) | SOME/IP风格车云epoll网关 |
