# InspectGo · 分布式设施巡检与复核系统

> 用现场证据回答“哪里没巡、异常是否复核”。

![3D 校园巡检工作台](docs/screenshots/stage08/01-campus-overview.png)

校园与室内为设计示意，截图使用隔离教学数据。

## 1. 它解决什么问题

| 项 | 内容 |
| --- | --- |
| 给谁用 | 校园巡检员、现场复核人员和值班老师 |
| 在什么场景 | 教学楼与实验实训空间的设施巡检、现场补证和阶段复盘 |
| 解决什么 | 找出当轮未覆盖的必检点，串联异常、复核证据与判断依据 |
| 明确不做什么 | 不替代专业诊断、安全验收，不自动派单、销号或控制真实设备 |

## 2. 核心功能

| 功能 | 当前状态 | 一句话说明 |
| --- | --- | --- |
| 3D 校园与点位导航 | 已实现 | C01—C08 对应独立设备场景，进入巡查打开真实点位详情 |
| 搜索、筛选与覆盖 | 已实现 | 按编号、设施和位置查询；覆盖由当轮有效现场记录计算 |
| T1—T5 与运行历史 | 已实现 | 依赖证据计算任务四态，生成独立 run_id，保留旧运行 |
| 现场记录、照片与复核 | 已实现 | 保存实际文件与人工结论，处理重复提交、冲突、超时及取消 |
| 业务提醒与远程告警 | 已实现 | 待补检独立提醒；有效补证后的新判断确认缺口消除才消警；新远程告警弹窗 |
| 方案比较与条件推演 | 已实现 | 比较甲乙丙方案；what-if 保留父运行，不改现实结果 |
| T5 分析与 AI 助手 | 已实现 | 统计 → Web API → RAG → ML → 工具 → LLM；AI 只解释和建议 |
| 照片 AI 描述 | 部分实现 | 已有辅助摘要流程，需配置支持图像的模型并由用户主动授权 |
| Web 本地语音 | 已实现 | 安装可选 Vosk 中文模型后转写为草稿，人工校对后提交 |

固定 3 区域、8 点位；必检为 C01、C02、C04、C05、C07、C08，轮次窗口为 09:00—09:45。远程状态不等于现场覆盖；取证失败不等于设施故障；上传照片不等于复核通过。

## 3. 10 分钟快速复现

在 Windows 的 **PowerShell 7** 中执行，先安装 Git 和可由 `py -3.11 --version` 检查的 Python 3.11。以下命令从 GitHub `main` 获取源码，不依赖本机盘符。

**仓库访问：** [InspectGo 项目仓库](https://github.com/Kzzzzz-power/Campus-Inspection) 已公开，可按下列命令克隆；默认分支为 `main`。

**获取源码：**

```powershell
git clone --branch main https://github.com/Kzzzzz-power/Campus-Inspection.git
cd Campus-Inspection
```

**首次安装，两条路线共用：**

```powershell
New-Item -ItemType Directory -Force .runtime/tmp | Out-Null
$env:TEMP = Join-Path $PWD '.runtime/tmp'
$env:TMP = $env:TEMP
$env:PIP_CACHE_DIR = Join-Path $PWD '.runtime/pip-cache'
py -3.11 -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements-lock.txt
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

### 路线 A：完整模式（依赖外部服务）

首次执行第一行安装教学 Broker；第二行同时启动 Broker、API 和模拟器，并加载 `.env`。外部 AI 需按第 5 节配置；未配置时保留结构化结果和受限说明。

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File .\Setup-Realtime.ps1
.\.venv\Scripts\python.exe -m dotenv -f .env run -- pwsh -NoProfile -ExecutionPolicy Bypass -File .\Start-Realtime.ps1 -Port 8017 -MqttPort 1884
```

期望结果：终端显示 `Ready: http://127.0.0.1:8017/`，工作台显示 MQTT／WebSocket 实时教学状态。

### 路线 B：最小可跑模式（无密钥 / 无外网 / 依赖不可用时）

首次获取源码和安装依赖需要联网；完成后，本路线关闭 MQTT 与外部模型，不加载 `.env`，可离线运行核心业务。

```powershell
$env:INSPECTGO_MQTT_ENABLED = 'false'
$env:INSPECTGO_SEED_DEMO = 'false'
$env:INSPECTGO_LLM_ENABLED = 'false'
$env:INSPECTGO_VLM_ENABLED = 'false'
pwsh -NoProfile -ExecutionPolicy Bypass -File .\Start.ps1 -Port 8017
```

降级后的能力：3D 导航、搜索、教学实例、任务判断、现场记录、照片、复核与历史保留；实时状态显示未启用，模型解释显示受限，不伪造结果。

**打开页面：** [http://127.0.0.1:8017/](http://127.0.0.1:8017/)，在“轮次与任务”选择 S1—S5，点击“建立对应教学实例”，再“生成运行判断”。初次为空库，切换下拉选项不会改变已保存运行。

两条路线任选其一；Ctrl+C 停止。再次使用只需运行所选路线的启动命令。数据保存在 `backend/inspectgo.db` 与 `backend/uploads/`，不随 Git 上传；重新克隆不会恢复其他电脑的历史记录。

## 4. 环境与依赖

| 软件 | 要求版本 | 检查命令 | 已验证环境 |
| --- | --- | --- | --- |
| Git | 支持 HTTPS 克隆 | `git --version` | 本机已完成克隆与推送 |
| Python | 3.11，其他大版本未验证 | `py -3.11 --version` | 3.11.17 |
| PowerShell | 7.0 起，实时脚本有版本检查 | `$PSVersionTable.PSVersion` | 7.6.5 |
| Chrome | 支持 WebGL2 | 浏览器“关于 Chrome” | 156.0.8078.74 |

API 依赖锁定于 `requirements-lock.txt`，Broker 锁定于 `requirements-broker-lock.txt`。前端为原生 HTML／CSS／JavaScript，Three.js、模型和贴图已随源码提供，无需 npm 安装或前端打包。

**可选本地语音：** 在源码根目录执行下列命令安装依赖并下载中文模型，完成后重启服务。模型不随 Git 提交；缺失时仍可手写现场说明。

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File .\Setup-Speech.ps1
```

## 5. 环境变量

第 3 节已从 `.env.example` 创建 `.env`。需要外部 AI 时执行 `notepad .env`，填写配置后按路线 A 重启；基础演示无需密钥。

| 变量名 | 是否必填 | 作用 | 不填会怎样 |
| --- | --- | --- | --- |
| `INSPECTGO_LLM_ENABLED` | 可选，默认 false | 配好模型后设为 true | 结构化判断与统计可用，模型解释受限 |
| `INSPECTGO_LLM_BASE_URL`、`INSPECTGO_LLM_MODEL`、`INSPECTGO_LLM_API_KEY` | 启用 LLM 时必填 | 服务端模型地址、名称与凭据 | 配置不完整时不给出模型解释 |
| `INSPECTGO_VLM_ENABLED` 与 `INSPECTGO_VLM_BASE_URL`、`INSPECTGO_VLM_MODEL`、`INSPECTGO_VLM_API_KEY` | 可选；启用时需完整配置 | 照片辅助描述 | 实际照片保存、查看和人工复核仍可用 |
| `INSPECTGO_ASR_MODEL_PATH` | 可选 | 本地语音模型目录 | 使用 Setup-Speech 安装的默认模型 |
| `INSPECTGO_HTTP_TRUST_ENV` | 可选，默认 false | 是否使用环境代理请求模型 | 默认直连 |

密钥只放服务端 `.env`，不提交 Git。模型失败不改变 T1—T4 的判断；配置或网络不可用时显示原因。

## 6. 复现实测记录

以下为 2026-11-03 首次复现的操作者登记。原始起止时间与过程截图未归档，复现指南中的起止时刻按示例设定和登记耗时推算。

| 项    | 内容                                                                                                                                                                         |
| ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 测试人  | 本人                                                                                                                                                                         |
| 机器   | CPU：AMD Ryzen 5 9600X，6 核 12 线程；内存：已安装 16 GiB；系统可用总量约 15.16 GiB；操作系统：Windows 11 专业工作站版，25H2，64 位，内部版本 26200.9711；独立显卡：NVIDIA GeForce RTX 5060；集成显卡：AMD Radeon(TM) Graphics |
| 是否首次 | 首次                                                                                                                                                                         |
| 测试日期 | 2026-11-03                                                                                                                                                                 |
| 总耗时  | 路线 A：6 分 57 秒（417 秒）；路线 B：7 分 35 秒（455 秒）                                                                                                                                  |

机器配置于 2026-11-04 采集，下表按已登记的实际操作顺序整理，未登记的卡点不写作“无失败”。

| 步骤       | 命令 / 操作                                                      | 实测耗时                      | 备注（卡在哪）                                           |
| -------- | ------------------------------------------------------------ | ------------------------- | ------------------------------------------------- |
| 1. 拉代码   | 第 3 节的 `git clone` 命令                                        | A：45 秒；B：1 分钟             | Git 三次失败：域名解析、连接重置、连接超时                            |
| 2. 装依赖   | 创建虚拟环境并安装 `requirements-lock.txt`；A 另执行 `Setup-Realtime.ps1` | A：3 分 57 秒；B：5 分钟         | A 包含教学 Broker 安装；卡点未登记                            |
| 3. 配环境变量 | A 配置 `.env`；B 按第 3 节关闭 MQTT 与外部模型                            | A：45 秒；B：45 秒             | 卡点未登记                                             |
| 4. 启动    | 执行第 3 节所选路线的启动命令                                             | A：30 秒；B：42 秒             | 卡点未登记                                             |
| 5. 打开页面  | 访问 `http://127.0.0.1:8017/`                                  | A：45 秒；B：4 秒              | 早期初始化脚本漏开对话框导致定位超时，修正自动化流程后成功。两次测量不是同一条首次 3D 加载曲线 |
| 6. 初始化数据 | 在页面选择场景，点击“建立对应教学实例”                                         | A：15 秒；B：4 秒              | 数据库由服务自动建表；本步实际发生在打开页面之后                          |
| **合计**   | **逐项加总**                                                     | **A：6 分 57 秒；B：7 分 35 秒** | **本次登记耗时均低于 10 分钟**                                 |

统计范围为拉取源码、安装依赖、配置、启动、页面打开及教学实例初始化；不包括服务账号申请、可选语音模型下载和后来追加的真实 LLM/VLM 调用验证。结论仅适用于本次登记的操作范围与条件。

## 7. 验收清单（跑起来该看到什么）

本次验收范围为 **Web 工作台与 3D 场景**。以下由实际操作者检查；配图是既有隔离教学记录。

- [ ] 启动：首页标题为“校园设施巡检与复核”，可见 3D 校园与五个工作区。[总览](docs/screenshots/stage08/01-campus-overview.png)。
- [ ] 点位：C01 进入照明控制，C07 进入网络交换场景；“进入巡查”打开对应真实详情，返回和 Escape 可用。
- [ ] 任务与方案：建立 S3 并生成判断，仅 T1 受阻；甲可执行，乙丙不可执行。[任务](docs/screenshots/stage08/02-s3-tasks.png)／[方案](docs/screenshots/stage08/03-plan-comparison.png)。
- [ ] 现场补证：从 C07 的业务提醒发起现场复核，提交现场证据与人工确认；查看关联的新 reality，确认旧运行保留。[保存结果](docs/screenshots/stage08/04-review-saved.png)。
- [ ] 条件推演：从 S3 创建 what-if，修改假设后重算方案，原 reality 不变。[推演](docs/screenshots/stage08/05-s4-whatif-a.png)。
- [ ] 可信降级：S5 的 C03 unavailable 时 T2 状态未知、T4 待验证；补证后生成新运行。[原判断](docs/screenshots/stage08/06-s5-unknown.png)／[新判断](docs/screenshots/stage08/07-s5-followup.png)。
- [ ] 告警与提醒：路线 A 新有效远程告警弹窗，首次已有告警只入列表；S2 的 C07 待补检独立提醒，新判断确认缺口消除后移除。仅上传照片、关闭提示、逾期补证或读取失败不消警。[远程告警](docs/screenshots/stage08/08-new-remote-alert.png)。
- [ ] 留存与适配：重启后可查询本次记录与运行；移动端仍可打开详情并提交现场记录。[移动端](docs/screenshots/stage08/09-mobile-alert.png)。

## 8. 排错 FAQ

**Q：笔记本和台式机页面不同。**

原因是版本、实例或缓存不同。核对完整地址和 `git log -1 --oneline`，再刷新资源。用户反馈后，AI 通过分支、运行地址与实际文件定位差异，没有重置数据来复原截图。

**Q：语音提示 network，现场说明没有文字。**

原浏览器在线识别未完成。按第 4 节安装本地 Vosk，允许麦克风并使用 localhost；失败时可继续手写。用户选择本地识别后，AI 将结果接入可编辑草稿，提交前仍需人工核对。

**Q：复制 `.env` 后 AI 配置没有生效。**

启动器不自动加载 `.env`；路线 A 已使用 dotenv 显式加载，编辑后需重启。AI 阅读启动代码后补齐了加载步骤。

**Q：换了场景仍看到 T1 受阻，或远程告警为 0。**

下拉框选择的是待建立场景，需“建立对应教学实例”后再生成判断；旧运行不会自动改变。待补检属于业务提醒，与设备远程告警分开。用户指出歧义后，AI 增加场景提示和独立提醒；补证是否消警仍由新判断决定。

## 9. AI 协同说明

| 项 | 内容 |
| --- | --- |
| 使用的工具 | Codex；历史案例另有 WorkBuddy、DeepSeek 记录 |
| 怎么用的 | 用户明确需求与业务边界，AI 阅读代码、提出方案、实现并检查，用户多轮纠偏 |
| 对话记录 | 日常截图见 `docs/ai-sessions/`；Web 动效、3D 建模、补证消警三个整理案例放在单独的参赛提交材料中 |
| 人工把关 | 区分覆盖、设施状态与复核；禁止 AI 直接判状态、伪造证据或把设计示意当作实景 |

## 10. 目录结构

```text
Campus-Inspection/
├── backend/                 # API、证据、判断、复核与分析
├── frontend/                # Web、Three.js、本地模型与贴图
├── config/                  # 设施、任务、方案及 S1—S5
├── simulators/              # 状态模拟器与 MQTT Broker
├── scripts/                 # 启动、安装与回归入口
├── tests/                   # 规则、接口和交互检查
├── docs/                    # 项目文档、截图与历史对话
├── .env.example             # 服务端配置样例
├── requirements-lock.txt    # 主环境锁定依赖
├── Setup-Realtime.ps1       # 安装实时链路依赖
├── Start-Realtime.ps1       # 启动完整模式
└── Start.ps1                # 启动 Web 服务
```

## 11. 已知限制

| 限制 | 影响 | 处理方式 |
| --- | --- | --- |
| 重新克隆不含本机历史数据 | 其他电脑的现场记录与照片不会自动恢复 | 建立教学实例；真实数据迁移需成套保留数据库与照片 |
| 未做干净机器首次安装计时 | 不能据预算证明 10 分钟复现 | 提交前记录真实机器、过程和耗时 |
| 设施状态为模拟数据，校园为设计示意 | 不能作为真实监测、实景测绘或专业安全结论 | 页面与材料保留边界标识 |
| AI 与本地语音依赖配置或模型 | 外部服务可失败，语音编号和方言需校对 | 可信降级，保留文字与人工复核 |
| 尚无多人身份认证 | 不适合直接作为公开生产服务 | 当前仅用于本地教学演示 |
| 微信小程序不纳入本次验收 | 既有工程保留，不计入完成标准 | 本项目后续不新增小程序功能 |
