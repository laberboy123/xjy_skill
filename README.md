# embedded-skills

面向嵌入式软件工程师的 agent 技能库。C 语言；Linux / RT-Thread / FreeRTOS / 裸机；拿到成品 PCB 后反向点亮器件、按客户需求实现应用、必要时写上位机自测。

八个工作技能 + 两个支撑技能：它不只是"一套流程"，而是**一套会在使用中自己变好、但改动要过审的流程**。

| 技能 | 解决什么 |
|---|---|
| `sensor-bringup` | 点亮器件卡很久：L0–L6 分层、每层 30 分钟时间盒、禁止跳层 |
| `customer-spec` | 需求转化耗时：先要验收定义、十段模板、一轮问完、假设台账 |
| `design-budget` | 不知道 RAM 够不够、任务怎么分：六张表把方案与资源账算平 |
| `board-test` | 缺测试体系：T0–T4 分层、bus_ops 打桩、上位机命令通道 |
| `code-audit` | 合入前不知道查什么：A0–A7 八层审计（内存/栈/优先级/锁/指针/时序） |
| `release` | 发版没底、产线与 OTA 不敢放手：版本可追溯 + 回读确认 + 分批发布与回滚演练 |
| `field-support` | 客户说坏了但说不清：信息包 + 黑匣子 + 四级归因 + 回流 |
| `experience-retro` | 经验/简历转化率低：六格经验卡 → 规则 + 量化简历条目 |
| `capture-learnings` | 不想手动记录：会话结束自动从对话抽条目，零人工 |
| `skill-curator` | 技能如何自己进化：提案、人工批准、定期退役 |

## 目录

```
embedded-skills/
├─ AGENTS.md                    任何 agent 进入本仓库的行为约定（先读这个）
├─ workflow.md                  全流程闭环：门禁表 + 交接契约 + 覆盖度
├─ README.md
├─ CHANGELOG.md
├─ LICENSE
├─ .claude-plugin/              Claude Code 插件与 marketplace 清单
│  ├─ marketplace.json
│  └─ plugin.json
├─ skills/
│  ├─ sensor-bringup/
│  │  ├─ SKILL.md               L0–L6 分层流程
│  │  ├─ bringup-log.md         证据日志
│  │  └─ layered-bringup.md     判据速查 + I2C/SPI 坑位表 + 平台命令
│  ├─ customer-spec/
│  │  ├─ SKILL.md
│  │  ├─ spec.md                十段模板
│  │  └─ change-request.md      变更单
│  ├─ design-budget/
│  │  └─ SKILL.md               预算表 / 优先级表 / 失败模式表
│  ├─ board-test/
│  │  ├─ SKILL.md
│  │  ├─ test-plan.md
│  │  └─ host-protocol.md       上位机 JSON lines 协议
│  ├─ code-audit/
│  │  ├─ SKILL.md               A0–A7 审计流程
│  │  ├─ commands.md            编译告警 / 静态分析 / grep / 运行时测量
│  │  └─ audit-report.md        报告模板 + 优先级表 + 栈水位表
│  ├─ release/
│  │  ├─ SKILL.md               R0–R6 发布流程
│  │  └─ release-checklist.md   发布清单 + 回滚计划 + 分批观测
│  ├─ field-support/
│  │  ├─ SKILL.md               F0–F5 现场流程
│  │  └─ field-packet.md        现场信息包模板
│  ├─ experience-retro/
│  │  ├─ SKILL.md
│  │  └─ experience-card.md
│  ├─ capture-learnings/
│  │  └─ SKILL.md               会话结束自动抽条目，零人工
│  └─ skill-curator/
│     ├─ SKILL.md               四层维护机制
│     └─ proposal.md            提案模板
├─ validation/
│  ├─ kickoff-prompt.md         验证模式提示词（测量用，会停下等确认）
│  ├─ work-prompt.md            工作模式：边干活边记一行日志（日常用这份）
│  ├─ resume-prompt.md          新会话怎么接上（状态快照格式 + 开场提示词）
│  └─ feedback-format.md        反馈字段说明与消费方式
├─ learnings/
│  └─ inbox.example.md          模板；inbox.md 是本机文件，不进仓库
└─ evals/
   └─ cases.md                  15 条回归用例，改动后必跑
```

**每个技能目录自包含**：模板和参考文档都在技能自己的目录里，单独安装某一个技能不会丢文件。

## 怎么让所有 agent 拿到（三种方式）

| 方式 | 命令 | 支持范围 | 更新行为 |
|---|---|---|---|
| **A. skills CLI**（最通用） | `npx skills@latest add laberboy123/xjy_skill` | 70+ 种 agent，按需挑技能 | 复制文件，手动 `npx skills update` |
| **B. Claude Code 插件** | `/plugin marketplace add laberboy123/xjy_skill`<br>`/plugin install embedded-skills@embedded-skills` | 仅 Claude Code | **自动更新** |
| **C. clone + 软链**（推荐自用） | 见下 | 所有 agent，所见即最新 | `git pull` 即更新 |

方式 C 的做法（pstack 也是这么分发的）：

```bash
git clone git@github.com:laberboy123/xjy_skill.git ~/xjy_skill

# Claude Code
mkdir -p ~/.claude/skills
for s in ~/xjy_skill/skills/*/; do
  ln -sfn "$s" ~/.claude/skills/"$(basename "$s")"
done

# Codex / 其他读 ~/.agents/skills 的宿主
mkdir -p ~/.agents/skills
for s in ~/xjy_skill/skills/*/; do
  ln -sfn "$s" ~/.agents/skills/"$(basename "$s")"
done
```

软链之后：**改一次，所有 agent 下次会话都拿到新的**；`git pull` 就是升级。`learnings/` 和 `evals/` 留在仓库里，天然共享（路径解析见 `AGENTS.md`）。

> 注意：不要同时用两种方式装同一套技能，会出现两份、指令互相打架。选一种。

## 仓库信息

- 仓库：`laberboy123/xjy_skill`（SSH：`git@github.com:laberboy123/xjy_skill.git`）
- 插件 / marketplace id：`embedded-skills`（安装时写 `embedded-skills@embedded-skills`）
- 仓库名与插件 id 不同是故意的：仓库以后改名不影响已安装的人。

私有仓库也能用：方式 A/B 需要对应的访问凭据，方式 C 用 SSH 最省事。

## 跑在哪台机器上（重要）

技能是**文件**，必须放在运行 agent 的那台机器上。编译在哪台机器上无关——那是命令，不是文件位置。

| 你的做法 | 技能落在哪 | 公司能看到吗 |
|---|---|---|
| 本地跑 agent，用 ssh 到服务器编译 | 只在你笔记本 | 不能 |
| 本地跑 agent，服务器目录用 SSHFS / 共享盘挂进来 | 只在你笔记本 | 不能（代码在服务器，技能不在） |
| 在服务器上跑 agent（ssh 进去开） | 服务器家目录 | **能**。管理员可读，按"完全可见"假设 |

如果必须在服务器上跑 agent：

- **只装流程类**（`sensor-bringup`、`board-test`），不装会产出个人材料的（`experience-retro`）。
- `learnings/inbox.md` 默认**不进仓库**（见 `.gitignore`）。它只在本机积累，不随仓库走。
- 不要在服务器上把项目内容写进这个仓库再推——**公开的东西撤不回来**。

判断标准一句话：**这个文件如果明天出现在公司群里，你能不能接受。**

> 想让 inbox 跨机器同步？单独建一个 **private** 仓库（例如 `xjy_learnings`）只放 `learnings/`，主仓库继续公开。不要把两者混在一起。

## 它怎么自己变好（核心机制）

```
使用技能  ──►  T0 捕获（capture-learnings 自动读对话，只追加 inbox）
                 │
攒够 5 条 ──►  T1 提案：聚类 → 生成 diff → 写进 inbox 提案区（agent 自动，仍不改）
                 │
                T2 批准：人看 diff + 跑 evals → 合并（人）
                 │
                 └─►  T3 整理：每月去重、抽象、压缩、退役（人主导）
```

三条设计原则：

1. **捕获自动化，判据不自动改**。一次偶发被写进判据，会在之后每次使用中误导人，而且没人会再检查它——比"没有判据"更贵。
2. **两次复现才算信号**。同一类问题在 2 个不同项目/器件上出现，或单次损失 ≥1 天，才够格进提案。
3. **只增不减的技能库会烂掉**。90 天零命中且无证据的判据，提案删除；`evals/cases.md` 保证删改没把好东西弄坏。

## 零人工：会话结束自动沉淀

默认不需要你做任何事。`capture-learnings` 在会话收尾时自动读对话、抽 0–5 条、写进 `learnings/inbox.md`，只在回复末尾给你一行摘要。

触发不需要你提醒：说"搞定 / 收工 / 今天到这"、刚跑完一次"故障 → 修复"、你两次纠正同一类做法、或长会话自然结束，都会触发。

它**不做**的事：不改判据、不生成提案、不写客户名和项目代号、一次不超过 5 条、对话里没有证据就不写。

历史坑不用补——从今天起自动积累，攒够量自然会提议提升。这也意味着**越用越准**：前两周条目少是正常的。

## 首次上手

1. **跑一遍真实任务**：挑一颗最近要点的传感器，用 `sensor-bringup` 走一遍，让它维护 `bringup-log.md`。
2. **收工时不用管**：结束后它自己记条目，你偶尔看一眼 inbox 就行。
3. **攒够 5 条**：让它生成提案，你只审 diff（`skill-curator` 的 T2）。

## 想更保险：每日定时回顾

自动捕获依赖两件事：会话还在（上下文没被清），以及模型遵循 `AGENTS.md`。如果想让"每天必有一次回顾"成为硬保证，可以加一条每日定时任务，让 agent 读当天的会话记录补录遗漏的条目。

- DSH：可直接建定时任务（每天固定时间跑同一段指令）。
- Claude Code / 其它：`cron` + `claude -p "<同一段指令>"`。

代价是每天一次模型调用。**不是必需的**，只在"你这周根本没说过'收工'"这种场景下才有价值。

## 与那三个开源仓库的关系

| 来源 | 借鉴 | 未采用 |
|---|---|---|
| mattpocock/skills | 追问原语、共享词典、垂直切片 TDD、分阶段诊断、"改环境不改结论"的复盘 | issue/triage/GitHub 流程 |
| addyosmani/agent-skills | `SKILL.md` 五段结构（Process / Rationalizations / Red Flags / Verification）、Verification 作硬出口、每技能带 eval | 6 阶段 10 命令 persona 流水线 |
| michael-denyer/pstack-claude | 证据纪律、"写代码的不能自己判过"（本库把它推广到"改判据的不能是产生判据的那次实验"） | 多模型 arena/interrogate、gh PR 自动化、autopilot |

结构来自 agent-skills，纪律来自 pstack，追问来自 mattpocock，领域判据来自你自己的项目。

## 维护原则

- **只改判据，不改结构**。骨架稳定，agent 的遵循率才稳。
- **每次踩坑都回写，但先落 inbox**。这是唯一的复利来源。
- **判据必须可观测**。"数据看起来正常"不是判据。
- **不要把客户名、协议细节、凭据写进 `learnings/`**。只写可复用的技术判据。
- **不要让 agent 替你签字**。客户确认、验收结论、安全判断必须是人给的。
