# 如何在 Fincept Qt 中新增一个 Screen

本文是 `fincept-qt` 的实战手册，目标是回答一个非常具体的问题:

> 我想给 `Fincept Terminal` 新增一个页面，应该改哪些文件，按什么顺序做，怎样才能让它真正出现在界面上并可调试？

适用目录:

- `d:\open-source\FinceptTerminal\fincept-qt`

建议配合阅读:

- [SOURCE_STRUCTURE_GUIDE.md](file:///d:/open-source/FinceptTerminal/fincept-qt/docs/SOURCE_STRUCTURE_GUIDE.md)

---

## 1. 先记住最小链路

在这个项目里，“新增一个页面”最小要完成 4 件事:

1. 新建 `Screen` 类
2. 把 `.cpp` 加进 `CMakeLists.txt`
3. 在 `WindowFrame.cpp` 注册 `register_factory("screen_id", ...)`
4. 给它一个可触达入口，例如命令栏、导航菜单，或者代码里主动 `navigate`

如果只做第 1 步，页面不会自动出现。

---

## 2. 先决定你要加的是哪种页面

新增页面前，先判断目标属于哪一类。

### 2.1 纯展示 / 设置类页面

特点:

- 主要是静态 UI、配置表单、帮助信息
- 不强依赖实时数据

适合参考:

- `src/screens/about/AboutScreen.*`
- `src/screens/support/SupportScreen.*`

### 2.2 业务页面

特点:

- 页面要调服务、读数据库、拉 HTTP、调 Python

适合参考:

- `src/screens/news/*`
- `src/screens/portfolio/*`
- `src/screens/akshare/*`

### 2.3 共享数据 / 实时类页面

特点:

- 需要跨页面共享数据
- 适合接 `DataHub`

适合参考:

- `src/screens/markets/*`
- `src/screens/watchlist/*`
- `src/screens/news/*`

---

## 3. 推荐的新增流程

推荐顺序如下:

1. 先创建最小 `Screen`
2. 让它能被编译
3. 让它能被 `navigate("your_id")` 打开
4. 再补导航入口
5. 最后再补状态保存、symbol 联动、服务调用

不要一上来就把 UI、服务、数据流、持久化一次全做完。

---

## 4. 最小可运行示例

这里用一个假设页面 `Research Lab` 演示。

约定:

- screen id: `research_lab`
- 类名: `ResearchLabScreen`
- 目录: `src/screens/research_lab/`

### 4.1 新建头文件

文件:

- `src/screens/research_lab/ResearchLabScreen.h`

```cpp
#pragma once

#include <QWidget>

namespace fincept::screens {

class ResearchLabScreen : public QWidget {
    Q_OBJECT
  public:
    explicit ResearchLabScreen(QWidget* parent = nullptr);
};

} // namespace fincept::screens
```

### 4.2 新建实现文件

文件:

- `src/screens/research_lab/ResearchLabScreen.cpp`

```cpp
#include "screens/research_lab/ResearchLabScreen.h"

#include "ui/theme/Theme.h"

#include <QLabel>
#include <QVBoxLayout>

namespace fincept::screens {

ResearchLabScreen::ResearchLabScreen(QWidget* parent) : QWidget(parent) {
    setStyleSheet(QString("background: %1;").arg(ui::colors::BG_BASE()));

    auto* root = new QVBoxLayout(this);
    root->setContentsMargins(24, 24, 24, 24);
    root->setSpacing(12);

    auto* title = new QLabel("Research Lab");
    title->setStyleSheet(QString("color:%1; font-size:18px; font-weight:700;")
                             .arg(ui::colors::TEXT_PRIMARY()));

    auto* subtitle = new QLabel("Custom experimental screen");
    subtitle->setStyleSheet(QString("color:%1; font-size:12px;")
                                .arg(ui::colors::TEXT_SECONDARY()));

    root->addWidget(title);
    root->addWidget(subtitle);
    root->addStretch();
}

} // namespace fincept::screens
```

这一步完成后，类本身已经能编译，但还不能在产品里打开。

---

## 5. 让它进入构建系统

### 5.1 修改 `CMakeLists.txt`

需要把新 `.cpp` 加进 `SCREEN_SOURCES`。

位置参考:

- [CMakeLists.txt](file:///d:/open-source/FinceptTerminal/fincept-qt/CMakeLists.txt)

在 `set(SCREEN_SOURCES ...)` 中加入:

```cmake
src/screens/research_lab/ResearchLabScreen.cpp
```

如果你忘了这一步，常见现象是:

- 头文件能被 IDE 识别
- 但链接或编译时找不到实现
- 或者新页面完全没参与构建

---

## 6. 让路由认识这个页面

### 6.1 在 `WindowFrame.cpp` 引入头文件

文件:

- `src/app/WindowFrame.cpp`

在 include 区增加:

```cpp
#include "screens/research_lab/ResearchLabScreen.h"
```

### 6.2 注册 screen factory

在 `WindowFrame.cpp` 的 `register_factory(...)` 区域加入:

```cpp
dock_router_->register_factory("research_lab", []() {
    return new screens::ResearchLabScreen;
});
```

你可以直接搜:

```cpp
register_factory(
```

来找到同类位置。

### 6.3 这一步的意义

注册以后，这个页面才能被:

- `dock_router_->navigate("research_lab")`
- 导航菜单
- 命令栏
- `EventBus` 导航事件

真正打开。

---

## 7. 给页面一个标题

虽然有时不加也能开，但最好同步把标题映射补上。

文件:

- `src/app/DockScreenRouter.cpp`

在 `title_for_id()` 的哈希表里加入:

```cpp
{"research_lab", "Research Lab"},
```

否则你可能看到:

- tab 标题直接显示原始 id
- 或部分地方显示不友好名称

---

## 8. 给页面加入口

页面注册后，理论上已经可以通过代码导航打开，但用户仍然未必能找到它。

最常见入口有 3 类。

### 8.1 导航菜单入口

文件:

- `src/ui/navigation/ToolBar.cpp`

你可以照着现有写法，在合适分组下加入:

```cpp
nav(tools, "Research Lab", "research_lab");
```

适合:

- 二级页面
- 主 tab 之外的工具页

### 8.2 命令栏搜索入口

文件:

- `src/ui/navigation/CommandBar.cpp`

在命令数据列表里加入一条:

```cpp
{"research_lab",
 "Research Lab",
 "Experimental research workspace",
 {"research", "lab", "rlab"},
 "",
 {"research", "lab", "experiment"}},
```

这样用户就可以在顶部命令栏直接搜索 `research_lab` 或别名。

### 8.3 代码主动跳转

如果某个按钮点击后要打开页面，可以调用:

```cpp
dock_router_->navigate("research_lab");
```

或者通过事件:

```cpp
EventBus::instance().publish("nav.switch_screen", {
    {"screen_id", "research_lab"}
});
```

适合:

- 页面间跳转
- 菜单动作
- 向导流程

---

## 9. 什么时候需要改主导航 Tab

大多数新页面不需要进顶部主 Tab。

这个项目的很多页面其实都属于:

- 已注册 screen
- 可通过菜单 / 命令栏进入
- 但不是一级 tab

所以更推荐的顺序是:

1. 先做成可通过 `navigate` 打开的页面
2. 再决定是否值得放进一级导航

除非这个页面是核心功能，否则不要一开始就往主导航塞。

---

## 10. 如果页面要保存状态

如果你希望页面记住:

- 上次选中的 tab
- 上次搜索词
- 上次筛选条件
- 当前选中的 symbol / region

建议实现 `IStatefulScreen`。

文件:

- `src/screens/IStatefulScreen.h`

### 10.1 典型写法

```cpp
class ResearchLabScreen : public QWidget, public IStatefulScreen {
    Q_OBJECT
  public:
    explicit ResearchLabScreen(QWidget* parent = nullptr);

    void restore_state(const QVariantMap& state) override;
    QVariantMap save_state() const override;
    QString state_key() const override { return "research_lab"; }
    int state_version() const override { return 1; }
};
```

### 10.2 最小实现示例

```cpp
void ResearchLabScreen::restore_state(const QVariantMap& state) {
    if (search_edit_) {
        search_edit_->setText(state.value("search").toString());
    }
}

QVariantMap ResearchLabScreen::save_state() const {
    QVariantMap state;
    if (search_edit_) {
        state["search"] = search_edit_->text();
    }
    return state;
}
```

### 10.3 什么时候值得做

值得做:

- 用户会反复使用的工作页面
- 有明显过滤 / 选择状态的页面

没必要做:

- 静态说明页
- 简单 about / support / legal 页面

---

## 11. 如果页面要支持 symbol group 联动

如果页面需要和别的行情页面联动 symbol，比如:

- 切换组 A 的 symbol 时，当前页面也跟着切

要看:

- `src/core/symbol/IGroupLinked.h`
- `src/app/DockScreenRouter.cpp`

这属于进阶能力，不是新增页面的必做项。

适合的页面:

- `NewsScreen`
- `WatchlistScreen`
- `PortfolioScreen`
- `EquityTradingScreen`

不适合的页面:

- About
- Docs
- Settings

---

## 12. 如果页面要有业务逻辑

不要把所有业务都写进 `Screen.cpp`。

推荐模式:

```text
Screen -> Service -> Data source / Repository / PythonRunner
```

举例:

- 静态 UI 逻辑写在 `Screen`
- 数据抓取、缓存、解析写在 `Service`
- 数据库存取写在 `Repository`
- Python 数据提供者走 `PythonRunner`

### 12.1 什么时候可以先不建 Service

可以先不建:

- 纯说明页面
- 非复用的小型工具页

建议尽快建 Service:

- 页面要拉数据
- 后续可能被 MCP、AI、别的页面复用
- 逻辑超过一个文件就开始膨胀

---

## 13. 新页面用不用接 `DataHub`

判断标准很简单。

### 13.1 不需要 `DataHub`

适合:

- 一次性读取
- 单页面使用
- 配置 / 表单 / 文档类页面

### 13.2 建议接 `DataHub`

适合:

- 实时行情
- 多页面共享同一份数据
- 需要缓存与统一刷新策略
- 需要订阅型更新

如果你不确定，先不接，先把页面做出来。

---

## 14. 一个完整的最小修改清单

假设新增 `research_lab`，你通常要改这些文件:

- 新增 `src/screens/research_lab/ResearchLabScreen.h`
- 新增 `src/screens/research_lab/ResearchLabScreen.cpp`
- 修改 `CMakeLists.txt`
- 修改 `src/app/WindowFrame.cpp`
- 修改 `src/app/DockScreenRouter.cpp`

如果还要入口，再加:

- 修改 `src/ui/navigation/ToolBar.cpp`
- 修改 `src/ui/navigation/CommandBar.cpp`

如果还要业务:

- 新增 `src/services/research_lab/ResearchLabService.h`
- 新增 `src/services/research_lab/ResearchLabService.cpp`
- 修改 `CMakeLists.txt`

如果还要状态保存:

- 让 screen 实现 `IStatefulScreen`

---

## 15. 推荐的开发顺序

### 第 1 步

只做空页面和注册，让它能显示 “Hello Screen”。

### 第 2 步

补菜单 / 命令栏入口，让你不用每次改代码触发导航。

### 第 3 步

再补 UI 结构和布局。

### 第 4 步

最后再接 Service、Repository、PythonRunner 或 DataHub。

这个顺序最稳，因为:

- 能快速验证路由是否通
- 能快速判断问题在页面还是在数据层

---

## 16. 最常见的失败原因

### 16.1 页面写了，但打不开

优先检查:

1. `.cpp` 是否加入 `CMakeLists.txt`
2. `WindowFrame.cpp` 是否注册了 `register_factory`
3. screen id 是否前后一致

### 16.2 页面能打开，但标题不对

检查:

- `DockScreenRouter.cpp` 的 `title_for_id()`

### 16.3 页面能通过代码打开，但用户找不到

检查:

- `ToolBar.cpp`
- `CommandBar.cpp`
- 是否有按钮或事件调用 `navigate`

### 16.4 页面状态不保存

检查:

- 是否实现 `IStatefulScreen`
- `state_key()` 是否和页面 id 对应

### 16.5 新页面编译不过

优先检查:

- include 路径
- `CMakeLists.txt`
- 命名空间是否是 `fincept::screens`

---

## 17. 建议照着抄的参考页面

### 17.1 最简单参考

- `src/screens/about/AboutScreen.*`

适合学习:

- 最基础 QWidget 页面怎么写
- 怎么组织样式和布局

### 17.2 带状态的参考

- `src/screens/notes/NotesScreen.*`

适合学习:

- `IStatefulScreen`
- 页面内部结构拆分
- 事件订阅和持久化状态

### 17.3 带 Python 的参考

- `src/screens/akshare/*`
- `src/services/akshare/*`

适合学习:

- screen 如何接 Python 驱动服务

### 17.4 带复杂业务的参考

- `src/screens/news/*`
- `src/screens/portfolio/*`

适合学习:

- 页面拆分
- 业务和 UI 的边界

---

## 18. 最短版本清单

如果你只想记住新增 screen 的最短步骤，记这 5 条就够:

1. 建 `src/screens/<name>/<Name>Screen.h/.cpp`
2. 把 `.cpp` 加进 `CMakeLists.txt` 的 `SCREEN_SOURCES`
3. 在 `WindowFrame.cpp` 注册 `register_factory("<id>", ...)`
4. 在 `DockScreenRouter.cpp` 补标题映射
5. 在 `ToolBar.cpp` 或 `CommandBar.cpp` 加入口

---

## 19. 后续建议

当你已经能稳定加页面后，下一步最值得整理的是:

- `HOW_TO_ADD_A_SERVICE.md`
- `DEBUGGING_CHECKLIST.md`

因为真正的开发成本，通常不是“页面显示出来”，而是:

- 数据接不进来
- 页面状态不一致
- 路由和导航分散
- Python / DataHub / Repository 三套机制混在一起

把这两份也补上后，整个项目的二次开发门槛会明显下降。
