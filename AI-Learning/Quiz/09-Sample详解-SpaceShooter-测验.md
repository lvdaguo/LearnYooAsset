# 测验 · 09 · Sample 详解 · Space Shooter

> **难度：中高**（Sample 你已经自己跑过、打过包了，题目往"代码路径 × 设计决策"走）

---

## Q1. Boot.cs 启动流程的顺序

```csharp
IEnumerator Start()
{
    UniEvent.Initalize();           // ① 为啥先初始化事件系统
    YooAssets.Initialize();         // ② 再初始化 YooAsset
    var go = Resources.Load<GameObject>("PatchWindow");  // ③ 为啥这里用 Resources
    GameObject.Instantiate(go);
    var operation = new PatchOperation("DefaultPackage", PlayMode);  // ④
    YooAssets.StartOperation(operation);
    yield return operation;
    ...
}
```

- a) ①和②的顺序能不能交换？为什么？
- b) 为什么 PatchWindow 要从 `Resources.Load` 加载，而不是 `YooAssets.LoadAssetAsync`？

### 我的答案
_（在此作答）_
a 2会依赖1？我推断的，不知道为啥2要依赖1，uniEvent不是第三方的框架吗，为什么yooasset会耦合
b 因为这是登录界面，本身就是热更新需要用的资源，如果这个也需要热更新，那就循环依赖了吧
---

## Q2. PatchOperation 的基类为什么是 GameAsyncOperation？

```csharp
public class PatchOperation : GameAsyncOperation
{
    protected override void OnStart() { _machine.Run<FsmInitializePackage>(); }
    protected override void OnUpdate() { _machine.Update(); }
    protected override void OnAbort() { }
    public void SetFinish() { Status = EOperationStatus.Succeed; }
}
```

问：
- a) 为什么不直接用 MonoBehaviour 或 普通类？
- b) `yield return operation` 是怎么让协程等它结束的？背后机制（可以联系 OperationSystem）

### 我的答案
_（在此作答）_
a 继承这个可以统一管理？
b 协程就是继承unity的yield instruction就可以了 不知道咋联系的OperationSystem
---

## Q3. 状态机失败回退的实际跳转

玩家进游戏时断网了，`FsmRequestPackageVersion` 失败。用户在弹窗上点"重试"。状态机从哪个节点重新开始？

- A. 从 FsmInitializePackage（重新初始化）
- B. 从 FsmRequestPackageVersion（只重新请求版本）
- C. 从 Boot.cs 重来
- D. 取决于 Sample 怎么实现

### 我的答案
_（在此作答）_
我看看代码去
失败发            PatchEventDefine.PackageVersionRequestFailed.SendEventMessage();
然后弹窗            System.Action callback = () =>
            {
                UserEventDefine.UserTryRequestPackageVersion.SendEventMessage();
            };
            ShowMessageBox($"Failed to request package version, please check the network status.", callback);
点击确定发事件会强制切换状态机状态
            _machine.ChangeState<FsmRequestPackageVersion>();
所以就是B
---

## Q4. 无更新时如何跳过下载

代码里这段逻辑：

```csharp
// FsmCreateDownloader.cs
if (downloader.TotalDownloadCount == 0)
{
    Debug.Log("Not found any download files !");
    _machine.ChangeState<FsmStartGame>();
}
else
{
    PatchEventDefine.FoundUpdateFiles.SendEventMessage(totalDownloadCount, totalDownloadBytes);
}
```

问：
- a) `== 0` 直接跳 FsmStartGame 是合理的设计吗？为什么不跳到 FsmClearCacheBundle？
- b) `else` 分支为什么不直接 `ChangeState<FsmDownloadPackageFiles>`，而是发事件？

### 我的答案
_（在此作答）_
a 可以吧，没更新下载文件为啥要清除文件
b 看了一下代码，发了个事件是一个弹窗，给用户显示更新量让他确定，然后确定了再进入下个状态，直接下载读条
---

## Q5. BattleRoom 的 Handle 管理模式

```csharp
private readonly List<AssetHandle> _handles = new List<AssetHandle>(1000);

// 每次开火 / 敌人生成都往列表加
var handle = YooAssets.LoadAssetAsync<GameObject>("player_bullet");
handle.Completed += h => h.InstantiateSync(...);
_handles.Add(handle);

// DestroyRoom 时统一 Release
foreach (var h in _handles) h.Release();
```

问：
- a) 这种"所有 Handle 攒到一起，退出时统一 Release"的做法，**有什么缺点**？（想想长时间战斗）
- b) 如果要改成"子弹实例消失时就 Release 对应 Handle"，会有什么问题？（回顾 Q5/Q9 of 05 测验）
- c) 工业级项目的 Handle 管理会怎么做？

### 我的答案
_（在此作答）_
a 每次生成都重复申请一次资源，会浪费吧，感觉可以直接缓存加载到的handle，然后每次instantiate（但是LoadAssetAsync会不会有缓存机制，他官方都这么写，是不是没毛病）
b 如果还是a的做法，要release对应handle需要存一个字典吧，然后如果release也没啥效果，就是引用计数一直--，其实也不会释放吧
c 和我a里说的一样都是复用一个handle
---

## Q6. SceneBattle.OnDestroy 的特殊写法

```csharp
private void OnDestroy()
{
    _windowHandle.Release();
    _musicHandle.Release();
    _battleRoom.DestroyRoom();
    if (YooAssets.Initialized)
    {
        var operation = package.UnloadUnusedAssetsAsync();
        operation.WaitForAsyncComplete();  // ← 特别注意
    }
}
```

问：
- a) 为什么这里用 `WaitForAsyncComplete()` 而不是 `yield return`？
- b) 为什么要检查 `YooAssets.Initialized`？什么情况下会是 false？

### 我的答案
_（在此作答）_
a 这个ondestroy不是协程方法啊，用yield return也用不了啊
b 初始化失败了？不太可能吧
---

## Q7. 状态机下载后的"未完待续"

```csharp
// FsmDownloadPackageFiles.cs
downloader.DownloadErrorCallback = PatchEventDefine.WebFileDownloadFailed.SendEventMessage;
downloader.DownloadUpdateCallback = PatchEventDefine.DownloadUpdate.SendEventMessage;
downloader.BeginDownload();
yield return downloader;

if (downloader.Status != EOperationStatus.Succeed)
    yield break;

_machine.ChangeState<FsmDownloadPackageOver>();
```

问：**`yield return downloader` 结束后，流程是跑到 `if` 检查，然后 ChangeState**。  
如果下载中发生**单文件失败**（3 次重试都失败），`DownloadErrorCallback` 被触发。此时 `yield return downloader` 还会结束吗？UI 怎么响应？

（提示：想想 Downloader 的整体 Status 和单文件 Error 的关系）

### 我的答案
_（在此作答）_
文件下载失败3次 DownloadErrorCallback触发 弹窗点击重试
download还是会结束吧，只不过status不会是success了
弹窗重试点击会重新进入上一个状态FsmCreateDownloader（但是到了这个状态不是又会弹一次下载弹窗，这不会有点啰嗦吗）
---

## Q8. EditorSimulateMode 下 8 步状态机跑不跑

你之前问过"EditorSimulateMode 下 8 个阶段还在吗？"。回到这题 —— 看 Sample 实际代码：

```csharp
// FsmRequestPackageVersion.cs（简化）
var operation = package.RequestPackageVersionAsync();
yield return operation;
if (operation.Status != EOperationStatus.Succeed) { 弹窗重试 }
else { _machine.ChangeState<FsmUpdatePackageManifest>(); }
```

问：EditorSimulateMode 下，`package.RequestPackageVersionAsync()`：
- 会不会发 HTTP 请求？
- 返回什么？Status 是什么？
- 最终会 `ChangeState` 吗？

（答完可以不跑代码，但要能从"EditorSimulate 没 CDN"这个前提推理）

### 我的答案
_（在此作答）_
我先推理
1、肯定不会
2、返回的也是operation吧，只不过是编辑器环境特殊的 status肯定 succcess吧
3、会
---

## Q9. 事件系统 UniEvent 的设计价值

Sample 里事件用得很多：
```csharp
PatchEventDefine.FoundUpdateFiles.SendEventMessage(count, bytes);   // 状态机 → UI
UserEventDefine.UserBeginDownloadWebFiles.SendEventMessage();        // UI → 状态机
```

为什么不直接在 FsmCreateDownloader 里调用 `patchWindow.ShowUpdateConfirmDialog(count, bytes)`？

（考察点：耦合设计）

### 我的答案
_（在此作答）_
有点类似MVC思想吧，至少得把ui和逻辑拆分开，两边通过发事件相互交流，方便拓展代码
---

## Q10. 开放题：如果你做自己的热更新，哪些要改？

你之前在 C++/Java/ObjC 搞过热更新系统。现在看完 Sample 后，列 2~3 个你会**抄下来用的设计**，和 2~3 个你**会改的设计**。

（这题没标准答案，考察你的工程判断）

### 我的答案
_（在此作答）_
基本差不多，就是ui弹窗不一样，重试也是各个阶段重试的
---

## AI 批改结果

**总分：77/100 (77%)** — 比 08 测验（54%）回升明显。亮点：Q3 你主动看代码验证得出 B，Q7 的追问"重试会不会又弹一次下载弹窗"非常敏锐，Q6a 直接指出"OnDestroy 不是协程方法"也说明基础扎实。丢分主要在 Q2（不知道 OperationSystem 和 yield 的连接机制）和 Q10（开放题答得敷衍）。

---

### Q1 · 8/10 ✅ b 完美，a 推理错了但诚实承认

**a) 能不能交换？**

你说"2 依赖 1" —— **实际上不依赖**。我查了源码：
- `YooAssets.Initialize()` 只创建 `YooAssetsDriver` GameObject 和启动 OperationSystem，不碰 UniEvent
- `UniEvent.Initalize()` 是 UniFramework 的独立模块，和 YooAsset 无耦合

**真正的约束**：`PatchOperation` 构造函数里会 `_eventGroup.AddListener<...>`，**必须在 PatchOperation 创建前 UniEvent 已初始化**。Boot.cs 里 UniEvent.Initalize() 在 PatchOperation 之前就够，**①②交换也能跑**。

Sample 先 UniEvent 后 YooAssets 的**语义意图**是"先准备事件系统这个基础设施，再准备业务层" —— 但不是技术必须。

**b) 为啥 PatchWindow 走 Resources** ✅ **完全正确**

你答的"循环依赖"是关键词：
- PatchWindow 本身是热更新 UI，必须在 YooAsset 热更新完成前就能显示
- 如果 PatchWindow 放 YooAsset 管理 → 加载它需要 YooAsset 初始化 → 初始化过程失败时没 UI 显示错误 → 用户看黑屏

**经典的"鸡和蛋"问题**。所以热更新 UI 必须走 Resources（或独立嵌入首包）。

> **📖 Q1 标准答案**  
> a) **能交换**。UniEvent 和 YooAssets.Initialize() 互不依赖，只要 UniEvent 在 PatchOperation 创建前初始化即可  
> b) PatchWindow 是热更新 UI 本身，不能被热更新管理（鸡和蛋问题）。走 Resources 保证一定在首包里，YooAsset 初始化前就能用

---

### Q2 · 5/10 ⚠️ a 浅了，b 没说到点子上

**a) 为啥不 MonoBehaviour？**

你答"统一管理" —— 方向对但太笼统。完整原因：

1. **生命周期解耦** —— MonoBehaviour 需要挂在场景里的 GameObject 上，Scene 切换时可能被销毁打断流程；GameAsyncOperation 独立于场景存在
2. **无 GameObject 依赖** —— PatchOperation 在任何上下文都能创建和执行
3. **统一异步模型** —— 所有 YooAsset 内部的 Load/Download/Init 都是 GameAsyncOperation，PatchOperation 跟着这个约定，就能和其他 op 组合、支持 `yield return` / `await` / `Completed +=` 三种等待
4. **OperationSystem 统一调度** —— 每帧 OperationSystem.Update() 遍历所有 Op 的 InternalUpdate，不需要 MonoBehaviour Update 驱动

**b) yield return 的机制 —— 查了源码帮你讲清**

```csharp
// AsyncOperationBase.cs 源码
public abstract class AsyncOperationBase : IEnumerator, IComparable<AsyncOperationBase>
{
    bool IEnumerator.MoveNext() { return !IsDone; }
    object IEnumerator.Current => null;
    // ...
}
```

所以 `yield return operation` 的真实机制：

```
Unity 协程执行到 yield return
  ↓
调 operation.MoveNext()
  ↓
内部 return !IsDone
  ↓ IsDone=false（没完）
协程挂起，下一帧继续
  ↓ 下一帧
OperationSystem.Update()
  ↓
op.InternalUpdate() 跑一次
  ↓
可能设置 Status = Succeed → IsDone 变 true
  ↓
下一帧协程调 MoveNext() → return true → 协程继续往下跑
```

**不是继承 Unity 的 YieldInstruction**（你答的），而是**自己实现 IEnumerator 接口**。这是 YooAsset 能同时支持协程/回调/UniTask 三种等待的底层原因。

> **📖 Q2 标准答案**  
> a) **生命周期解耦**（无 GameObject 依赖）+ **统一异步模型**（能和其他 Op 组合）+ **OperationSystem 统一调度**  
> b) `AsyncOperationBase` **实现 IEnumerator 接口**，`MoveNext() { return !IsDone; }`。yield return 每帧问 MoveNext，IsDone 由 OperationSystem.Update 驱动的 InternalUpdate 设置

---

### Q3 · 10/10 🌟 **主动看代码验证，答案完美**

你按代码路径走得很准：
1. 失败 → `PackageVersionRequestFailed` 事件
2. UI 弹窗显示，回调里发 `UserTryRequestPackageVersion`
3. `PatchOperation.OnHandleEventMessage` 里 `_machine.ChangeState<FsmRequestPackageVersion>()`

**选 B 正确**。这就是 UI 驱动式回退的完整链路。比我直接讲更有说服力 —— **源码 > 直觉**，这是这次测验最棒的答法。

> **📖 Q3 标准答案** **B**。事件链 `失败 → 发事件 → UI弹窗 → 用户点击 → 发用户事件 → 状态机 ChangeState 回对应节点`

---

### Q4 · 9/10 ✅

**a) 直接跳 StartGame 合理吗？**

你说"可以吧，没更新下载文件为啥要清除文件" —— **对**。  
TotalDownloadCount == 0 说明本地 Bundle 已是最新 Manifest 状态，理论上上一次更新时 ClearCacheBundle 已经清过旧文件。跳到 ClearCacheBundle 再跳 StartGame 是多做一次保险，但当前简化成直接 StartGame 也没问题。

**b) else 分支为啥发事件** ✅ **完全正确**

你的答案直指要害："**显示更新量让用户确定**"。

真实场景：玩家在移动网络下进游戏，突然发现要下 500MB 流量，App 自动扣他流量肯定气炸。所以 **"发现更新 → 用户确认 → 开始下载"** 是热更新 UX 的标准。

> **📖 Q4 标准答案**  
> a) 合理。TotalDownloadCount == 0 意味着本地 Bundle 已是最新 Manifest 状态，没旧文件需要清  
> b) 状态机暂停，通过事件让 UI 显示"发现 XX MB 更新，是否继续？"，用户确认后才进入 FsmDownloadPackageFiles。**流量友好 + 用户感知**

---

### Q5 · 8/10 ✅ 三问都有料

**a) 缺点** ⚠️ 你看到了"浪费"，但没说出最严重的问题

你的质疑"官方都这么写，是不是没毛病？" —— **问得对**。**Sample 这里确实不是最优解**，是教学简化。

完整的缺点：
1. **Handle 对象无限增长** —— 长时间战斗累积上万 Handle，列表本身占内存
2. **每次 LoadAssetAsync 创建新 Handle** —— 即使 Provider 去重了 Bundle 加载，Handle 还是新的，引用计数一直 +1
3. **GC 压力** —— 创建回调 lambda + List 扩容

**你的"LoadAssetAsync 会不会有缓存机制" —— 有，但只缓存 Provider 层，不缓存 Handle**。

**b) 子弹消失就 Release 的问题** ⚠️ **更正**：我之前说"Release 会导致 Bundle 卸载白模"是错的，你质疑得对，下面是校准版

你的直觉完全正确：
- YooAsset 有引用计数机制
- **默认 `AutoUnloadBundleWhenUnused = false`**（源码 `InitializeParameters.cs:49` 确认）
- Release 只是减引用计数，Bundle 不会自动卸载，直到手动调 `UnloadUnusedAssetsAsync()` 才真会被清

**"子弹消失就 Release"在 Sample 默认配置下是安全的**，不会白模：
- 情况 1：还有其他活着的子弹 Handle → 引用计数 > 0，Bundle 留着
- 情况 2：这是最后一个 Handle → 引用计数 = 0，但 AutoUnload=false，Bundle 仍在内存，下次 Load 还能用

**真正的缺点**（不是白模）：
1. **需要维护 GameObject → Handle 字典** —— 高频场景下字典增删查询有开销
2. **代码复杂** —— 要确保 Destroy 前先查到对应 Handle 再 Release
3. **容易漏** —— 忘了 Release 就内存泄漏；Release 两次会报错

**Sample 用"统一 Release"的真实原因**：
1. 代码简洁 —— 不用字典映射
2. 教学清晰 —— 先讲"Handle 必须 Release"这个核心，精细化管理是进阶
3. 性能 —— 退场时一把清比每次逐个字典查询快
4. 降低出错 —— 新手不易漏

**什么时候会真出白模**：主动调 `UnloadUnusedAssetsAsync()` + 正好有引用计数=0 的 Bundle + 场景里还有正在加载的资源。这是"Unload + 时序竞争"问题，不是 Release 单独引起。

**c) 工业级做法** ✅

你说"复用一个 handle" —— 对，这是核心。完整方案：
```csharp
// 启动场景时 preload
_bulletHandle = package.LoadAssetAsync<GameObject>("bullet");
yield return _bulletHandle;

// 开火时复用（不再 Load）
var go = _bulletHandle.InstantiateSync(...);
_pool.Add(go);

// 子弹爆炸时回收到池（不 Destroy）
_pool.Recycle(go);

// 退场景时统一销毁 + Release
foreach (var go in _pool) Destroy(go);
_bulletHandle.Release();
```

**Sample 教"Handle 统一管理"的思路，你要改造成"Handle 预加载 + GameObject 对象池"才是生产级**。

> **📖 Q5 标准答案（校准版）**  
> a) Handle 对象无限增长（内存 + GC + 引用计数累积）；Sample 是教学简化，不是最优  
> b) 需要 GO → Handle 字典映射（高频场景有开销）+ 容易漏/重复 Release。**不会白模**（AutoUnload 默认 false，Release 只减引用计数）  
> c) **Handle 预加载一次 + GameObject 对象池复用 + 退场景统一 Release Handle**

---

### Q6 · 8/10 ✅ a 完美

**a)** ✅ 你答"OnDestroy 不是协程方法啊，用 yield return 也用不了" —— **完全正确**。MonoBehaviour 生命周期回调除了 Start 之外都返回 void，只有 Start 能是 IEnumerator。`WaitForAsyncComplete` 是同步等待 —— Operation 内部循环 InternalUpdate 直到 IsDone，阻塞主线程但不会产生跨帧问题。

**b) YooAssets.Initialized 什么时候是 false** ⚠️ 你答太浅

"初始化失败" —— 这只是其中一种。常见情况还有：

1. **Editor 停止 Play 的销毁时序** —— 停止 Play 时 Unity 会乱序 OnDestroy 所有 GameObject。Boot.cs 的 OnDestroy 可能先于 Scene 的 OnDestroy，也可能反过来。如果 Boot 先跑了 `YooAssets.Destroy()` 清理，SceneBattle.OnDestroy 再访问就崩了
2. **手动调用 YooAssets.Destroy()** —— 某些项目在切换主场景或登出账号时主动清理
3. **多场景切换的中间态** —— 卸载场景瞬间

**所以这个判断是防御式编程，避免 NullReferenceException**。

> **📖 Q6 标准答案**  
> a) `OnDestroy()` 返回 void，不能 yield。`WaitForAsyncComplete` 同步等待 Operation 完成（主线程阻塞）  
> b) Editor 停止 Play 的销毁时序乱序 / 主动调 YooAssets.Destroy() / 场景切换中间态。防御式编程避免空引用

---

### Q7 · 9/10 🌟 **你的追问"会不会又弹一次"特别敏锐**

你的答案三点都对：
1. 单文件 3 次失败 → 触发 DownloadErrorCallback ✅
2. Downloader 整体 Status 变 Failed，yield return 会结束 ✅
3. yield break 后状态机停在此节点，等事件切回 FsmCreateDownloader ✅

**你的追问"重试会不会又弹一次下载弹窗"—— 会。原因：**

- 切回 FsmCreateDownloader → 重新 `CreateResourceDownloader()`
- 此时本地已下完的 Bundle 仍在（断点续传保留），TotalDownloadCount **只包含还没下完的文件**
- 所以 FoundUpdateFiles 事件还是会发，弹出"还需下载 XX MB"

**这个"有点啰嗦"的观察是对的**，工业级项目会优化：
- 首次弹窗"发现 500MB 更新"
- 下载失败重试时，直接静默 Resume（或者只弹"网络中断，继续？"的轻提示）
- 而不是再次弹"发现 X MB 更新"这种全局确认

**Sample 用相同流程处理首次和重试，是教学简化**。

> **📖 Q7 标准答案** yield return 会结束（整体 Status=Failed）；ErrorCallback 已经通过事件通知 UI 弹窗；重试会切回 FsmCreateDownloader，重新 CreateResourceDownloader 时因断点续传 TotalDownloadCount 只剩剩余量。确实会"再弹一次"，工业级做法会区分"首次确认"和"重试 Resume"两种 UI

---

### Q8 · 10/10 ✅ 推理全对

你三个推理都对：
1. 不发 HTTP 请求 ✅（EditorSimulate 没 CDN 可访问）
2. 返回 Operation，Status = Succeed ✅（EditorSimulateModeFileSystem 直接返回模拟版本号）
3. 会 ChangeState ✅（if 分支判断 Succeed，进下一步）

**推理准确 —— 说明你已经把"EditorSimulate = 假装网络请求瞬时成功"的模型内化了**。

> **📖 Q8 标准答案**  
> - ❌ 不发 HTTP 请求  
> - ✅ 返回 Operation，**Status = Succeed**（模拟版本号，瞬时完成）  
> - ✅ 会 ChangeState 到下一节点  
> EditorSimulate 下 8 步状态机完整跑完，只是每步都是 0 延迟的 no-op

---

### Q9 · 7/10 ✅ 方向对

你答"MVC思想，UI 和逻辑拆分" —— 对，但可以更深。完整答案：

**事件系统解决的三类问题**：

1. **引用方向解耦** —— 状态机不持有 UI 引用，UI 不持有状态机引用。各自独立编译、测试
2. **生命周期解耦** —— UI 可能还没创建（启动时）、可能已销毁（场景切换），直接调用会空引用
3. **一对多订阅** —— 多个 UI 组件可同时监听一个事件（比如 DownloadUpdate 同时刷新进度条和 Debug 面板）
4. **可替换性** —— 换个 PatchWindow 实现（比如出海版用别的 UI 风格），状态机代码不改

这是 **Observer 模式** 的工业级应用。典型的"MVC 通信总线"。

> **📖 Q9 标准答案** ① 引用方向解耦 ② 生命周期解耦 ③ 一对多订阅 ④ 可替换 UI 实现。Observer 模式

---

### Q10 · 3/10 😄 这题你偷懒了

"基本差不多，就是 ui 弹窗不一样，重试也是各个阶段重试的" —— **信息量太少**。开放题没有错答，但要能体现思考。

**我替你展开几个可以讨论的点，你可以挑感兴趣的补充**：

**值得抄的设计**：
- ✅ **状态机每节点独立 Fsm 文件** —— 比你之前可能的"一个大 switch-case"清晰
- ✅ **UI 驱动失败回退** —— 比自动 retry 更灵活（你之前"15 次 3 轮"的方案在弱网环境好用，但没用户选择权）
- ✅ **GameAsyncOperation + OperationSystem 的统一异步模型** —— 比平台各自线程池好管理

**值得改的设计**：
- ⚠️ **硬编码的 CDN URL 和版本号**（FsmInitializePackage.cs 里 `"http://127.0.0.1"` + `"v1.0"`）→ 生产要从**配置服务**拉
- ⚠️ **BattleRoom 的 Handle 管理**（见 Q5）→ 改成 Handle 预加载 + 对象池
- ⚠️ **HTTP 非 HTTPS** → 正式发布必须 HTTPS
- ⚠️ **无带宽限速** → 你之前做过"令牌桶"限速，YooAsset 需要业务层自己加
- ⚠️ **重试策略太保守**（failedTryAgain=3）→ 弱网可以加到 15 次或指数退避

这题**你自己的项目经验最值钱**，建议之后自己补充两三条具体的。

> **📖 Q10 标准答案** 没有标准答案，但建议列出具体点。参考上方两组清单

---

## 📊 成长曲线

| 测验 | 分数 | 趋势 |
|------|------|------|
| 01 入门概览 | 70% | — |
| 02 核心原理 | 76% | 📈 |
| 03 初始化模式 | 67% | 📉 |
| 04 热更新 | 49% | 📉📉 |
| 05 资源加载 | 79% | 🚀 |
| 06 下载系统 | 76% | ✅ |
| 07 构建系统 | **89%** | 🚀🚀🚀 |
| 08 文件系统 | 54% | 📉（难度回调）|
| **09 Sample 详解** | **77%** | ✅ 回升 |

---

## 🎯 这份测验的最大亮点

**Q3 你主动看代码验证 + Q7 的追问 + Q8 纯推理全对** —— 说明你已经能做到"读代码 + 横向联系 + 正向推理"三件事。这是从"学 YooAsset"升级到"用 YooAsset 改进项目"的关键能力。

---

## 💡 两个需要强化的点

1. **OperationSystem ↔ 协程 yield** 这条链路（Q2b）没通 —— 推荐单独读 `AsyncOperationBase.cs` 的 MoveNext 实现 + `OperationSystem.Update` 的 InternalUpdate 调度，20 分钟能搞通
2. **开放题别偷懒**（Q10）—— 这是测验里最值钱的部分（对自研经验的复盘）

接下来可以做 **10 实战应用指南** 的测验（最后一章），或者**跑一次 HostPlayMode 验证完整热更新流程**。想做哪个？
