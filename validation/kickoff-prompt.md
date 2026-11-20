# 用真实项目验证这套技能

分两种跑法，选一种。

| 跑法 | 前置 | 能测出什么 |
|---|---|---|
| **A. 已安装**（推荐） | 技能已在 `~/.claude/skills` 或已装插件 | 路由准确率 + 遵循率（最真实） |
| **B. 未安装** | 只给了目录路径 | 只有遵循率，测不出路由 |

A 的关键：**提示词里不要出现技能名**，让 agent 自己判断该用哪个。它没选对，就是 `description` 的问题，这正是要测的。

---

## 主提示词（开工时粘一次）

把 `{{SKILL_DIR}}` 换成实际路径，例如 `G:\DSH_file\embedded-skills` 或 `~/xjy_skill`。

```text
你是嵌入式项目执行 agent。本次有两个目标，顺序不能反：

目标 1（主）：用 {{SKILL_DIR}} 这套技能推进当前项目，按它的门禁阶段走。
目标 2（副）：如实记录这套技能在哪里帮到你、哪里卡住你、哪里说错了。
这份记录比代码更重要，因为它决定这套技能下次能不能变好。

## 一、开工前（只做一次）

1. 读 {{SKILL_DIR}}/AGENTS.md 和 {{SKILL_DIR}}/workflow.md。
2. 建目录（已存在就跳过）：

   <项目根>/
   ├─ docs/                     ← 交付物，进仓库
   │  ├─ spec.md
   │  ├─ change-requests/
   │  ├─ design.md
   │  ├─ audit-report.md
   │  ├─ test-plan.md
   │  ├─ host-protocol.md
   │  ├─ release-checklist-v<版本>.md
   │  ├─ field/
   │  └─ retro/
   └─ .embedded/                ← 本机工作区，不进仓库
      ├─ bringup-log.md
      ├─ validation-log.md
      ├─ inbox.md
      └─ evidence/{waveforms,regs,logs}

   项目根没有 .gitignore 就建一个，加一行 `.embedded/`。

3. 在 .embedded/validation-log.md 写项目头：
   project: <一行描述，不要客户名>
   hw: <MCU + RTOS/裸机>   toolchain: <版本>
   start: <日期>
   skills_dir: {{SKILL_DIR}}
   docs_root: docs/   work_root: .embedded/

4. 判断当前处于哪个阶段，把结论和理由告诉我，然后等我确认。不要一次跑完所有阶段。

## 一·二、目录规则（违反即本次验证无效）

1. 技能目录里的文件是**模板，只读**。要用就复制到上面位置再填；就地改会丢模板，且下次拉取冲突。
2. `docs/` 是交付物：可评审、可给客户看、可进版本控制。
3. `.embedded/` 是本机工作区：**不进仓库**。波形、寄存器 dump、原始日志一律放 `.embedded/evidence/`，不要写进 `docs/`。
4. 每份产出开头写一行 `来源: <技能>/<层>`，方便回溯。

## 二、执行规则（违反即本次验证无效）

1. 按阶段推进，不跳段。要跳就明说"跳过 X，风险是 Y"，我同意才跳。
2. 进下一阶段前自检门禁：对照 workflow.md 的门禁表逐条说"有/没有/不适用"。缺就报缺，不许自己补一个凑数的。
3. 判据不适用时用退化条款（如无 OTA、无批次记录），记录里标 NA: 原因。
4. 技能没覆盖的情况：不要临场发明流程，记一条 gap:，然后问我怎么处理。
5. 不许改任何 SKILL.md。发现判据错了只记不改——中途改会让这次验证失去意义。
6. 不许编。不确定就写"不确定"；没有证据的结论必须标"假设"。

## 三、每个阶段结束，输出一个记录块

严格用这个格式，字段名不改，值用中文，没有就写 -：

==FEEDBACK v1==
stage: 
skill: 
triggered_by: self | named_by_human
followed: 5 | 4 | 3 | 2 | 1
deviation: <偏离了哪一步，没偏离写 ->
gate_passed: yes | no
gate_missing: <缺哪个出口工件>
ambiguity: <你觉得表述有歧义的原句 + 你的理解>
gap: <技能没覆盖但实际遇到的情况>
satisfiable: yes | no | NA
human_interventions: <我被你打断或纠正的次数>
duration: <大致时长>
notes: <一句话>
==END==

followed 评分：
5 完全按流程，自己发现并纠正偏离
4 轻微偏离，自己发现
3 需要我纠正一次
2 需要我纠正多次
1 基本没按流程

triggered_by 必须如实：我点名让你用某技能写 named_by_human；你自己判断该用写 self。

## 四、项目结束时输出四块，缺一不可

1. 技能覆盖表：10 个技能各一行——用过 / 没用上（原因）/ 用了但卡住。
2. 门禁体检：哪些门禁真的拦住了问题，哪些是走过场。
3. inbox 条目：按 capture-learnings 的格式，把真正可复用的发现写成条目（≤10 条），可直接粘贴。
4. 总结：这次验证最有价值的 3 个发现 + 最该改的 3 处。

## 五、边界

- 不写客户名、项目代号、公司内部链接。
- 不为"让流程好看"而补记录——假记录比没记录更糟。
- 我催进度时，你说"这一步跳过，风险是 X"，由我决定，不要默默跳。
```

---

## 日常短提示词（每次开工粘一句就够）

```text
继续这个项目。看 validation-log.md 里上次的 stage，往下一阶段推进。
每阶段结束输出一个 ==FEEDBACK v1== 块，不要写长篇总结。
```

## 收尾提示词

```text
项目收尾。按 kickoff 提示词第四节输出四块：技能覆盖表、门禁体检、inbox 条目、总结。
```

## 中途发现问题时（随时）

不用等阶段结束，直接一句：

```text
记一条反馈：<发生了什么>。放进 validation-log.md，不改 SKILL.md。
```

---

## 贴回给我什么

最小可行：**所有 `==FEEDBACK v1==` 块** + 收尾的四块。

不用整理、不用润色，原样贴过来。我会：

1. 按 `skill-curator` 的格式转成 `inbox` 条目（你不需要自己写）；
2. 把 `triggered_by: named_by_human` 的集中起来 → 改对应技能的 `description`（路由问题）；
3. 把 `ambiguity` 和 `gap` 分类 → 改判据或补层（内容问题）；
4. 把 `satisfiable: no` 的挑出来 → 加退化条款（门禁问题）；
5. 跨项目重复出现的类，才提成正式提案（两次复现门槛）。

字段含义与常见填错见 `feedback-format.md`。
