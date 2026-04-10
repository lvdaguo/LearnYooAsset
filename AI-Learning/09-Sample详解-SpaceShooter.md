# 09 · Sample 详解 · Space Shooter

> 路径：`Assets/YooAsset/Samples~/Space Shooter/`
> 这是 YooAsset 官方最完整的示例，覆盖了**初始化 → 热更新 → 资源加载 → 内存管理**全流程。

---

## 项目结构

```
Space Shooter/
├── Boot.unity              ← 启动场景（只含 Boot.cs 挂载的 GameObject）
├── GameRes/                ← 游戏资源（Prefab、音频、场景、配置等）
├── GameScript/
│   └── Runtime/
│       ├── Boot.cs                 ← 游戏启动入口
│       ├── PatchLogic/             ← 热更新状态机
│       │   ├── PatchOperation.cs   ← 热更新自定义 Operation
│       │   └── FsmNode/            ← 各状态节点（8个）
│       ├── GameLogic/
│       │   ├── SceneHome.cs        ← 主页场景逻辑
│       │   └── SceneBattle.cs      ← 战斗场景逻辑
│       ├── BattleLogic/
│       │   ├── BattleRoom.cs       ← 战斗核心：动态加载所有战斗实体
│       │   └── Entity*.cs          ← 各类实体（玩家/敌人/子弹/特效/陨石）
│       └── WindowLogic/            ← UI 窗口
└── ThirdParty/
    └── UniFramework/               ← 事件系统(UniEvent)、状态机(UniMachine)
```

---

## 启动流程（Boot.cs）

```csharp
IEnumerator Start()
{
    UniEvent.Initalize();           // 1. 初始化事件系统
    YooAssets.Initialize();         // 2. 初始化 YooAsset

    // 3. 用 Resources.Load 加载更新 UI（这是唯一不经过 YooAsset 的资源！）
    var go = Resources.Load<GameObject>("PatchWindow");
    GameObject.Instantiate(go);

    // 4. 启动热更新自定义 Operation
    var operation = new PatchOperation("DefaultPackage", PlayMode);
    YooAssets.StartOperation(operation);
    yield return operation;          // 等待热更新完成

    // 5. 设置默认包，进入游戏主场景
    var gamePackage = YooAssets.GetPackage("DefaultPackage");
    YooAssets.SetDefaultPackage(gamePackage);
    SceneEventDefine.ChangeToHomeScene.SendEventMessage();
}
```

**关键设计**：更新 UI（进度条、重试按钮）本身不能放在 YooAsset 管理的包里，所以放在 `Resources` 目录，是热更新期间唯一的内置加载。

---

## 热更新状态机（PatchOperation + 8个 FsmNode）

`PatchOperation` 继承 `GameAsyncOperation`，内部驱动一个状态机：

```
FsmInitializePackage
    └─[成功]→ FsmRequestPackageVersion
                  └─[成功]→ FsmUpdatePackageManifest
                                └─[成功]→ FsmCreateDownloader
                                              ├─[无更新]→ FsmStartGame
                                              └─[有更新，用户确认]→ FsmDownloadPackageFiles
                                                                        └─[成功]→ FsmDownloadPackageOver
                                                                                      └─→ FsmClearCacheBundle
                                                                                              └─→ FsmStartGame
```

### FsmInitializePackage — 四种模式分支

```csharp
// EditorSimulateMode：模拟构建
var buildResult = EditorSimulateModeHelper.SimulateBuild(packageName);
var parameters = new EditorSimulateModeParameters();
parameters.EditorFileSystemParameters =
    FileSystemParameters.CreateDefaultEditorFileSystemParameters(buildResult.PackageRootDirectory);

// OfflinePlayMode：纯内置
var parameters = new OfflinePlayModeParameters();
parameters.BuildinFileSystemParameters =
    FileSystemParameters.CreateDefaultBuildinFileSystemParameters();

// HostPlayMode：内置 + CDN 缓存
var parameters = new HostPlayModeParameters();
parameters.BuildinFileSystemParameters =
    FileSystemParameters.CreateDefaultBuildinFileSystemParameters();
parameters.CacheFileSystemParameters =
    FileSystemParameters.CreateDefaultCacheFileSystemParameters(remoteServices);
```

### FsmCreateDownloader — "暂停"设计

```csharp
var downloader = package.CreateResourceDownloader(10, 3);

if (downloader.TotalDownloadCount == 0)
{
    // 无需下载，直接跳到 FsmStartGame
    _machine.ChangeState<FsmStartGame>();
}
else
{
    // 发现更新文件 → 暂停状态机，通知 UI 显示下载提示
    // 等待用户点击"开始下载"按钮后，UI 发送事件继续
    PatchEventDefine.FoundUpdateFiles.SendEventMessage(totalCount, totalBytes);
}
```

用户交互通过 UniEvent 双向驱动：
- 框架 → UI：`PatchEventDefine.FoundUpdateFiles.SendEventMessage()`
- UI → 框架：`UserEventDefine.UserBeginDownloadWebFiles.SendEventMessage()`

---

## 战斗场景资源加载（BattleRoom.cs）

BattleRoom 展示了**高频异步加载 + 统一管理 Handle** 的最佳实践：

### 加载模式：回调式

```csharp
// 所有 Handle 统一存到列表
private readonly List<AssetHandle> _handles = new List<AssetHandle>(1000);

// 加载并立即实例化（不等待，用回调）
var assetHandle = YooAssets.LoadAssetAsync<GameObject>("player_ship");
assetHandle.Completed += (AssetHandle handle) =>
{
    handle.InstantiateSync(_roomRoot.transform);
};
_handles.Add(assetHandle);  // 加入统一管理列表
```

注意：`LoadAssetAsync` 在 `Completed` 回调已注册后，如果**当帧已完成**也会触发回调，所以先注册回调，再加入列表，这个顺序是安全的。

### 统一释放

```csharp
public void DestroyRoom()
{
    foreach (var handle in _handles)
        handle.Release();       // 批量释放所有 Handle
    _handles.Clear();
}
```

---

## SceneBattle.cs — 场景级资源管理

```csharp
// Start：加载场景资源
_windowHandle = YooAssets.LoadAssetAsync<GameObject>("UIBattle");
yield return _windowHandle;
_windowHandle.InstantiateSync(CanvasDesktop.transform);

_musicHandle = YooAssets.LoadAssetAsync<AudioClip>("music_background");
yield return _musicHandle;
audioSource.clip = _musicHandle.AssetObject as AudioClip;

// 场景进入后清理无用资源
var package = YooAssets.GetPackage("DefaultPackage");
yield return package.UnloadUnusedAssetsAsync();

// OnDestroy：释放所有 Handle + 清理资源
_windowHandle.Release();
_musicHandle.Release();
_battleRoom.DestroyRoom();
// 同步等待清理（OnDestroy 中不能 yield）
package.UnloadUnusedAssetsAsync().WaitForAsyncComplete();
```

---

## 可寻址地址使用示例

Sample 中使用的都是**短地址**（AddressByFileName 规则），在 Collector 配置中设置：

| 短地址 | 对应资源 |
|--------|---------|
| `"UIHome"` | Assets/GameRes/...UIHome.prefab |
| `"UIBattle"` | Assets/GameRes/...UIBattle.prefab |
| `"player_ship"` | Assets/GameRes/...player_ship.prefab |
| `"enemy_ship"` | Assets/GameRes/...enemy_ship.prefab |
| `"music_background"` | Assets/GameRes/Audio/...mp3 |
| `"asteroid01~03"` | Assets/GameRes/...陨石 Prefab |
| `"explosion_*"` | Assets/GameRes/...爆炸特效 Prefab |

---

## 核心设计模式总结

| 模式 | 在 Sample 中的体现 |
|------|-----------------|
| **自定义 Operation** | `PatchOperation` 封装完整热更新流程，外部只需 `yield return` 一句 |
| **状态机驱动复杂流程** | 8 个 FsmNode 解耦每个步骤，失败时只需重进对应状态 |
| **事件驱动 UI 交互** | 框架和 UI 互不直接引用，通过 UniEvent 解耦 |
| **Handle 统一管理** | `_handles` 列表统一追踪，`DestroyRoom` 时批量 Release |
| **场景切换时清理** | 每个场景 `OnDestroy` 中 Release + `UnloadUnusedAssetsAsync` |
| **更新 UI 走 Resources** | 热更新进度 UI 不受 YooAsset 管理，避免鸡生蛋问题 |

---

下一步 → [10-实战应用指南](10-实战应用指南.md)
