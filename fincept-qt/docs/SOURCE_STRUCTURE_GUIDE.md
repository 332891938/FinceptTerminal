# Fincept Qt 源码结构与开发指南

本文面向需要在 `fincept-qt` 中调试、修改和新增功能的开发者。

目标不是简单罗列目录，而是帮你建立一套可工作的心智模型:

- 程序从哪里启动
- 界面页面是如何注册和打开的
- 数据如何从 Python / HTTP / WebSocket 进入 UI
- 数据保存在哪里
- 新增一个页面、服务、数据源时要改哪些地方

---

## 1. 先建立整体心智模型

可以把 `fincept-qt` 理解成 8 层:

1. `app/`
   - 应用启动、多窗口、页面路由、主窗口框架
2. `screens/`
   - 各个功能界面，直接面向用户
3. `services/`
   - 业务逻辑和数据拉取逻辑
4. `datahub/`
   - 统一的数据发布 / 订阅层，适合实时数据和共享缓存
5. `python/`
   - C++ 调 Python 脚本的桥接层
6. `storage/`
   - SQLite、Repository、SecureStorage、工作区持久化
7. `core/`
   - 全局基础设施，像事件总线、配置、日志、快捷键、布局、符号联动
8. `ui/`
   - 通用 UI 组件、导航、命令栏、主题、工作区对话框

一句话概括:

`WindowFrame` 负责装配 UI，`screens` 负责界面，`services` 负责业务，`DataHub` 和 `storage` 负责共享数据与持久化，`python` 负责把脚本生态接进来。

---

## 2. 顶层目录怎么读

以 `d:\open-source\FinceptTerminal\fincept-qt` 为根目录，最重要的目录如下:

```text
fincept-qt/
├─ CMakeLists.txt
├─ CMakePresets.json
├─ DATAHUB_ARCHITECTURE.md
├─ docs/
├─ resources/
├─ scripts/
├─ src/
├─ packaging/
└─ cmake/
```

各目录职责:

- `CMakeLists.txt`
  - 总构建入口，新增 `.cpp` 文件通常要在这里加入对应 source list
- `CMakePresets.json`
  - 常用构建预设，Windows 下优先看 `win-debug`、`win-fast`、`win-release`
- `resources/`
  - 图标、HTML、组件目录、Python requirements 等运行资源
- `scripts/`
  - 大量 Python 数据脚本，AkShare、FRED、News、Exchange、AI Quant 等都在这里
- `src/`
  - 主体 C++ 源码
- `docs/`
  - 项目内专项文档，不是完整源码地图，但对局部模块有帮助
- `packaging/`
  - 打包相关
- `cmake/`
  - 一些辅助 CMake 脚本

---

## 3. 启动链路: 程序从哪里开始

最值得先读的 3 个文件:

- `src/app/main.cpp`
- `src/app/WindowFrame.cpp`
- `src/app/DockScreenRouter.cpp`

启动过程可以简化为:

```text
main.cpp
  -> 初始化 Profile / CrashHandler / 单实例控制
  -> 初始化 TerminalShell
  -> 注册 DataHub 元类型和核心 Producer
  -> 初始化数据库、配置、服务
  -> 创建 WindowFrame
  -> WindowFrame 注册所有 screen factory
  -> 通过 router 打开默认页面
```

### 3.1 `main.cpp` 做了什么

`src/app/main.cpp` 是总入口，关键职责有:

- 解析 `--profile`
- 初始化 `AppPaths`
- 安装崩溃处理器
- 建立单实例机制 `SingleApplication`
- 初始化 `TerminalShell`
- 注册 `DataHub` 元类型
- 注册多个核心服务到 `DataHub`
- 初始化数据库和迁移
- 创建主窗口 `WindowFrame`

如果你遇到以下问题，先从 `main.cpp` 下断点:

- 程序启动闪退
- 页面完全没起来
- 服务没有注册到 Hub
- Python 环境没有初始化
- 数据库迁移没有执行

---

## 4. 主窗口和页面系统

### 4.1 `WindowFrame` 是真正的 UI 装配中心

`src/app/WindowFrame.cpp` 是最重要的文件之一。

它负责:

- 创建主窗口框架
- 组装工具栏、导航栏、状态栏、命令栏
- 注册 screen factory
- 接入认证、锁屏、快捷键、工作区恢复
- 把具体 screen 放进 dock 系统

如果你想回答“某个页面为什么能打开”，大概率都要看这里。

### 4.2 页面不是写完就自动出现

页面要真正可用，通常至少要经过两步:

1. 有 screen 类
2. 在 `WindowFrame.cpp` 中注册 factory

例如页面注册集中在 `WindowFrame.cpp` 里，像:

- `dashboard`
- `markets`
- `news`
- `portfolio`
- `akshare`
- `data_sources`
- `docs`

都通过 `dock_router_->register_factory("screen_id", []() { ... });` 接进来。

这意味着:

- 你新增了一个 `Screen` 类，不注册就不会被路由打开
- 你能搜 `register_factory(` 找到整个产品支持的页面列表

### 4.3 `DockScreenRouter` 管页面标题、实例和符号联动

`src/app/DockScreenRouter.cpp` 负责:

- screen id 到标题的映射
- 懒加载创建页面
- 处理 tab / dock 生命周期
- 对支持 `IGroupLinked` 的页面做 symbol group 联动

如果某页面:

- 标题不对
- 路由能打开但标签名不对
- 分组联动失效

先看这里。

---

## 5. `src/` 目录应该怎么理解

### 5.1 建议先把 `src/` 当成这几块

```text
src/
├─ app/
├─ core/
├─ datahub/
├─ python/
├─ network/
├─ storage/
├─ services/
├─ screens/
├─ ui/
├─ mcp/
└─ ai_chat/
```

### 5.2 `core/` 是基础设施层

常改的子目录:

- `core/config/`
  - `AppPaths`、`AppConfig`、`ProfileManager`
- `core/events/`
  - `EventBus`，偏命令和事件广播
- `core/logging/`
  - 日志
- `core/session/`
  - 页面状态、窗口几何、会话恢复
- `core/layout/`
  - 工作区和布局目录
- `core/symbol/`
  - 跨页面 symbol / group 联动
- `core/actions/`
  - 全局动作和动作注册
- `core/window/`
  - 多窗口注册管理

这里的判断标准:

- 如果问题是“应用级规则”，先看 `core`
- 如果问题是“某个业务怎么拉数据”，先看 `services`

### 5.3 `screens/` 是界面层

`screens/` 下面基本是一屏一个目录，常见模式是:

```text
src/screens/news/
  NewsScreen.cpp
  NewsScreen.h
  NewsFeedPanel.cpp
  NewsFeedModel.cpp
  NewsDetailPanel.cpp
```

通常:

- `XXXScreen` 是页面根容器
- `Panel` / `Widget` / `Tab` 是页面子块
- 较复杂页面会拆出 model、delegate、toolbar、dialog

如果你是改 UI、改交互、改按钮行为，通常从 `screens/` 入手。

### 5.4 `services/` 是业务层

`services/` 很大，几乎每个业务域都有一套服务:

- `markets/`
- `news/`
- `economics/`
- `akshare/`
- `portfolio/`
- `forum/`
- `wallet/`
- `prediction/`
- `workspace/`
- `workflow/`

典型职责:

- 发 HTTP / WebSocket 请求
- 调 Python 脚本
- 组织缓存
- 对 UI 暴露 callback、signal 或 DataHub producer

你可以把 `services` 理解成“可复用的后端逻辑层”。

### 5.5 `storage/` 是持久化层

最重要的几块:

- `storage/sqlite/`
  - SQLite 封装和 migration
- `storage/repositories/`
  - Repository 模式，面向业务对象读写数据库
- `storage/secure/`
  - SecureStorage，存 API Key 等敏感信息
- `storage/workspace/`
  - 工作区快照、恢复、workspace db
- `storage/cache/`
  - 一些缓存和会话类存储

如果你改的是:

- 设置项
- watchlist
- notes
- portfolio
- data source 连接
- report

大概率要落到 repository。

### 5.6 `python/` 是 C++ 和脚本生态之间的桥

关键文件:

- `src/python/PythonRunner.cpp`
- `src/python/PythonSetupManager.cpp`
- `src/python/PythonWorker.cpp`

职责:

- 找到 Python 解释器或虚拟环境
- 组装环境变量
- 注入 SecureStorage 里的 API Key
- 运行 `scripts/` 下的脚本
- 处理异步执行和输出

凡是“这个功能背后其实跑了 Python”，都要看这里。

---

## 6. 三种最常见的数据流

理解源码时，最有帮助的是先判断这个功能属于哪一种数据流。

### 6.1 纯 C++ 服务型

路径通常是:

```text
Screen -> Service -> HttpClient / Repository -> UI 更新
```

适合:

- 设置
- 文件管理
- 一部分论坛、工作区、配置类功能

### 6.2 Python 脚本驱动型

路径通常是:

```text
Screen -> Service -> PythonRunner -> scripts/*.py -> JSON -> Service -> UI
```

典型模块:

- `AkShare`
- `Economics`
- `Markets` 的部分数据
- 各类宏观和政府数据源

这是本项目非常重要的一条数据接入路线。

### 6.3 DataHub 推送型

路径通常是:

```text
Producer Service -> DataHub topic -> 多个 Screen/Widget 订阅
```

适合:

- 实时行情
- 共享 market quote
- WebSocket 推送
- 多页面复用同一份数据

如果你想减少重复刷新、重复脚本执行、重复网络请求，应优先考虑走 `DataHub`。

---

## 7. `EventBus` 和 `DataHub` 的区别

这个项目同时有 `EventBus` 和 `DataHub`，要分清楚。

### 7.1 `EventBus`

文件:

- `src/core/events/EventBus.h`

用途:

- 应用级事件广播
- 更像“通知别人发生了一件事”

适合:

- 页面导航事件
- 某个动作触发
- 状态变化通知

### 7.2 `DataHub`

文件:

- `src/datahub/DataHub.h`
- `DATAHUB_ARCHITECTURE.md`

用途:

- 数据状态发布 / 订阅
- 持有 last-known-good value
- 管 producer、topic policy、缓存有效期、刷新节流

适合:

- `market:quote:AAPL`
- `news:symbol:NVDA`
- `econ:fred:GDP`
- `ws:kraken:BTC-USD`

简单判断:

- “发生了什么” -> `EventBus`
- “现在这份数据是什么” -> `DataHub`

---

## 8. Python 脚本生态怎么接进来的

`scripts/` 目录很大，实际上是产品能力的重要组成部分。

常见脚本族:

- `akshare_*.py`
- `fred_data.py`
- `tradingview_data.py`
- `polygon_io_data.py`
- `exchange/*.py`
- `agents/*.py`
- `ai_quant_lab/*.py`

### 8.1 为什么功能很多但 C++ 服务不算复杂

因为很多页面是“C++ 壳 + Python 数据脚本”模式。

例如 AkShare:

```text
AkShareScreen
  -> AkShareService
  -> PythonRunner
  -> scripts/akshare_xxx.py
  -> JSON 返回
  -> 表格 / 面板渲染
```

### 8.2 Python 环境的关键点

`PythonRunner` 会做这些事:

- 寻找 `venv-numpy1`、`venv-numpy2`、`.venv`
- 注入 `PYTHONPATH`
- 从 `SecureStorage` 取 API Key 并注入环境变量
- 根据脚本类型选择不同 venv

所以调试 Python 类问题时，先确认:

1. `scripts/` 是否被正确定位
2. Python 是否可用
3. 所需 API Key 是否已写入 `SecureStorage`
4. 脚本标准输出是否是预期 JSON

---

## 9. 存储和数据库怎么走

### 9.1 SQLite 是主存储

关键文件:

- `src/storage/sqlite/Database.h`
- `src/storage/sqlite/migrations/MigrationRunner.h`

特点:

- 使用 SQLite
- 有版本化 migration
- 应用启动时会执行迁移

常见数据都落在 repository:

- `WatchlistRepository`
- `PortfolioRepository`
- `NotesRepository`
- `SettingsRepository`
- `DataSourceRepository`
- `ReportRepository`

### 9.2 `SecureStorage` 存敏感凭证

关键文件:

- `src/storage/secure/SecureStorage.h`

平台差异:

- Windows: Credential Manager
- macOS: Keychain
- Linux: 当前注释里明确写了是临时性弱保护方案

所以 API Key 相关问题要分两类看:

- 配置是否写入了 `SecureStorage`
- 运行脚本时是否被 `PythonRunner` 注入到环境变量

### 9.3 `AppPaths` 决定本地数据在哪里

关键文件:

- `src/core/config/AppPaths.h`

你需要知道这些目录:

- `data/`
- `logs/`
- `cache/`
- `runtime/`
- `workspaces/`
- `crashdumps/`

调试时很有用:

- 看 SQLite 文件
- 看日志
- 看 Python runtime
- 看 crash dump

---

## 10. Build 系统要怎么理解

### 10.1 入口

关键文件:

- `CMakeLists.txt`
- `CMakePresets.json`

### 10.2 你最需要知道的 3 件事

1. 这个项目用 `CMake + Ninja + Qt 6.8.3`
2. 很多源文件通过大列表维护，比如 `SCREEN_SOURCES`
3. 你新增 `.cpp` 后，往往需要手动加入对应 source list

### 10.3 常用 preset

- `win-debug`
  - 最适合下断点
- `win-fast`
  - 日常快速迭代
- `win-release`
  - 正常 Release 构建

### 10.4 新增文件为什么会“代码写了但不编译”

因为项目不是纯粹靠 glob 自动抓取源码，而是在 `CMakeLists.txt` 里按模块维护 source list。

最常见漏改点:

- 新增 `Screen.cpp` 但没进 `SCREEN_SOURCES`
- 新增 `Service.cpp` 但没进 `SERVICE_SOURCES`
- 新增 `ui/*.cpp` 但没进 `UI_SOURCES`

---

## 11. 常见开发任务从哪里入手

### 11.1 修改现有页面 UI

优先顺序:

1. 找 `src/screens/<module>/`
2. 找根页面 `XXXScreen.cpp`
3. 看它依赖哪些 panel / service
4. 如果数据不对，再追到 `services/`

例子:

- 改新闻页 -> `src/screens/news/`
- 改组合页 -> `src/screens/portfolio/`
- 改 AkShare -> `src/screens/akshare/` + `src/services/akshare/`

### 11.2 新增一个页面

最小步骤:

1. 新建 `src/screens/<module>/MyScreen.h/.cpp`
2. 在 `CMakeLists.txt` 把 `.cpp` 加入 `SCREEN_SOURCES`
3. 在 `WindowFrame.cpp` 注册 `register_factory("my_screen", ...)`
4. 如需导航入口，再改:
   - `src/ui/navigation/ToolBar.cpp`
   - `src/ui/navigation/CommandBar.cpp`
   - 或相关导航栏实现
5. 如需标题映射，补 `DockScreenRouter.cpp`

如果不做第 3 步，页面通常不会出现在可导航体系里。

### 11.3 新增一个业务服务

推荐步骤:

1. 新建 `src/services/<domain>/MyService.h/.cpp`
2. 加入 `CMakeLists.txt` 的 `SERVICE_SOURCES`
3. 如果需要给多个页面共享数据，考虑实现 `DataHub::Producer`
4. 如果只是单页面简单调用，也可以先暴露 callback / signal

建议:

- 单次拉取、单页面消费 -> callback / signal 即可
- 多页面复用、实时更新、需要共享缓存 -> 走 `DataHub`

### 11.4 新增一个 Python 驱动的数据功能

推荐步骤:

1. 在 `scripts/` 加 Python 脚本
2. 在 `src/services/<domain>/` 加一个 service 包装
3. 通过 `PythonRunner` 调脚本
4. 在 `screens/` 做 UI
5. 如果是可复用数据，再决定是否发布到 `DataHub`

适合照着现有模块抄模式的例子:

- `AkShareService`
- `EconomicsService`
- `MarketDataService`

### 11.5 新增一个 Data Source 连接器

如果你要接入 `Data Sources` 页面，重点看:

- `src/screens/data_sources/ConnectorRegistry.h`
- `src/screens/data_sources/DataSourceTypes.h`
- `src/screens/data_sources/DataSourcesScreen.cpp`
- `src/storage/repositories/DataSourceRepository.*`

这个体系和“独立功能页”不是一回事。

有些能力虽然已经有页面，比如 AkShare，但未必已经进入 `Data Sources` 连接器注册体系。

### 11.6 新增一个工作区可保存页面

重点看:

- `src/services/workspace/WorkspaceManager.h`
- `src/screens/IStatefulScreen.h`
- `src/core/session/ScreenStateManager.*`

如果页面状态希望被保存 / 恢复，通常要实现相应的 stateful 接口。

---

## 12. 调试时最有价值的入口文件

如果你第一次接手这套代码，我建议按下面顺序读:

1. `src/app/main.cpp`
2. `src/app/WindowFrame.cpp`
3. `src/app/DockScreenRouter.cpp`
4. `src/datahub/DataHub.h`
5. `src/python/PythonRunner.cpp`
6. `src/storage/sqlite/Database.h`
7. `src/storage/sqlite/migrations/MigrationRunner.h`
8. 你当前要修改的那个模块目录:
   - `src/screens/...`
   - `src/services/...`

### 12.1 如果你在查页面入口

搜这些关键词:

- `register_factory(`
- `ToolBar`
- `CommandBar`
- `navigate(`

### 12.2 如果你在查数据来源

搜这些关键词:

- `PythonRunner::instance()`
- `DataHub::instance()`
- `fetch_`
- `execute(`
- `subscribe(`

### 12.3 如果你在查持久化

搜这些关键词:

- `Repository::instance()`
- `save(`
- `list_all(`
- `Database::instance()`

---

## 13. 推荐的断点位置

### 13.1 启动失败

- `main.cpp`
- `WindowFrame` 构造函数
- 数据库打开和 migration 位置

### 13.2 页面打不开

- `WindowFrame.cpp` 中的 `register_factory`
- `DockScreenRouter::navigate` 相关逻辑
- 导航栏触发的 action

### 13.3 页面打开了但没数据

优先检查:

1. screen 是否调用 service
2. service 是否调用 Python / HTTP / WebSocket
3. Python 脚本是否真的返回了数据
4. DataHub topic 是否有发布
5. UI 是否订阅或接收到了结果

### 13.4 设置了 API Key 仍然不生效

依次检查:

1. `SettingsScreen` 是否保存成功
2. `SecureStorage` 是否成功读取
3. `PythonRunner` 是否把 key 注入环境变量
4. Python 脚本读取的 env 名是否一致

---

## 14. 对目录做一个“开发用途”分类

### 14.1 日常最常改

- `src/screens/`
- `src/services/`
- `src/ui/`
- `scripts/`

### 14.2 改架构或全局行为时常改

- `src/app/`
- `src/core/`
- `src/datahub/`
- `src/storage/`

### 14.3 只在特定需求下改

- `src/mcp/`
- `packaging/`
- `resources/wallet/`
- `cmake/`

---

## 15. 新人最容易踩的坑

- 写了新页面，但忘了在 `WindowFrame.cpp` 里注册 factory
- 写了新 `.cpp`，但没加入 `CMakeLists.txt`
- 只改了 screen，没有追到真正的数据 service
- 以为 `EventBus` 和 `DataHub` 是同一个东西
- 以为 `Data Sources` 页面会自动识别所有数据功能
- 只看 C++，没看 `scripts/`，结果误判“功能没实现”
- API Key 已保存，但脚本环境里没拿到
- 改了 repository 或 migration，没有验证老数据升级路径

---

## 16. 建议的阅读顺序

如果你的目标是“能稳定改功能”，建议这样读:

### 第 1 轮: 建立框架感

- `main.cpp`
- `WindowFrame.cpp`
- `DockScreenRouter.cpp`
- `CMakeLists.txt`

### 第 2 轮: 建立数据流概念

- `DataHub.h`
- `PythonRunner.cpp`
- `Database.h`
- `MigrationRunner.h`

### 第 3 轮: 精读你要改的模块

例如你要改:

- AkShare -> `screens/akshare` + `services/akshare` + `scripts/akshare_*.py`
- News -> `screens/news` + `services/news`
- Portfolio -> `screens/portfolio` + `services/portfolio` + repositories
- Data Sources -> `screens/data_sources` + `DataSourceRepository`

---

## 17. 快速定位表

| 需求 | 先看哪里 |
|---|---|
| 程序为什么启动失败 | `src/app/main.cpp` |
| 页面为什么没出现 | `src/app/WindowFrame.cpp` |
| 页面标题和路由 | `src/app/DockScreenRouter.cpp` |
| 全局事件广播 | `src/core/events/EventBus.*` |
| 数据共享与订阅 | `src/datahub/*` |
| Python 脚本执行 | `src/python/*` |
| API Key 存储 | `src/storage/secure/*` |
| SQLite 和迁移 | `src/storage/sqlite/*` |
| 页面状态保存 | `src/core/session/*`、`src/services/workspace/*` |
| Data Sources 连接器 | `src/screens/data_sources/*` |
| AkShare 功能 | `src/screens/akshare/*`、`src/services/akshare/*`、`scripts/akshare_*.py` |

---

## 18. 对当前项目的一句话判断

这不是一个“纯 Qt 界面项目”，而是一个:

`Qt 多页面工作台 + 大量 Python 数据脚本 + SQLite 持久化 + DataHub 实时数据层`

如果你只盯着某一层，容易看不清全貌。

最稳妥的做法是:

- 先确认问题属于哪条数据流
- 再追 `screen -> service -> bridge -> storage/datahub -> UI`

---

## 19. 后续可继续补充的文档

如果你准备长期维护这个项目，建议后续继续拆出这些专题文档:

- `HOW_TO_ADD_A_SCREEN.md`
- `HOW_TO_ADD_A_SERVICE.md`
- `HOW_TO_ADD_A_PYTHON_PROVIDER.md`
- `HOW_TO_ADD_A_DATASOURCE_CONNECTOR.md`
- `DEBUGGING_CHECKLIST.md`

这样后续新增功能时，团队不需要每次重新摸索。
