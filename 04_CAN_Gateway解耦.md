# 04 — CAN_Gateway 解耦（Qt 依赖剥离）

## 4.1 拆分前 CanManager（问题）

```
CanManager : public QObject
├── Q_OBJECT + moc（依赖Qt元对象系统）
├── QSocketNotifier（绑定Qt主循环，无法独立进程使用）
├── QTimer 心跳
├── 业务解析：syncMsg_17FF0001 / emitUiUpdate17FE0001（耦合业务）
└── 缓存：m_msg_17FE0000/0001/0002 + m_msg_17FF0000/0001/0002
```

→ **问题**：CanManager 同时是硬件驱动 + 业务协议解析 + 状态缓存 + UI信号发射器。拆分后必须分层。

## 4.2 拆分后分层

```
┌─────────────────────────────────────────────────────────┐
│                HMI_UI 进程（保留向后兼容）                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │  CanManagerQt（可选兼容层，非必须）                 │  │
│  │     继承QObject，不直接碰SocketCAN                 │  │
│  │     内部用 NodeClient → 发REQ给CAN_Gateway         │  │
│  │     收NOTIFY → 转成Qt信号给旧UI（迁移成本为0）       │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                           │ UDS
                           ▼
┌─────────────────────────────────────────────────────────┐
│              CAN_Gateway 独立进程（无Qt依赖）              │
│  ┌───────────────────────────────────────────────────┐  │
│  │  RawCan（纯 C++17，可移植）                         │  │
│  │    open(dev)  :: socket() + bind() + ioctl(SIOCGIFINDEX)│  │
│  │    close()                                          │  │
│  │    sendFrame(id,data,dlc) :: write()                │  │
│  │    epollReadLoop()  ← 独立线程，读到帧回调           │  │
│  └───────────────────────────────────────────────────┘  │
│                           ▲                              │
│                           │ 函数调用（进程内）            │
│  ┌───────────────────────────────────────────────────┐  │
│  │  NodeClient（vipc库）                               │  │
│  │    ↔ REGISTER SVC_CAN_GATEWAY                      │  │
│  │    ↔ 收到SEND_FRAME → RawCan::sendFrame            │  │
│  │    ↔ RawCan读回调  → publish(NOTIFY_RECV_FRAME)     │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
         │
         ▼ SocketCAN
       车载VCU
```

## 4.3 RawCan API（头文件草案）

```cpp
// can_gateway/RawCan.h
class RawCan {
public:
    using FrameCb = std::function<void(uint32_t can_id,
                                       const uint8_t* data,
                                       uint8_t dlc)>;
    bool open(const std::string& dev = "vcan0");
    void close();
    bool sendFrame(uint32_t can_id, const uint8_t* data, uint8_t dlc);
    void setCallback(FrameCb cb);   // 在epoll线程调用
    void start();  // 启动epoll读线程
    void stop();
};
```

## 4.4 兼容性迁移步骤（零宕机）

1. **Step 1**：新增 RawCan + CAN_Gateway 独立进程（HMI 暂时仍走旧 CanManager，**两系统并行**）
2. **Step 2**：HMI 新增编译宏 `VIPC_CAN`，启用后将 `CanManager::sendFrame()` 改为 `NodeClient sendRequest(CAN_GW::SEND_FRAME)`
3. **Step 3**：Stress Test 通过后，删除旧 CanManager 中 SocketCAN 相关代码，保留其作为 QObject 信号转发壳
4. **Step 4**：State_Manager 接手所有 `m_msg_17FF0000` 解析（CanManager 不再解析业务）

## 4.5 故障隔离实现

```
CAN_Gateway 进程崩溃（如 SIGSEGV）
    ↓
supervisord / systemd 用户服务 → 500ms 内自动重启
    ↓
启动后 REGISTER SVC_CAN_GATEWAY
    ↓
MessageBus 更新路由表 → 后续 REQUEST 重新可达
    ↓
State_Manager 检测到 REGISTER → 推送 "CAN 已恢复" NOTIFY → UI 解除灰显
```
