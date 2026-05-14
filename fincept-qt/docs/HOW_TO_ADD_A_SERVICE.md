# 如何在 Fincept Qt 中新增一个 Service

本文面向 `fincept-qt` 的二次开发者，解决的问题是:

> 我想新增一个业务服务，让页面、数据源、脚本、数据库或 DataHub 能通过统一方式接进项目，应该怎么设计和落地？

建议配合阅读:

- [SOURCE_STRUCTURE_GUIDE.md](file:///d:/open-source/FinceptTerminal/fincept-qt/docs/SOURCE_STRUCTURE_GUIDE.md)
- [HOW_TO_ADD_A_SCREEN.md](file:///d:/open-source/FinceptTerminal/fincept-qt/docs/HOW_TO_ADD_A_SCREEN.md)
- [ARCHITECTURE.md](file:///d:/open-source/FinceptTerminal/docs/ARCHITECTURE.md)
- [CPP_CONTRIBUTOR_GUIDE.md](file:///d:/open-source/FinceptTerminal/docs/CPP_CONTRIBUTOR_GUIDE.md)
- [PYTHON_CONTRIBUTOR_GUIDE.md](file:///d:/open-source/FinceptTerminal/docs/PYTHON_CONTRIBUTOR_GUIDE.md)
- [datahub-guide.md](file:///d:/open-source/FinceptTerminal/fincept-qt/docs/agents/datahub-guide.md)

---

## 1. 先记住一句话

在这个项目里:

- `Screen` 负责 UI
- `Service` 负责业务逻辑、数据拉取、缓存协调、对外信号
- `Repository` 负责持久化
- `PythonRunner` 负责脚本桥接
- `DataHub` 负责共享数据流

所以新增 service 时，最重要的不是“能不能跑”，而是“它应该站在哪一层”。

---

## 2. 什么情况下应该新建 Service

推荐新建 service 的场景:

- 页面需要拉取远程数据
- 页面需要调 Python 脚本
- 多个页面会复用同一套业务逻辑
- 需要对 Repository 做统一封装
- 需要通过信号通知多个 UI
- 需要接入 DataHub 统一共享数据

不建议新建 service 的场景:

- 只是一个纯静态说明页
- 页面内只有非常轻微、不可复用的本地 UI 逻辑

简单判断:

- “UI 之外的逻辑”开始变重时，就应该抽出 service

---

## 3. 先判断你要加的是哪种 Service

这个项目里的 service 不是一种形态，而是 4 种常见形态。

### 3.1 纯本地业务 / CRUD Service

特点:

- 主要读写 Repository
- 基本不碰远程 API
- 偏工作流、配置、文档、文件管理

适合参考:

- `PortfolioService`
- `WorkflowService`
- `FileManagerService`

### 3.2 HTTP / WebSocket Service

特点:

- 通过 `QNetworkAccessManager`、`HttpClient` 或 WebSocket 拉数据
- 解析 JSON 后通过 signal 发给 UI

适合参考:

- `NewsService`
- `PolymarketService`
- `MaritimeService`

### 3.3 Python 包装 Service

特点:

- 自己不直接做复杂数据逻辑
- 只是把请求转给 `PythonRunner`
- 将脚本输出解析成 Qt / C++ 可消费的数据

适合参考:

- `AkShareService`
- `EconomicsService`
- `MarketDataService` 的脚本调用部分

### 3.4 DataHub Producer 型 Service

特点:

- 不只是“给一个 screen 用”
- 要成为共享数据源
- 通过 topic 供多个 screen / widget / MCP 工具复用

适合参考:

- `MarketDataService`
- `NewsService`
- `EconomicsService`

如果你不确定先做哪种，优先从最简单的非 Producer 版本开始。

---

## 4. 推荐的新增顺序

建议按下面顺序做:

1. 先定义 service 的职责
2. 先确定输入输出类型
3. 先做最小可调用版本
4. 让一个 screen 先接通
5. 再判断是否值得升级成 `DataHub::Producer`
6. 最后再补缓存、持久化、配置、MCP

这样做的好处是:

- 能快速验证责任边界
- 不会一开始就把 service 设计得过重

---

## 5. Service 的基本结构

大多数 service 都是:

- `QObject`
- 单例 `instance()`
- 公开业务方法
- 用 `signals` 通知 screen 或其他调用方

最小模板如下。

### 5.1 头文件模板

文件示例:

- `src/services/research_lab/ResearchLabService.h`

```cpp
#pragma once

#include <QObject>
#include <QString>

namespace fincept::services {

class ResearchLabService : public QObject {
    Q_OBJECT
  public:
    static ResearchLabService& instance();

    void load_overview(const QString& symbol);

  signals:
    void overview_loaded(QString symbol, QString summary);
    void overview_failed(QString error);

  private:
    ResearchLabService();
};

} // namespace fincept::services
```

### 5.2 实现文件模板

文件示例:

- `src/services/research_lab/ResearchLabService.cpp`

```cpp
#include "services/research_lab/ResearchLabService.h"

namespace fincept::services {

ResearchLabService& ResearchLabService::instance() {
    static ResearchLabService s;
    return s;
}

ResearchLabService::ResearchLabService() {}

void ResearchLabService::load_overview(const QString& symbol) {
    if (symbol.trimmed().isEmpty()) {
        emit overview_failed("Symbol is empty");
        return;
    }

    emit overview_loaded(symbol, QString("Overview for %1").arg(symbol));
}

} // namespace fincept::services
```

这个版本还没接数据库、HTTP、Python，但已经建立了 service 形态。

---

## 6. 先让它进入构建系统

新增 service 后，通常要把 `.cpp` 加到 `CMakeLists.txt` 的 `SERVICE_SOURCES`。

文件:

- [CMakeLists.txt](file:///d:/open-source/FinceptTerminal/fincept-qt/CMakeLists.txt)

在 `set(SERVICE_SOURCES ...)` 区域加入:

```cmake
src/services/research_lab/ResearchLabService.cpp
```

如果你忘了这一步，常见现象是:

- 头文件能被识别
- 编译时链接不到实现
- 代码看起来写完了，但构建产物里没有这项功能

---

## 7. Service 应该暴露什么接口

建议优先想清楚 3 件事:

1. 调用方怎么触发它
2. 结果怎么回传
3. 错误怎么表达

### 7.1 触发方式

常见有:

- 公共成员函数
- 少量直接返回 `Result<T>`
- 异步函数 + callback
- 异步函数 + Qt signals

### 7.2 回传方式

项目里常见 3 种:

- callback
- signals
- DataHub publish

经验建议:

- 只给一个调用方、偏一次性 -> callback 可以
- 面向 screen、Qt 生态 -> signals 更自然
- 多个订阅方、共享数据 -> DataHub

### 7.3 错误表达

优先使用:

- `signals` 发错误
- callback 里带 `ok/error`
- Repository 层用 `Result<T>`

不要让 screen 自己去猜失败原因。

---

## 8. 4 种常见接法

---

## 8.1 Repository / CRUD 型 Service

适合:

- 本地业务对象管理
- 用户数据、配置、工作流、组合等

典型路径:

```text
Screen -> Service -> Repository -> SQLite
```

### 8.1.1 什么时候选它

- 数据主要在本地
- 页面需要统一读写入口
- 业务逻辑不适合散落在 screen 中

### 8.1.2 设计建议

- service 暴露业务动作，如 `load_xxx()`、`save_xxx()`、`delete_xxx()`
- repository 负责具体 SQL / 持久化
- screen 只 connect 信号，不直接碰 repository

### 8.1.3 参考模式

`PortfolioScreen` 会连接 `PortfolioService` 的大量信号，而不是自己直接访问数据库。

参考:

- [PortfolioScreen.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/screens/portfolio/PortfolioScreen.cpp)
- [PortfolioService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/portfolio/PortfolioService.h)

---

## 8.2 Python 包装型 Service

适合:

- 数据源本身已经有 Python 脚本
- 项目里已有对应脚本生态
- C++ 只需要调脚本、拿 JSON、解析后给页面

典型路径:

```text
Screen -> Service -> PythonRunner -> scripts/*.py -> JSON -> Service -> Screen
```

### 8.2.1 什么时候选它

- 已有 Python 数据脚本最省事
- 数据接入逻辑复杂，放 Python 更方便
- 不需要实时共享流，只是用户触发查询

### 8.2.2 最值得参考的样板

- [AkShareService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/akshare/AkShareService.h)
- [AkShareService.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/akshare/AkShareService.cpp)

这个 service 的定位非常清楚:

- 不直接让 screen spawn Python
- service 内部调 `PythonRunner`
- 将 JSON 解析为 `EndpointsResult` / `QueryResult`
- 回调给 screen

### 8.2.3 推荐做法

- 在 service 层定义明确的 result struct
- 统一解析脚本返回 JSON
- 统一记录日志
- 统一做错误翻译

### 8.2.4 最小模板

```cpp
struct QueryResult {
    bool success = false;
    QJsonObject data;
    QString error;
};

using QueryCallback = std::function<void(const QueryResult&)>;

class ResearchLabService : public QObject {
    Q_OBJECT
  public:
    static ResearchLabService& instance();
    void run_query(const QString& symbol, QueryCallback cb);
};
```

```cpp
void ResearchLabService::run_query(const QString& symbol, QueryCallback cb) {
    python::PythonRunner::instance().run(
        "research_lab.py", {symbol},
        [cb = std::move(cb)](const python::PythonResult& result) {
            QueryResult out;
            if (!result.success) {
                out.error = result.error;
                if (cb) cb(out);
                return;
            }

            const QString json_str = python::extract_json(result.output);
            QJsonParseError err;
            const auto doc = QJsonDocument::fromJson(json_str.toUtf8(), &err);
            if (doc.isNull() || !doc.isObject()) {
                out.error = err.errorString();
                if (cb) cb(out);
                return;
            }

            out.success = true;
            out.data = doc.object();
            if (cb) cb(out);
        });
}
```

### 8.2.5 什么时候不要急着上 DataHub

像 AkShare 这种“浏览器式、用户主动点查询”的能力，不一定适合一开始就做成 Producer。

如果满足下面任意一条，先别上 DataHub:

- 没有实时性
- 不需要跨多个页面共享
- topic 设计还不稳定
- 只是临时或探索型功能

---

## 8.3 HTTP / WebSocket 型 Service

适合:

- 外部接口本来就是在线请求
- 不依赖 Python 脚本
- 需要网络层更细粒度控制

典型路径:

```text
Screen -> Service -> HttpClient / QNetworkAccessManager -> JSON -> Service -> Screen
```

### 8.3.1 什么时候选它

- 目标 API 简单明确
- C++ 直接请求更省事
- 不想增加 Python 依赖

### 8.3.2 `NewsService` 是典型参考

`NewsService` 既有 Qt signals，也支持 `DataHub::Producer`，非常适合当“中大型 service”样板。

参考:

- [NewsService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/news/NewsService.h)

### 8.3.3 设计建议

- 把网络请求和解析放 service
- 不要把 `QNetworkReply` 细节泄露给 screen
- screen 只关心“数据到了没”“错误是什么”

---

## 8.4 DataHub Producer 型 Service

适合:

- 数据会被多个 screen / widget 共享
- 数据有 topic 语义
- 需要统一刷新策略、缓存 TTL、节流

典型路径:

```text
Producer Service -> DataHub topic -> 多个消费者订阅
```

### 8.4.1 什么时候值得做 Producer

满足这些条件时很值得:

- 多个页面都要用这份数据
- 刷新成本高，不能每个页面各拉一遍
- 需要订阅型更新
- 想让 MCP / AI / widget 也能复用

### 8.4.2 最好的参考

- [MarketDataService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/markets/MarketDataService.h)
- [MarketDataService.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/markets/MarketDataService.cpp)
- [DataHub.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/datahub/DataHub.h)
- [DATAHUB_ARCHITECTURE.md](file:///d:/open-source/FinceptTerminal/fincept-qt/DATAHUB_ARCHITECTURE.md)

### 8.4.3 你需要实现什么

一个 Producer 型 service 通常要做这些事:

- 继承 `QObject`
- 继承 `fincept::datahub::Producer`
- 实现:
  - `topic_patterns()`
  - `refresh(const QStringList& topics)`
  - `max_requests_per_sec()`
- 增加 `ensure_registered_with_hub()`
- 在合适时机 `DataHub::publish(...)`

### 8.4.4 最小模板

```cpp
class ResearchLabService : public QObject, public fincept::datahub::Producer {
    Q_OBJECT
  public:
    static ResearchLabService& instance();

    void ensure_registered_with_hub();

    QStringList topic_patterns() const override;
    void refresh(const QStringList& topics) override;
    int max_requests_per_sec() const override;

  private:
    ResearchLabService();
    bool hub_registered_ = false;
};
```

```cpp
QStringList ResearchLabService::topic_patterns() const {
    return {QStringLiteral("research:overview:*")};
}

int ResearchLabService::max_requests_per_sec() const {
    return 2;
}

void ResearchLabService::refresh(const QStringList& topics) {
    for (const auto& topic : topics) {
        const QString symbol = topic.section(':', 2, 2);
        QJsonObject payload;
        payload["symbol"] = symbol;
        payload["summary"] = QString("Overview for %1").arg(symbol);
        datahub::DataHub::instance().publish(topic, payload);
    }
}

void ResearchLabService::ensure_registered_with_hub() {
    if (hub_registered_)
        return;
    auto& hub = datahub::DataHub::instance();
    hub.register_producer(this);

    datahub::TopicPolicy policy;
    policy.ttl_ms = 60'000;
    policy.min_interval_ms = 5'000;
    hub.set_policy_pattern(QStringLiteral("research:overview:*"), policy);

    hub_registered_ = true;
}
```

### 8.4.5 什么时候不要一上来就做 Producer

先别做，如果:

- 只有一个页面会用
- 数据结构还在快速变
- 页面功能还没稳定
- 只是用户手动点击一次查询

否则会过早把 topic 契约固化。

---

## 9. Service 和 Screen 怎么连接

最常见模式是 screen 在构造函数里 connect service 信号。

例如 `PortfolioScreen` 在构造时集中连接 `PortfolioService` 信号。

参考:

- [PortfolioScreen.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/screens/portfolio/PortfolioScreen.cpp)

推荐模式:

```cpp
auto& svc = services::ResearchLabService::instance();
connect(&svc, &services::ResearchLabService::overview_loaded,
        this, &ResearchLabScreen::on_overview_loaded);
connect(&svc, &services::ResearchLabService::overview_failed,
        this, &ResearchLabScreen::on_overview_failed);
```

然后在用户动作里调用:

```cpp
services::ResearchLabService::instance().load_overview("AAPL");
```

不要让 screen:

- 直接调 `HttpClient`
- 直接调 `PythonRunner`
- 直接写复杂 JSON 解析

---

## 10. 什么时候要加 Repository

如果 service 需要落地保存数据，建议加 repository。

适合:

- 用户创建和编辑的数据
- 设置项
- watchlist
- portfolio
- workflow
- report
- data source 连接配置

不一定需要:

- 临时结果
- 短期缓存
- 纯实时流

判断标准:

- “重启后还要存在” -> Repository
- “只是当前会话里显示一下” -> 不一定要 Repository

---

## 11. 什么时候要加 `Types.h`

当 service 的输入输出开始变复杂时，建议单独加类型头文件。

适合:

- 对外有多个结果结构
- screen 和 service 之间要共享类型
- 可能要注册到 Qt 元类型或 DataHub

参考:

- `PortfolioTypes.h`
- `PolymarketTypes.h`
- `BacktestingTypes.h`

不建议一开始就加:

- 只有一个很小的返回结构
- 还在试验阶段

---

## 12. Python 型 Service 的设计要点

对照 [PYTHON_CONTRIBUTOR_GUIDE.md](file:///d:/open-source/FinceptTerminal/docs/PYTHON_CONTRIBUTOR_GUIDE.md)，这类 service 最重要的是约束 Python 脚本接口。

### 12.1 推荐规则

- Python 脚本输入尽量清晰
- 输出尽量统一 JSON
- service 层统一解析 JSON
- 不让 screen 感知脚本原始 stdout

### 12.2 推荐脚本通信习惯

脚本最好输出:

```json
{"success": true, "data": {...}}
```

或者:

```json
{"success": false, "error": "message"}
```

这样 service 层最容易统一解析。

### 12.3 最常见踩坑点

- 脚本输出混杂日志和 JSON
- `stdout` 不是合法 JSON
- 参数格式在 C++ 和 Python 侧不一致
- 环境变量 key 名和 `SecureStorage` 里保存的不一致

---

## 13. DataHub 型 Service 的设计要点

如果决定上 DataHub，最重要的是 topic 设计。

### 13.1 topic 要稳定

建议遵循:

```text
family:sub:id[:qualifier]
```

例如:

- `market:quote:AAPL`
- `news:symbol:NVDA`
- `econ:fred:GDP`

### 13.2 topic 不要太随意

坏例子:

- `mydata1`
- `screen_temp_topic`
- `overview_final_new`

好例子:

- `research:overview:AAPL`
- `research:history:AAPL:1y`

### 13.3 policy 要合理

你要想清楚:

- `ttl_ms`
- `min_interval_ms`
- 是否 `push_only`
- 是否需要 `pause_when_inactive`

不要把所有 topic 都设成高频刷新。

---

## 14. 什么时候要在 `main.cpp` 里初始化 Service

不是所有 service 都要在启动时主动初始化。

### 14.1 需要在 `main.cpp` 里处理的情况

适合:

- 需要 `ensure_registered_with_hub()`
- 需要启动时注册 adapter / provider
- 需要进程级生命周期管理

例如:

- `MarketDataService`
- `NewsService`
- `EconomicsService`

### 14.2 不需要放到 `main.cpp` 的情况

适合:

- 懒加载页面服务
- 单页面使用
- 用户进入页面后再初始化也没问题

例如:

- `AkShareService`

经验建议:

- 没有全局注册需求，就先别往 `main.cpp` 塞

---

## 15. 新增 Service 的完整改动清单

假设新增 `ResearchLabService`，最常见需要改这些文件:

- 新增 `src/services/research_lab/ResearchLabService.h`
- 新增 `src/services/research_lab/ResearchLabService.cpp`
- 修改 `CMakeLists.txt` 的 `SERVICE_SOURCES`

如果有 screen 消费:

- 修改 `src/screens/research_lab/ResearchLabScreen.cpp`

如果要接 Python:

- 新增 `scripts/research_lab.py`

如果要持久化:

- 新增或修改 `src/storage/repositories/...`
- 必要时新增 migration

如果要接 DataHub:

- 实现 `Producer`
- 在 `main.cpp` 或合适位置调用 `ensure_registered_with_hub()`

---

## 16. 推荐的最小落地顺序

### 第 1 步

先写一个同步假实现，确认 screen 到 service 的调用通路是通的。

### 第 2 步

再接真实数据源:

- Repository
- HTTP
- Python
- WebSocket

### 第 3 步

如果多个消费者开始出现，再升级到 `DataHub`

### 第 4 步

最后再考虑:

- 缓存
- 速率限制
- 配置项
- API Key
- MCP 集成

这样最稳。

---

## 17. 常见错误

### 17.1 Service 变成“胖页面的搬运工”

表现:

- service 只是把 screen 里的代码原样搬走
- 没有更清晰的边界

修正:

- 让 service 真正承担数据与业务责任

### 17.2 Screen 直接碰网络或 Python

表现:

- `Screen.cpp` 里直接写 `HttpClient`
- 直接 spawn Python

修正:

- 全部收回到 service

### 17.3 过早上 DataHub

表现:

- 功能还没稳定就设计一堆 topic
- 最后 topic 契约频繁重构

修正:

- 先做非 Producer 版本，稳定后再升级

### 17.4 Service 没有清晰错误输出

表现:

- 页面只能看到“没反应”

修正:

- signal / callback 里明确发错误
- service 内统一记录日志

### 17.5 Repository、Service、Screen 三层职责混乱

表现:

- Repository 开始做业务判断
- Screen 开始做 SQL
- Service 开始直接拼 UI 文本

修正:

- Repository 只管数据访问
- Service 只管业务和数据流
- Screen 只管展示和交互

---

## 18. 最值得照着抄的几个 service

### 18.1 轻量 Python 包装

- [AkShareService.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/akshare/AkShareService.cpp)

适合学习:

- callback 风格
- JSON 解析
- service 不让 screen 直接碰脚本

### 18.2 DataHub Producer

- [MarketDataService.cpp](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/markets/MarketDataService.cpp)

适合学习:

- `Producer` 实现
- topic 切分
- `ensure_registered_with_hub()`
- 批量请求和 publish

### 18.3 中大型业务 service

- [NewsService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/news/NewsService.h)
- [PortfolioService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/portfolio/PortfolioService.h)
- [WorkflowService.h](file:///d:/open-source/FinceptTerminal/fincept-qt/src/services/workflow/WorkflowService.h)

适合学习:

- signal 设计
- 复杂职责拆分
- 本地数据和远程数据混合处理

---

## 19. 最短版本清单

如果你只想记住新增 service 的最短步骤，记这 6 条:

1. 新建 `src/services/<domain>/<Name>Service.h/.cpp`
2. 用 `QObject + instance()` 建立最小 service 结构
3. 把 `.cpp` 加进 `CMakeLists.txt` 的 `SERVICE_SOURCES`
4. 在 screen 中 connect 这个 service 的信号或调用其方法
5. 再决定它是 Repository 型、Python 型、HTTP 型还是 DataHub 型
6. 只有在多消费者共享数据时，才升级成 `DataHub::Producer`

---

## 20. 下一步建议

如果你接下来要真正高频改这个项目，最值得继续补的文档是:

- `DEBUGGING_CHECKLIST.md`
- `HOW_TO_ADD_A_PYTHON_PROVIDER.md`
- `HOW_TO_ADD_A_DATASOURCE_CONNECTOR.md`

因为实际开发里，最常见的问题不是“service 怎么写”，而是:

- 服务明明写了，页面没接对
- 脚本能跑，JSON 不对
- topic 发了，UI 没订阅
- 连接器做了，但没进 Data Sources 管理体系

把这些再补上，整个 `Fincept` 的二开文档体系就会很完整。
