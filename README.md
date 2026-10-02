# Unity 热更新框架（AB 增量更新）

基于 AssetBundle + FTP 的轻量热更新框架：编辑器出包工具 + 客户端增量更新 + AB 加载器。

## 仓库内容

| 文件 | 说明 |
|---|---|
| `HotUpdate.unitypackage` | Unity 导入包（编辑器工具 + 运行时脚本） |
| `README.md` | 使用说明：三个目录的分工、发布步骤、加载 API、注意事项 |

## 怎么用

1. Unity 里 `Assets → Import Package → Custom Package...`，导入 `HotUpdate.unitypackage`
2. 菜单 `AB包工具 / 打开工具窗口` 打开出包工具
3. 按 README 里「发布新版本的 4 步」操作

---
# 热更新框架 使用说明

> 适用工程：HotUpdate（Unity 2021.3）

## 一、先记住这三个目录（核心）

| 目录 | 里面放什么 | 谁放进去的 | 客户端会用吗 |
|---|---|---|---|
| `Assets/ArtRes/AB/<平台>/`<br>例：`AB/PC`、`AB/IOS`、`AB/Android` | 打好的**全部** AB 包 + `ABCompareInfo.txt` | 你打 AB 时输出到这里；工具窗口也从这里生成对比文件、上传 | ❌ 不会。它只是**发布源** |
| `Assets/StreamingAssets/` | **首包**：想跟游戏包一起发出去的 AB + `ABCompareInfo.txt` | 从 `ArtRes/AB/<平台>` 里**挑一部分**复制过来（工具窗口「保存默认资源到StreamingAssets」） | ✅ 会。首次启动就有的资源，用这里 |
| `persistentDataPath`<br>（可读写目录） | **热更下载下来的** AB + `ABCompareInfo.txt` | 运行时自动下载 | ✅ 会，而且**优先用它**（它比首包新） |

一句话：**`ArtRes/AB/<平台>` 是发布源 → 挑一部分进 `StreamingAssets` 当首包 → 剩下的靠客户端下载到 `persistentDataPath`。**

### 一个资源从打包到运行的完整旅程

```
① 打 AB       产出到   Assets/ArtRes/AB/PC/          （prefab、lua、camera、PC 清单包）
② 挑首包      复制到   Assets/StreamingAssets/        （比如只带 lua、PC）
③ 生成清单             ArtRes/AB/PC/ABCompareInfo.txt        ← 给服务器用
                       Assets/StreamingAssets/ABCompareInfo.txt ← 给首包用
④ 上传        传到     服务器 ftp://<IP>/AB/PC/        （AB 本体 + 对比文件，.manifest 不上传）
⑤ 客户端启动   比对服务器清单 vs 本地清单 → 缺的/变了的下载到 persistentDataPath/
⑥ 加载资源     先找 persistentDataPath/，找不到再找 StreamingAssets/
```

> 客户端**只认后两个目录**：`persistentDataPath` 优先、`StreamingAssets` 兜底。

## 二、有哪些功能

| 模块 | 文件 | 功能 |
|---|---|---|
| 编辑器工具窗口 | `Editor/ABTools.cs` | 选平台、改服务器地址、生成对比文件、保存首包、上传资源 |
| 热更管理器 | `Scripts/ABUpdateManager.cs` | 启动时检查更新：增量下载、自动删除已下架的旧包 |
| AB 加载器 | `Scripts/ABManager.cs` | 加载资源、自动处理依赖包、卸载包 |
| 单例基类 | `Scripts/Singleton/` | `SingletonAutoMono<T>`：自动创建、过场景不销毁 |
| 启动入口 | `Scripts/Main.cs` | 调用检查更新 |

框架的能力：增量更新（只下新增/变化的包）、自动清理已下架的包、首包+热更两套资源、多平台分目录、更新与下载进度回调。

## 三、编辑器工具窗口

菜单：`AB包工具 / 打开工具窗口`

| 控件 | 作用 |
|---|---|
| 平台页签 PC / IOS / Android | 选中后，后续操作都针对 `ArtRes/AB/<该平台>` |
| 资源服务器IP地址 + 「更改」 | 改服务器地址并同步给客户端（窗口里显示的就是客户端当前在用的地址） |
| 「创建对比文件」 | 生成两份清单：`ArtRes/AB/<平台>/` 和 `StreamingAssets/` 各一份；目录为空则跳过 |
| 「保存默认资源到StreamingAssets」 | 把 Project 窗口里**选中**的 AB 复制进 `StreamingAssets`（= 定首包内容） |
| 「上传AB包和对比文件」 | 把 `ArtRes/AB/<平台>/` 下的 AB + 对比文件上传到 `/AB/<平台>/` |

## 四、发布新版本的 4 步

| 步骤 | 操作 | 结果落在哪 |
|---|---|---|
| 1. 打 AB | `Window → AssetBundle Browser`，输出目录设为 `Assets/ArtRes/AB/<平台>` | `ArtRes/AB/<平台>/` |
| 2. 生成对比文件 | 工具窗口选好平台 → 点「创建对比文件」 | `ArtRes/AB/<平台>/ABCompareInfo.txt` + `StreamingAssets/ABCompareInfo.txt` |
| 3. 准备首包（可选） | Project 里选中要随包发布的 AB → 点「保存默认资源到StreamingAssets」→ **再点一次「创建对比文件」** | `StreamingAssets/` |
| 4. 上传 | 确认服务器已有 `/AB/<平台>/` 目录 → 点「上传AB包和对比文件」 | 服务器 `/AB/<平台>/` |

> 三个平台各打一份、各传一份；资源包分平台，不能混用。

## 五、客户端使用

**检查更新**（启动时调一次）：

```csharp
ABUpdateManager.Instance.CheckUpdate(
    (ok)   => { /* ok=false 时提示玩家检查网络 */ },
    (info) => { /* 进度/提示文字，可接进度条 */ });
```

**加载资源**：

```csharp
ABManager.Instance.LoadResAsync<GameObject>("prefab", "Cube", (obj) => Instantiate(obj));
ABManager.Instance.LoadResAsync<TextAsset>("lua", "luaText.lua", (txt) => print(txt.text));
ABManager.Instance.LoadResAsync<GameObject>("prefab", "Cube", cb, true);   // isSync=true 走同步加载
ABManager.Instance.UnLoad("prefab", (ok) => { });
ABManager.Instance.ClearAB();
```

| API | 说明 |
|---|---|
| `LoadResAsync<T>(包名, 资源名, 回调, isSync = false)` | 泛型加载 |
| `LoadResAsync(包名, Type, 资源名, 回调, isSync = false)` | 按类型加载 |
| `LoadResAsync(包名, 资源名, 回调, isSync = false)` | 按资源名加载 |
| `UnLoad(包名, 回调)` | 卸载单个包（包已在管理器中时**必须传回调**） |
| `ClearAB()` | 卸载全部包 |

> 包名 = 打 AB 时指定的包名（如 `prefab`）；资源名 = 资源在包里的名字（如 `Cube`）。

## 六、注意事项

| 项 | 说明 |
|---|---|
| 先更新、后加载 | 包被加载后文件会被占用，此时热更覆写会失败；必要时先 `ClearAB()` |
| 首包决定首次下载量 | 首包里有的不用下，首包里没有的会在首次启动时下载到 `persistentDataPath` |
| Android 首包 | Android 上首包在 apk 内无法直接加载，需补"把首包复制到 persistentDataPath"这一步（暂未实现） |
| 服务器目录先建好 | FTP 上传不会自动创建目录，缺目录会报 `(550)` |
| 上传没日志 | 说明 `ArtRes/AB/<平台>` 里没有 AB 包，先去打这个平台的包 |
| 平台名要一致 | 工具窗口页签 = 服务器目录名 = 运行时平台 = `ABManager` 里的主包名 |
| 别在场景里挂管理器 | `ABManager` / `ABUpdateManager` 会自动创建，场景里再挂会产生重复实例 |
| 复位到"首次启动" | 删掉 `persistentDataPath` 里的 `ABCompareInfo.txt` 和各 AB 文件 |

> `persistentDataPath`（Windows）：`C:\Users\<用户名>\AppData\LocalLow\<公司名>\<产品名>`，本工程为 `...\DefaultCompany\HotUpdate`；启动时 Console 会打印这个路径。
