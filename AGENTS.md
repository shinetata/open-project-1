# 工程工作说明（面向协作/自动化助手）

本仓库是 Unity Open Project #1「Chop Chop」的完整 Unity 工程。主要可编辑内容位于 `UOP1_Project/Assets`，其中脚本集中在 `UOP1_Project/Assets/Scripts`。

## 1. 工程概览

- **引擎版本**：`Unity 2020.3.17f1`（见 `UOP1_Project/ProjectSettings/ProjectVersion.txt`），建议通过 Unity Hub 安装同版本后打开。
- **项目类型**：第三人称动作/冒险 Demo（已停止持续开发，README 保留为历史信息）。
- **关键依赖**（见 `UOP1_Project/Packages/manifest.json`）：
  - Addressables（资源/场景异步加载）
  - Input System（输入系统）
  - Localization（本地化）
  - URP、Cinemachine 等

## 2. 目录结构（高频）

- `UOP1_Project/`：Unity 项目根目录（用 Unity 打开这一层）
  - `Assets/`：所有资源与代码
    - `Assets/Scenes/`：场景
      - `Initialization.unity`：**唯一**被加入 Build Settings 的场景（入口场景）
      - `Locations/`：关卡/地点场景
      - `Managers/`：管理器相关场景（持久化管理器）
      - `Menus/`：菜单场景
    - `Assets/Scripts/`：C# 脚本（主要逻辑入口）
      - `SceneManagement/`：场景加载/切换（基于 Addressables + 事件通道）
      - `Events/`：ScriptableObject 事件通道（Event Channels）与监听器
      - `Input/`：Input System 封装（`InputReader`）
      - `SaveSystem/`：存档/读取（通过 Addressables GUID 还原物品/地点）
      - `StateMachine/`：状态机系统（独立 asmdef：`UOP1.StateMachine`）
    - `Assets/AddressableAssetsData/`：Addressables 设置与分组（尽量用 Unity 的 Addressables 窗口维护）
    - `Assets/LocalizationFiles/`：本地化配置与表格资源
  - `Packages/`：UPM 包清单与锁文件
  - `ProjectSettings/`：Unity 项目设置（谨慎修改）
  - `Library/`、`Temp/`、`Logs/`：Unity 生成目录（不要依赖其内容进行逻辑修改；如需提交变更请非常谨慎）

## 3. 核心运行链路（场景/加载架构）

工程采用“**入口场景 + 持久化管理器场景 + 目标场景**”的模式，并通过 ScriptableObject 事件通道解耦系统：

1. **Build Settings 入口**：`Assets/Scenes/Initialization.unity`
2. `InitializationLoader`（`UOP1_Project/Assets/Scripts/SceneManagement/InitializationLoader.cs`）在启动时：
   - 以 Additive 方式加载 **Persistent Managers** 场景（`GameSceneSO.sceneReference.LoadSceneAsync`）
   - 通过 Addressables 加载一个 `LoadEventChannelSO`，并广播“加载主菜单”的请求
   - 随后卸载 `Initialization` 场景（索引 0）
3. `SceneLoader`（`UOP1_Project/Assets/Scripts/SceneManagement/SceneLoader.cs`）监听 `LoadEventChannelSO`：
   - 统一处理菜单/地点场景的加载与卸载（淡入淡出、加载界面、设置 ActiveScene 等）
   - 玩法管理器（Gameplay managers）场景按需常驻，菜单与地点场景按请求切换
4. **编辑器冷启动**：`EditorColdStartup`（`UOP1_Project/Assets/Scripts/EditorTools/MonoBehaviours/EditorColdStartup.cs`）
   - 允许在编辑器中直接从某个 Location 场景按 Play（不经过 Initialization）也能补齐必要的持久化管理器

### 场景数据抽象

- `GameSceneSO`（`UOP1_Project/Assets/Scripts/SceneManagement/ScriptableObjects/GameSceneSO.cs`）
  - 描述“可加载的场景”与其 `AssetReference`（Addressables）
  - `sceneType` 用于区分 Location/Menu/Managers 等
- `LocationSO`（`UOP1_Project/Assets/Scripts/SceneManagement/ScriptableObjects/LocationSO.cs`）
  - Location 额外包含 `LocalizedString` 的地点名称

## 4. 事件通道（Event Channels）约定

大量系统通过 ScriptableObject 通道通信，常见于 `UOP1_Project/Assets/Scripts/Events/ScriptableObjects/`：

- `LoadEventChannelSO`：场景加载请求（`GameSceneSO + showLoadingScreen + fadeScreen`）
- `VoidEventChannelSO`、`BoolEventChannelSO`、`IntEventChannelSO` 等：无参/基础类型事件
- UI 相关通道位于 `.../ScriptableObjects/UI/`

建议新增系统间通信时优先复用/扩展通道模式，避免直接硬引用或到处拉单例。

## 5. 开发与验证（本地）

- **打开工程**：Unity Hub → Open → 选择 `UOP1_Project/`
- **入口场景**：通常直接打开并运行 `Assets/Scenes/Initialization.unity`
- **从任意地点场景调试**：若场景里挂了 `EditorColdStartup`，可直接 Play（仅编辑器）

## 6. 代码风格与自动格式化

- 代码风格由根目录 `.editorconfig` 约束；`*.cs` **使用 Tab 缩进**。
- 仓库包含 GitHub Actions：`.github/workflows/linter.yml` 会对 `UOP1_Project/Assets/Scripts` 执行 `dotnet-format` 并自动提交格式化结果。
  - 本地如需对齐 CI，可在 `UOP1_Project/` 下执行：`dotnet-format -f Assets/Scripts -v d`

## 7. Unity 资源协作注意事项

- **不要手改 `.meta`**；移动/重命名资源请在 Unity 编辑器中操作，确保 GUID 稳定。
- 场景/Prefab 等 YAML 资产易产生大量 diff：尽量避免无关的重新保存与批量重序列化。
- Addressables/Localization 等配置尽量通过对应窗口维护，避免直接编辑生成文件导致不一致。
