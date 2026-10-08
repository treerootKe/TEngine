# 07 热更新（HybridCLR）与打包

核心源码：`Assets/GameScripts/Procedure/ProcedureLoadAssembly.cs`（268 行，值得整篇精读）
配置资产：`Assets/TEngine/Settings/UpdateSetting.asset`（`Settings.UpdateSetting`）

## 1. UpdateSetting 关键字段（UpdateSetting.cs）

| 字段 | 值（本仓库默认） | 含义 |
|---|---|---|
| `HotUpdateAssemblies` | GameProto.dll, GameLogic.dll | 热更程序集清单（与 HybridCLR 全局设置同步） |
| `LogicMainDllName` | GameLogic.dll | 主逻辑程序集（Entrance 反射入口所在） |
| `AOTMetaAssemblies` | mscor/System/System.Core/TEngine.Runtime/UniTask/YooAsset | 需要**补充元数据**的 AOT 程序集 |
| `AssemblyTextAssetPath` | AssetRaw/DLL | 热更 dll(bytes) 所在目录（被 AB 收集器 DLL 组覆盖） |
| `AssemblyTextAssetExtension` | .bytes | dll 伪装成 TextAsset 的扩展名 |
| `UpdateStyle` | Force / Optional | 断网时是否强制更新 |
| `ResDownLoadPath / Fallback` | http://127.0.0.1:8081(:8082)/项目名/平台 | HostPlayMode 的资源服务器 |
| `ReplaceAssetPathWithAddress` | false | 可寻址模式（省清单内存，地址≠文件名时用） |

`Enable` 属性 = 是否定义了 `ENABLE_HYBRIDCLR` 宏（菜单 `HybridCLR/Define Symbols/Enable|Disable HybridCLR` 切换）。

## 2. ProcedureLoadAssembly 逐段解读（热更加载三步曲）

**第 1 步 · AOT 补充元数据**（LoadMetadataForAOTAssembly, :224-254）
```csharp
HomologousImageMode mode = HomologousImageMode.SuperSet;
HybridCLR.RuntimeApi.LoadMetadataForAOTAssembly(dllBytes, mode);
```
- 目的：AOT 程序集被 IL2CPP 裁剪后缺元数据，热更代码里用到泛型组合时运行时找不到 → 补充后自动降级为解释执行
- 加载的 bytes 是**打包时 IL2CPP strip 后**的同版本 dll（HybridCLR 构建流程自动复制到 `HybridCLRData/AssembliesPostIl2CppStrip/` 再转入 AssetRaw/DLL）
- 注意：给 **AOT** 补，不是给热更 dll 补；编辑器下跳过（`#if !UNITY_EDITOR`，:58-63）

**第 2 步 · 加载热更程序集**（LoadAssembly, :50-108）
- EditorSimulateMode 或宏关闭 → `GetMainLogicAssembly()` 直接从 `AppDomain.CurrentDomain.GetAssemblies()` 找（编辑器下热更代码本来就编进去了，:70-73）
- 真机 → 遍历 `HotUpdateAssemblies`，`LoadAssetAsync<TextAsset>(地址)` 读 bytes → `Assembly.Load(bytes)`（:92-94, :199）；主逻辑程序集记入 `_mainLogicAssembly`
- 计数器 + OnUpdate 轮询等两批（热更 dll + 元数据 dll）全部就绪（:110-122）

**第 3 步 · 反射交接**（AllAssemblyLoadComplete, :124-150）
```csharp
ChangeState<ProcedureStartGame>(...);              // 先切流程（收起启动UI）
var appType  = _mainLogicAssembly.GetType("GameApp");
var entry    = appType.GetMethod("Entrance");
entry.Invoke(appType, new object[]{ new object[]{ _hotfixAssemblyList } });
```
GameApp 在**全局命名空间**（不带 namespace），因为反射用的是字面名 `"GameApp"`；它标了 `#pragma warning disable CS0436`（与生成代码协同）。

## 3. 完整打包/热更操作流（真机）

**前置**：菜单 `HybridCLR/Define Symbols/Enable HybridCLR`；HybridCLR 已 Install（`HybridCLR/Installer`，仓库已装在 `HybridCLRData/`）。

```
1. HybridCLR/Generate/All  或  BuildDLLCommand（Assets/TEngine/Editor/HybridCLR/）
   → 编译热更 dll → strip 后的 AOT dll 一起复制到 AssetRaw/DLL（.bytes）
   菜单快捷项：HybridCLR/Build/BuildAssets And CopyTo AssemblyTextAssetPath
2. TEngine/Build/一键打包AssetBundle _F8    （或 YooAsset/AssetBundle Builder）
   → 资源+DLL 一起进 AB；产物在平台输出目录
3. 首包进 StreamingAssets（UpdateSetting 勾 isAutoAssetCopeToBuildAddress 或手动拷贝）
4. 增量热更：把新版本 bundles + 清单上传到 ResDownLoadPath 服务器
5. TEngine/Build/一键打包Android/IOS/Window  出母包
```
仓库根还有 `BuildCLI/`（命令行打包）与 `Tools/unity.exe`（unity-cli）可用于自动化。

**运行时版本链回顾**（01 篇流程 4-8 步）：RequestPackageVersion → UpdatePackageManifest → CreateResourceDownloader → BeginDownload → ClearCache。版本号回写 PlayerPrefs("GAME_VERSION")，作为下次断网兜底。

## 4. 程序集边界（改代码前必看）

- 热更程序集（GameLogic/GameProto）**不得**引用 Editor 程序集
- AOT（Assembly-CSharp/Launcher/TEngine.Runtime）**不得**引用热更程序集 —— 交接只能靠：反射（ProcedureLoadAssembly）、事件（GameEvent）、接口注入
- 新增热更程序集：UpdateSetting.HotUpdateAssemblies + HybridCLR 设置两侧都要加，并重新 Generate

## 5. 常见故障定位

| 症状 | 根因 |
|---|---|
| `PackageManifest_DefaultPackage.version 404` | Offline/Host 模式 StreamingAssets 没有清单（见 ProcedureInitPackage.cs:102 的专门提示） |
| `Main logic assembly missing` | ENABLE_HYBRIDCLR 未定义或 AssetRaw/DLL 缺 GameLogic.bytes（:132） |
| 真机某泛型调用抛 ExecutionEngineException | AOTMetaAssemblies 漏了对应程序集，补上并重走打包链 |
| 热更没生效 | 只出了包没传服务器，或 Player 没走到 Download 流程（版本号未变） |
