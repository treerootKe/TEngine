# TEngine 桐源码阅读手册

基于本仓库源码逐行精读产出，供学习 TEngine 框架使用。所有 API、路径、菜单均以本仓库实际代码为准（TEngine 6.0.0 / YooAsset 2.3.17 / UniTask 2.5.11 / Unity 2021.3.45f2c1 中国版）。

## 文档目录

| 文档 | 内容 | 适合场景 |
|---|---|---|
| [01-架构总览与启动流程](01-架构总览与启动流程.md) | 分层结构、启动链每一步在干什么、HybridCLR 交接点 | 第一次接触本项目，先读这篇 |
| [02-模块系统与核心基础](02-模块系统与核心基础.md) | ModuleSystem/RootModule/UpdateDriver/Settings/Log/MemoryPool/Utility | 理解"模块"如何被创建和轮询 |
| [03-资源模块](03-资源模块.md) | 加载/释放全链路、加载预制体/图片示例、AB 打包与从 AB 加载、多包、加密 | 业务开发每天都要用 |
| [04-UI模块](04-UI模块.md) | 窗口栈/层级/生命周期、WindowAttribute、ScriptGenerator、UIWidget、新增一个 UI 的完整步骤 | 写界面前必读 |
| [05-事件模块](05-事件模块.md) | int/string 事件、[EventInterface] 接口事件、UI 内事件自动注销 | 模块间通信 |
| [06-音频-计时-场景-对象池](06-音频-计时-场景-对象池.md) | Audio/Timer/Scene/ObjectPool 四个功能模块 API 手册 | 用到时查 |
| [07-热更新HybridCLR](07-热更新HybridCLR.md) | UpdateSetting 配置、AOT 补元数据、热更 DLL 加载、打包菜单 | 理解"热更新代码如何跑起来" |
| [08-其他模块与工具](08-其他模块与工具.md) | 调试器、本地化、扩展库(Json/Tween)、Launcher 启动 UI、Fsm、编辑器工具 | 拓展了解 |

## 推荐阅读顺序

1. 01 总览 → 02 模块系统（框架骨架）
2. 07 热更新（搞清楚"哪段代码是热更的"）
3. 03 资源 → 04 UI → 05 事件（业务三大件）
4. 06/08 按需查阅

## 运行方式

- 打开 `UnityProject/Assets/Scenes/main.unity` 直接 Play。
- 编辑器内资源模式由 TEngine 的 EditorPlayMode 决定（见 03 篇"运行模式"）。

## 目录约定速查

```
UnityProject/Assets/
├── Launcher/              # AOT 主包：启动器 UI（LauncherMgr/LoadUpdateUI）
├── GameScripts/
│   ├── GameEntry.cs       # AOT 主包：游戏入口 MonoBehaviour
│   ├── Procedure/         # AOT 主包：11 个启动流程 + 基类
│   └── HotFix/
│       ├── GameProto/     # 热更程序集：协议
│       └── GameLogic/     # 热更主逻辑：GameApp/GameModule/UIModule/SingletonSystem
├── TEngine/               # 框架本体（AOT，Runtime + Editor）
│   ├── Runtime/Core/      # ModuleSystem、GameEvent、Log、MemoryPool、Utility
│   └── Runtime/Module/    # Resource/UI以外的框架模块
├── AssetRaw/              # 所有可打包资源（AB 收集根目录）
└── AssetArt/              # 美术工程资源
Configs/（仓库根）          # 游戏配置源（GameConfig）
```
