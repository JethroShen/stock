# AGENTS.md

## 1. 项目概览

本项目是一个面向 A 股与 ETF 的 Python 量化分析系统，核心能力包括：

- 每日行情与专题数据抓取
- MySQL 持久化存储与自动建表
- 技术指标计算
- K 线形态识别
- 内置策略选股
- 结果回测补全
- Tornado Web 可视化展示
- easytrader 自动交易
- Docker / cron / supervisor 部署

仓库当前没有前后端分离结构，整体是一个“单仓单体应用”：

- `instock/core` 负责抓取、计算、策略、图表和元数据
- `instock/job` 负责调度批处理
- `instock/web` 负责页面与接口
- `instock/trade` 负责自动交易
- `instock/lib` 负责数据库、交易日、公用运行模板等基础设施

项目主数据流：

1. `job` 调用 `core.stockfetch` 和 `core.singleton_*`
2. 数据写入 MySQL
3. `core.tablestructure` 和 `core.singleton_stock_web_module_data` 决定表结构与页面模块
4. `web` 从 MySQL 读表并渲染列表、指标图和 K 线图
5. `trade` 独立运行，按时钟事件加载策略并调用 easytrader 下单

## 2. 技术栈

- 语言：Python 3.11
- 数据处理：`pandas`、`numpy`
- 指标/形态：`TA-Lib`
- Web：`tornado`、`bokeh`
- 数据库：MySQL / MariaDB，访问方式为 `PyMySQL` + `SQLAlchemy` + 自带 `torndb`
- 交易：`easytrader`
- 其他：`requests`、`beautifulsoup4`、`py_mini_racer` / `mini-racer`

依赖入口文件：[`requirements.txt`](D:\workspac_ai\stock\requirements.txt)

## 3. 目录结构

```text
stock/
├─ .github/                CI 与镜像工作流
├─ cron/                   容器内定时任务脚本
├─ docker/                 Dockerfile 与 compose
├─ img/                    README 截图
├─ instock/                主应用代码
│  ├─ bin/                 启动脚本
│  ├─ config/              代理、Cookie、交易配置
│  ├─ core/                抓取/指标/策略/图表/元数据核心
│  ├─ job/                 日常批处理作业
│  ├─ lib/                 基础设施与通用库
│  ├─ trade/               自动交易模块
│  └─ web/                 Tornado 页面、接口、静态资源、模板
├─ supervisor/             容器进程编排
├─ README.md               功能与部署说明
└─ AGENTS.md               本文档
```

## 4. 顶层目录说明

### `.github/workflows`

- `docker-image.yml`：构建并发布 Docker 镜像。
- `azure-container-webapp.yml`：面向 Azure Web App 的部署流程。

### `cron`

容器内 cron 任务定义：

- `cron.hourly/run_hourly`：运行 `basic_data_daily_job.py`，用于盘中/高频基础抓取。
- `cron.workdayly/run_workdayly`：运行 `execute_daily_job.py`，用于工作日全量作业。
- `cron.monthly/run_monthly`：清理 `instock/cache/hist/*` 历史缓存。

### `docker`

- [`docker/Dockerfile`](D:\workspac_ai\stock\docker\Dockerfile)：基于 `python:3.11-slim-bullseye`，安装 TA-Lib、依赖、cron、supervisor，并将项目复制到 `/data/InStock`。
- [`docker/docker-compose.yml`](D:\workspac_ai\stock\docker\docker-compose.yml)：同时启动 `instock` 和 `mariadb`。
- `build.sh`：镜像构建脚本。

### `supervisor`

- [`supervisor/supervisord.conf`](D:\workspac_ai\stock\supervisor\supervisord.conf)：容器内同时托管
  - `run_job`
  - `run_web`
  - `run_cron`

## 5. `instock/` 详细结构

### 5.1 `instock/bin`

运维/启动入口：

- `run_job.sh` / `run_job.bat`：执行批处理作业
- `run_web.sh` / `run_web.bat`：启动 Tornado Web
- `run_trade.bat`：启动自动交易
- `run_cron.sh`：启动 cron
- `restart_web.sh`：重启 Web 辅助脚本

最重要的三个启动方向：

- 数据：[`instock/job/execute_daily_job.py`](D:\workspac_ai\stock\instock\job\execute_daily_job.py)
- Web：[`instock/web/web_service.py`](D:\workspac_ai\stock\instock\web\web_service.py)
- 交易：[`instock/trade/trade_service.py`](D:\workspac_ai\stock\instock\trade\trade_service.py)

### 5.2 `instock/config`

运行期配置文件：

- `proxy.txt`：代理列表
- `eastmoney_cookie.txt`：东方财富 Cookie
- `trade_client.json`：easytrader 账户配置

说明：

- 数据抓取大量依赖东方财富、Sina 等源站。
- 代理与 Cookie 的目的主要是降低频控失败概率。

### 5.3 `instock/lib`

基础设施层。

关键文件：

- [`instock/lib/database.py`](D:\workspac_ai\stock\instock\lib\database.py)
  - 读取环境变量中的数据库连接参数
  - 暴露 SQLAlchemy engine、PyMySQL 连接
  - 封装 DataFrame 入库、更新、执行 SQL、检查表存在
  - 首次写表后自动补主键和索引
- [`instock/lib/run_template.py`](D:\workspac_ai\stock\instock\lib\run_template.py)
  - 为各 job 统一处理命令行日期参数
  - 支持当前日期、单日期、多日期、日期区间
  - 自动识别交易日并并发运行
- `trade_time.py`
  - 交易日判断
  - 最近交易日获取
  - 历史区间计算
- `singleton_type.py`
  - 单例元类
- `torndb.py`
  - Web 层数据库连接封装
- `crypto_aes.py`
  - AES 工具，偏辅助用途
- `version.py`
  - 版本号

### 5.4 `instock/core`

项目核心域，几乎所有业务能力都在这里。

#### 5.4.1 表结构与元数据

- [`instock/core/tablestructure.py`](D:\workspac_ai\stock\instock\core\tablestructure.py)

这是全项目最关键的元数据文件之一，负责定义：

- 数据表名
- 中文标题
- 字段类型
- 字段中文名
- Web 展示宽度
- 指标列集合
- K 线形态映射
- 策略列表与策略函数绑定
- 选股大表字段映射

可以把它理解为“数据库模式 + 页面字段配置 + 策略注册表”的合体。

源码中最重要的几组定义：

- `TABLE_CN_STOCK_SPOT`：每日股票基础数据
- `TABLE_CN_ETF_SPOT`：每日 ETF 数据
- `TABLE_CN_STOCK_FUND_FLOW`：股票资金流
- `TABLE_CN_STOCK_INDICATORS`：技术指标结果表
- `TABLE_CN_STOCK_KLINE_PATTERN`：K 线形态结果表
- `TABLE_CN_STOCK_STRATEGIES`：策略注册表
- `TABLE_CN_STOCK_SELECTION`：综合选股结果表

#### 5.4.2 抓取聚合层

- [`instock/core/stockfetch.py`](D:\workspac_ai\stock\instock\core\stockfetch.py)

这是各 job 的统一抓取入口，职责包括：

- 过滤 A 股代码
- 过滤 ST、无开盘数据
- 聚合多个 crawling 子模块
- 统一字段命名
- 注入日期列
- 历史行情缓存到 `instock/cache/hist`

主要函数：

- `fetch_stocks(date)`：抓股票实时/日行情
- `fetch_etfs(date)`：抓 ETF 行情
- `fetch_stock_selection()`：抓综合选股结果
- `fetch_stocks_fund_flow(index)`：抓个股资金流
- `fetch_stocks_sector_fund_flow(...)`：抓行业/概念资金流
- `fetch_stocks_bonus(date)`：抓分红配送
- `fetch_stock_lhb_data(date)`：抓东方财富龙虎榜
- `fetch_stock_top_data(date)`：抓新浪龙虎榜统计
- `fetch_stock_blocktrade_data(date)`：抓大宗交易
- `fetch_stock_chip_race_open/end(date)`：抓早盘/尾盘抢筹
- `fetch_stock_limitup_reason(date)`：抓涨停原因
- `fetch_stock_hist(...)` / `fetch_etf_hist(...)`：抓历史 K 线并计算基础涨跌幅

#### 5.4.3 `core/crawling`

具体数据源适配层，属于 `stockfetch.py` 下游。

主要文件：

- `trade_date_hist.py`：交易日历
- `fund_etf_em.py`：ETF 数据
- `stock_hist_em.py`：股票实时与历史行情
- `stock_fund_em.py`：个股/板块资金流
- `stock_fhps_em.py`：分红配送
- `stock_lhb_em.py`：东方财富龙虎榜
- `stock_lhb_sina.py`：新浪龙虎榜
- `stock_dzjy_em.py`：大宗交易
- `stock_chip_race.py`：抢筹数据
- `stock_limitup_reason.py`：涨停原因
- `stock_selection.py`：综合选股接口
- `stock_cpbd.py`：操盘必读类数据

命名规律：

- `_em` 基本表示东方财富数据源
- `_sina` 表示新浪数据源

#### 5.4.4 单例缓存层

关键文件：

- [`instock/core/singleton_stock.py`](D:\workspac_ai\stock\instock\core\singleton_stock.py)
- `singleton_trade_date.py`
- `singleton_proxy.py`
- `singleton_stock_web_module_data.py`

用途：

- 共享当日股票列表
- 共享历史行情批量结果
- 共享交易日
- 共享代理
- 共享 Web 模块定义

其中 `stock_hist_data` 会并发拉取多只股票历史数据，是指标、形态、策略、回测的公共上游。

#### 5.4.5 指标计算

- [`instock/core/indicator/calculate_indicator.py`](D:\workspac_ai\stock\instock\core\indicator\calculate_indicator.py)

这是技术指标核心实现，基于 TA-Lib 和手工公式混合计算。

特点：

- 指标种类多
- 某些指标用手工公式修正，以靠近同花顺/通信达
- 自动处理 `NaN` / `Inf`
- 支持按截止日期截断计算

主要能力：

- MACD / KDJ / BOLL / TRIX / RSI / WR / CCI / ATR / DMI
- MFI / VWMA / PPO / WT / Supertrend / ROC / OBV / SAR
- PSY / BRAR / EMV / BIAS / DPO / VHF / RVI / Force Index / ENE
- 成交量均线 / MA 均线等

对外主要接口：

- `get_indicators(data, ...)`
- `get_indicator(code_name, data, stock_column, date=None, ...)`

#### 5.4.6 K 线形态识别

- [`instock/core/pattern/pattern_recognitions.py`](D:\workspac_ai\stock\instock\core\pattern\pattern_recognitions.py)

职责：

- 遍历 `tablestructure.py` 中登记的 TA-Lib 形态函数
- 对单只股票历史 K 线打分
- 仅返回当日真正出现的形态

返回规则：

- `> 0`：买入信号
- `< 0`：卖出信号
- `0`：未出现

#### 5.4.7 选股策略

目录：`instock/core/strategy`

策略文件与含义：

- `enter.py`：放量上涨
- `keep_increasing.py`：均线多头
- `parking_apron.py`：停机坪
- `backtrace_ma250.py`：回踩年线
- `breakthrough_platform.py`：突破平台
- `low_backtrace_increase.py`：无大幅回撤
- `turtle_trade.py`：海龟交易法则
- `high_tight_flag.py`：高而窄的旗形
- `climax_limitdown.py`：放量跌停
- `low_atr.py`：低 ATR 成长

策略注册中心在 `tablestructure.py` 的 `TABLE_CN_STOCK_STRATEGIES`。

调度时通过表定义里的 `func` 字段动态调用对应策略函数。

#### 5.4.8 回测

目录：`instock/core/backtest`

- `rate_stats.py`：根据选股日期后的未来行情，回填 N 日收益率等回测字段。

回测不是一个独立策略框架，而是对“指标买卖表 + 策略表”的结果做后处理补数。

#### 5.4.9 K 线可视化

目录：`instock/core/kline`

关键文件：

- [`instock/core/kline/visualization.py`](D:\workspac_ai\stock\instock\core\kline\visualization.py)
- `indicator_web_dic.py`
- `cyq.py`
- `cyq.js`

职责：

- 生成 Bokeh K 线图
- 叠加均线、成交量、指标分页
- 叠加 K 线形态标签
- 叠加筹码分布图
- 生成 HTML `script/div` 片段给 Tornado 模板渲染

这里是项目里最“前端逻辑密集”的 Python 文件。

### 5.5 `instock/job`

批处理调度层，每个文件通常负责一类数据表的生产。

#### 核心入口

- [`instock/job/execute_daily_job.py`](D:\workspac_ai\stock\instock\job\execute_daily_job.py)

默认执行顺序：

1. `init_job.main()`：建库与基础表初始化
2. `basic_data_daily_job.main()`：实时基础行情
3. `selection_data_daily_job.main()`：综合选股
4. `basic_data_other_daily_job.main()`：龙虎榜、资金流、分红等
5. `basic_data_after_close_daily_job.main()`：收盘后数据

代码里原本还预留了：

- 指标计算
- K 线形态
- 策略选股
- 回测

但默认入口里有一部分被注释掉了，说明当前主流程偏“基础数据优先”，而不是“全功能默认全开”。

#### 各作业说明

- [`instock/job/init_job.py`](D:\workspac_ai\stock\instock\job\init_job.py)
  - 数据库不存在时自动创建
  - 初始化 `cn_stock_attention`

- [`instock/job/basic_data_daily_job.py`](D:\workspac_ai\stock\instock\job\basic_data_daily_job.py)
  - 写入 `cn_stock_spot`
  - 写入 `cn_etf_spot`

- [`instock/job/basic_data_other_daily_job.py`](D:\workspac_ai\stock\instock\job\basic_data_other_daily_job.py)
  - 龙虎榜
  - 分红配送
  - 个股资金流
  - 行业/概念资金流
  - 早盘抢筹
  - 涨停原因
  - 附带一个基本面选股 `stock_spot_buy`

- [`instock/job/basic_data_after_close_daily_job.py`](D:\workspac_ai\stock\instock\job\basic_data_after_close_daily_job.py)
  - 大宗交易
  - 尾盘抢筹

- [`instock/job/selection_data_daily_job.py`](D:\workspac_ai\stock\instock\job\selection_data_daily_job.py)
  - 综合选股大表写库

- [`instock/job/indicators_data_daily_job.py`](D:\workspac_ai\stock\instock\job\indicators_data_daily_job.py)
  - 为全市场股票计算指标表
  - 根据阈值二次筛出买入表和卖出表

- [`instock/job/klinepattern_data_daily_job.py`](D:\workspac_ai\stock\instock\job\klinepattern_data_daily_job.py)
  - 识别全市场当日 K 线形态

- [`instock/job/strategy_data_daily_job.py`](D:\workspac_ai\stock\instock\job\strategy_data_daily_job.py)
  - 并行运行全部内置策略
  - 每个策略产出一张结果表

- [`instock/job/backtest_data_daily_job.py`](D:\workspac_ai\stock\instock\job\backtest_data_daily_job.py)
  - 遍历指标/策略结果表
  - 将尚未回填的回测字段更新回原表

### 5.6 `instock/web`

Tornado Web 层。

关键文件：

- [`instock/web/web_service.py`](D:\workspac_ai\stock\instock\web\web_service.py)
  - 服务入口
  - 路由注册
  - 初始化全局数据库连接
- [`instock/web/base.py`](D:\workspac_ai\stock\instock\web\base.py)
  - 统一基类
  - 每次请求先检查数据库连接
- [`instock/web/dataTableHandler.py`](D:\workspac_ai\stock\instock\web\dataTableHandler.py)
  - 列表页 HTML
  - 列表数据 JSON
- [`instock/web/dataIndicatorsHandler.py`](D:\workspac_ai\stock\instock\web\dataIndicatorsHandler.py)
  - 个股指标/K 线详情页
  - 关注/取关接口

路由：

- `/`、`/instock/`：首页
- `/instock/data`：模块列表页
- `/instock/api_data`：表数据接口
- `/instock/data/indicators`：个股指标图页
- `/instock/control/attention`：关注操作

#### 模板与静态资源

- `templates/`
  - `index.html`
  - `stock_web.html`
  - `stock_indicators.html`
  - `common/` 公共片段
  - `layout/` 页面布局
- `static/`
  - Bootstrap / jQuery / Bokeh
  - SpreadJS 相关前端资源
  - 字体与图标

#### Web 模块装配机制

- [`instock/core/singleton_stock_web_module_data.py`](D:\workspac_ai\stock\instock\core\singleton_stock_web_module_data.py)
- [`instock/core/web_module_data.py`](D:\workspac_ai\stock\instock\core\web_module_data.py)

这里定义左侧菜单和页面模块清单。新增一个数据表想出现在 Web 中，通常需要：

1. 在 `tablestructure.py` 定义表和列
2. 在 `singleton_stock_web_module_data.py` 注册模块
3. 确保 job 会把数据写进该表

也就是说，Web 的扩展方式偏“配置驱动”，不是每个表都单写 handler。

### 5.7 `instock/trade`

自动交易子系统，和数据分析主流程相对独立。

关键文件：

- [`instock/trade/trade_service.py`](D:\workspac_ai\stock\instock\trade\trade_service.py)
  - 交易服务入口
  - 加载 easytrader broker
  - 启动主引擎
- [`instock/trade/usage.md`](D:\workspac_ai\stock\instock\trade\usage.md)
  - easytrader 配置说明

#### `trade/robot/engine`

- `event_engine.py`：事件总线
- `clock_engine.py`：时钟事件驱动
- [`main_engine.py`](D:\workspac_ai\stock\instock\trade\robot\engine\main_engine.py)
  - 加载策略模块
  - 监听文件变更热重载
  - 绑定时钟事件到策略
  - 管理优雅退出

#### `trade/robot/infrastructure`

- `strategy_template.py`：策略基类
- `strategy_wrapper.py`：策略包装
- `default_handler.py`：日志处理

#### `trade/strategies`

- [`stagging.py`](D:\workspac_ai\stock\instock\trade\strategies\stagging.py)
  - 内置“打新股”策略
  - 在交易日 10:00 触发 `user.auto_ipo()`
- `stratey1.py`
  - 示例或扩展策略文件

交易模块与分析模块的关系：

- 代码结构上相互独立
- 交易策略目前不直接消费分析结果表
- 更像是仓库内并存的第二套能力

## 6. 数据库设计理解

项目以“单表对应单能力结果”的方式组织数据。

常见类型：

- 原始/准原始数据表
  - `cn_stock_spot`
  - `cn_etf_spot`
  - `cn_stock_fund_flow`
  - `cn_stock_bonus`
  - `cn_stock_lhb`
  - `cn_stock_blocktrade`
- 分析结果表
  - `cn_stock_indicators`
  - `cn_stock_kline_pattern`
  - 各种 `cn_stock_strategy_*`
- 派生筛选表
  - `cn_stock_indicators_buy`
  - `cn_stock_indicators_sell`
  - `cn_stock_spot_buy`
- 综合宽表
  - `cn_stock_selection`
- 用户行为表
  - `cn_stock_attention`

共同特征：

- 多数表以 `date + code` 或 `date + name` 为主键
- 建表时用 DataFrame 自动落库
- 首次写表后补主键/索引
- 支持重复重跑某个日期数据，先删后写

## 7. 运行链路

### 本地常用入口

数据作业：

```bash
python instock/job/execute_daily_job.py
python instock/job/basic_data_daily_job.py
python instock/job/indicators_data_daily_job.py
python instock/job/strategy_data_daily_job.py
python instock/job/backtest_data_daily_job.py
```

Web：

```bash
python instock/web/web_service.py
```

交易：

```bash
python instock/trade/trade_service.py
```

### 日期参数机制

所有使用 `run_template.run_with_args` 的 job 都支持：

- 当前交易日
- 单日
- 多日枚举
- 日期区间

例如：

```bash
python instock/job/execute_daily_job.py 2026-04-01
python instock/job/execute_daily_job.py 2026-04-01,2026-04-02
python instock/job/execute_daily_job.py 2026-04-01 2026-04-09
```

## 8. 源码阅读优先级

如果后续 Agent 需要快速理解或改造本项目，建议按下面顺序阅读：

1. [`README.md`](D:\workspac_ai\stock\README.md)
2. [`instock/core/tablestructure.py`](D:\workspac_ai\stock\instock\core\tablestructure.py)
3. [`instock/core/stockfetch.py`](D:\workspac_ai\stock\instock\core\stockfetch.py)
4. [`instock/job/execute_daily_job.py`](D:\workspac_ai\stock\instock\job\execute_daily_job.py)
5. [`instock/job/basic_data_other_daily_job.py`](D:\workspac_ai\stock\instock\job\basic_data_other_daily_job.py)
6. [`instock/job/indicators_data_daily_job.py`](D:\workspac_ai\stock\instock\job\indicators_data_daily_job.py)
7. [`instock/job/strategy_data_daily_job.py`](D:\workspac_ai\stock\instock\job\strategy_data_daily_job.py)
8. [`instock/core/singleton_stock_web_module_data.py`](D:\workspac_ai\stock\instock\core\singleton_stock_web_module_data.py)
9. [`instock/web/web_service.py`](D:\workspac_ai\stock\instock\web\web_service.py)
10. [`instock/core/kline/visualization.py`](D:\workspac_ai\stock\instock\core\kline\visualization.py)
11. [`instock/trade/trade_service.py`](D:\workspac_ai\stock\instock\trade\trade_service.py)

## 9. 扩展建议

### 新增一个数据抓取模块

通常需要改动：

1. `instock/core/crawling/` 新增源站适配文件
2. `instock/core/stockfetch.py` 增加统一抓取函数
3. `instock/core/tablestructure.py` 新增表定义
4. `instock/job/` 新增或接入 job
5. `instock/core/singleton_stock_web_module_data.py` 注册到菜单

### 新增一个选股策略

通常需要改动：

1. `instock/core/strategy/` 新增策略文件
2. `instock/core/tablestructure.py` 的 `TABLE_CN_STOCK_STRATEGIES` 注册策略
3. 运行 `strategy_data_daily_job.py`
4. 如需展示，Web 菜单会因策略注册自动扩展

### 新增一个交易策略

通常需要改动：

1. `instock/trade/strategies/` 新增策略类文件
2. 继承 `StrategyTemplate`
3. 在 `init()` 中注册时钟
4. 在 `clock()` 中根据事件触发交易

注意：交易策略和分析策略是两套不同机制，不要混淆。

## 10. 当前代码特征与注意事项

- 这是一个偏脚本化、元数据驱动的 Python 单体项目。
- `tablestructure.py` 过于中心化，改字段或表时影响面很大。
- job 默认大量使用并发线程池，适合 I/O 密集抓取和批量计算。
- Web 层高度依赖数据库现成表，不做复杂业务编排。
- 自动交易模块是独立子系统，风险高于普通数据模块。
- `execute_daily_job.py` 当前默认没有把指标、形态、策略、回测全部启用，若要全链路执行，需要确认注释部分是否应恢复。
- 抓取成功率受代理、Cookie、源站限流、交易日判断影响很大。

## 11. 结论

从结构上看，本项目最核心的设计不是“复杂服务编排”，而是三件事：

1. 用 `stockfetch + crawling` 统一外部数据接入
2. 用 `tablestructure` 统一数据表与展示元数据
3. 用 `job` 将抓取、计算、筛选、回测拆成可单跑的批处理任务

如果把这个仓库看成一个系统，它本质上是：

- 以 MySQL 为中心的数据生产系统
- 以 Tornado 为界面的查询展示系统
- 附带一个 easytrader 驱动的自动交易子系统

后续任何大改动，优先判断它属于哪一层：

- 数据源接入层
- 数据表/元数据层
- 批处理作业层
- Web 展示层
- 自动交易层

按这五层定位后再改，成本最低。
