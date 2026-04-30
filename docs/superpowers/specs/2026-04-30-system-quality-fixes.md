# 灵石庄测试发现的系统级修复 Spec

> 日期：2026-04-30
> 范围：webnovel-writer 系统层面问题（与 LLM 模型能力无关）
> 测试基础：DeepSeek v4pro 在 `D:\wk\xiaoshuo\灵石庄` 上跑完 init / plan / write 第 1-2 章 + 审查
> 事实校验：本版已对照当前 repo 代码行为与 `D:\wk\xiaoshuo\灵石庄` 实际 artifacts 复核，修正了过度归因和不存在的落点
> 已交付：commit `f58d657 fix: align projection writers with real LLM commit schema`（schema 漂移系列）

---

## 1. 背景与范围

灵石庄项目用 DeepSeek v4pro 跑通 init→plan→write→review→commit 完整链路两章后，对生成产物做端到端审视。本 spec 列出**纯系统侧**的 8 个未修问题（schema 漂移已在 f58d657 修完），按优先级 P0→P3 排序。每条包含：现状证据、影响、根因、修复方案（具体文件 + 行号）、验收标准、工作量估计。

不在本 spec 范围内：
- LLM 输出本身的写作质量问题（AI 味、排比句、习惯动作过密等）—— 由审查 agent 本职拦截
- 已在 f58d657 中修复的 schema 漂移问题（state_projection / memory_writer / vector_writer / index_manager schema、protagonist_state 镜像、index 主角识别、entity 别名兜底）
- 已在历史 commit 修过的（init 项目根 9b8976e、entity alias fallback fc63628、安全写入项目经验记忆 d2571d2 等）

---

## 2. 执行计划

本节把下方 8 个已验证问题重新编排成可直接实现的工作流。问题本身仍以下方详细 spec 为事实源；这里负责明确顺序、依赖和落点，避免实现时在 prompt、runtime brief、init 模板、review 回流之间来回跳。

### Phase 1 — chapter directive / chapter_focus / 写章 prompt 排序（Issues #1 #2 #5）

**Objective**：让详细大纲里的章节执行约束进入 `chapter_brief` 顶层，并在写章任务书中压过 `dynamic_context`；同时切断 `chapter_focus` 从 reference summary 继承的误导链路。

**Files to change**：
- `webnovel-writer/scripts/chapter_outline_loader.py`
- `webnovel-writer/scripts/story_system.py`
- `webnovel-writer/scripts/data_modules/story_system_engine.py`
- `webnovel-writer/scripts/data_modules/runtime_contract_builder.py`（确认不混写 directive 与 review/runtime contract）
- `webnovel-writer/skills/webnovel-write/SKILL.md`
- 相关测试文件（优先沿用现有 story-system / outline loader 测试目录）

**Concrete tasks**：
- 在 `chapter_outline_loader.py` 增加或扩展 `load_chapter_execution_directive(project_root: Path, chapter: int)`，从 `大纲/第{vol}卷-详细大纲.md` 提取 `goal`、`obstacles`、`cost`、`time_anchor`、`chapter_span`、`countdown`、`cbn`、`cpns`、`cen`、`must_cover_nodes`、`forbidden_zones`、`chapter_end_open_question`、`hook_type`、`hook_strength`、`key_entities`、`strand`、`antagonist_tier`。
- 在 story-system brief 组装链路中写入顶层 `chapter_directive`；找不到章节块时保持老路径兼容。
- 修改 `_suggest_chapter_focus()`：优先使用 `chapter_directive.goal`；没有 directive 时使用非占位 query；禁止从 `dynamic_context[0]["核心摘要"]` 派生。
- 调整 `webnovel-write/SKILL.md` 任务书顺序：`本章硬性约束` → `CBN/CPNs/CEN` → `本章禁区` → `风格指引` → `dynamic_context` 风格参考。
- 增加测试覆盖：directive 字段落盘、`chapter_focus` 来源、无 directive 时不抄 reference summary、prompt 顺序人工 fixture。

**Dependencies**：
- 依赖现有 `chapter_outline_loader.py` 已能识别 `CBN / CPNs / CEN / mandatory_nodes / prohibitions`。
- Phase 2 的 reference ranking 会复用本阶段产出的 `chapter_directive.goal/key_entities/strand/antagonist_tier`，所以本阶段必须先做。

**Verification/tests**：
- `pytest --no-cov` 中新增的 outline/story-system 相关测试通过。
- 抽样生成第 1 章 brief，确认 `chapter_directive.time_anchor`、`must_cover_nodes`、`forbidden_zones` 存在。
- 人工检查写章任务书，确认 `dynamic_context` 被标注为补充参考且排在 directive 后。

**Estimated scope/risk**：中，约 4-6 小时。主要风险是详细大纲 markdown 标题/字段格式变体导致解析漏抓；需要正则兼容 `第1章`、`第一章`、空格变体。

### Phase 2 — query truthiness / reference chapter-aware ranking（Issue #3）

**Objective**：保证 story-system 收到真实章节目标 query，并在 reference 检索排序中放大章节语义，避免“题材对、场景错”的卡片压过借贷/调查类卡片。

**Files to change**：
- `webnovel-writer/scripts/story_system.py`
- `webnovel-writer/scripts/data_modules/story_system_engine.py`
- `webnovel-writer/skills/webnovel-plan/SKILL.md`
- `webnovel-writer/skills/webnovel-write/SKILL.md`
- `webnovel-writer/skills/webnovel-review/SKILL.md`
- reference/story-system 相关测试 fixture

**Concrete tasks**：
- 在 `story_system.py` CLI 入口增加 placeholder query 保护：识别 `{章纲目标}`、`第N章章纲目标`、空泛模板文本，输出诊断或降级警告。
- 更新 plan/write skill：调用 story-system 前必须解析真实章目标；不能把占位字符串传入 `query`。
- 更新 review skill：审查 brief 时对照 `chapter_brief.meta.query` 与详细大纲目标，把 placeholder query 标成系统问题。
- 在 `story_system_engine.py` 的 query 扩展/表收集/排序附近加入 chapter-aware scoring，输入来自 `chapter_directive.goal/key_entities/strand/antagonist_tier`。
- 将最终排序调整为题材匹配 + 章节关键词命中的组合分；同优先级下章节关键词命中更多者靠前。
- 保留 `reference_search.py` 通用能力，不新增不存在的路由层。

**Dependencies**：
- 依赖 Phase 1 提供 `chapter_directive`。如果 Phase 2 先做，只能先使用 query 文本，排序收益会明显降低。
- 需要一份包含金融/借据/利息关键词的 reference fixture，才能证明第 1 章场景召回改善。

**Verification/tests**：
- `test_story_system_rejects_or_warns_placeholder_query`：占位 query 被诊断。
- `test_story_system_reference_matching_prefers_chapter_keywords`：借贷类卡片排在论道/丹药/宗门经营泛卡前。
- 用灵石庄第 1 章 brief 抽样验证：`dynamic_context` 至少有一条与“借据/利息/复利/债”相关。

**Estimated scope/risk**：中，约 4 小时。主要风险是 reference CSV 字段不稳定，需要对 `关键词`、`意图与同义词`、`适用场景` 缺字段做容错。

### Phase 3 — init template pruning / conditional generation（Issue #4）

**Objective**：减少 init 后空壳设定文件和空目录，避免单主角项目生成 `主角组.md`，并把事实源集中到已有主角卡、世界观、卷纲等文件。

**Files to change**：
- `webnovel-writer/scripts/init_project.py`
- `webnovel-writer/templates/output/设定集-金手指.md`
- `webnovel-writer/templates/output/复合题材-融合逻辑.md`
- `webnovel-writer/templates/output/设定集-主角组.md`
- `webnovel-writer/templates/output/设定集-女主卡.md`
- `webnovel-writer/skills/webnovel-init/SKILL.md`
- init 相关测试

**Concrete tasks**：
- 从默认生成列表中移除被其它文件覆盖的空壳模板：金手指、爽点规划、复合题材融合逻辑（除非项目显式需要）。
- 按 `idea_bank.protagonist_structure` 条件生成 `主角组.md`：仅 `主角组`、`双主角`、`多主角` 类项目生成。
- 按 `heroine_config` 条件生成 `女主卡.md`：`无女主` 时不生成。
- `角色库/`、`物品库/`、`其他设定/` 改为第一次实体增量写入时再创建。
- 更新 init skill 文档，避免 prompt 继续要求创建已剪掉的模板。

**Dependencies**：
- 与 Phase 1/2 无代码依赖，可并行实现。
- 需要先搜索 `init_project.py`、`webnovel-init/SKILL.md`、模板路径中的生成清单，避免只删模板不改调用方。

**Verification/tests**：
- 单主角 fixture 下不生成 `设定集/主角组.md`，仍生成 `设定集/主角卡.md`。
- `无女主` fixture 下不生成 `设定集/女主卡.md`。
- 默认 init 后不生成 `金手指设计.md`、`爽点规划.md`、空 `角色库/物品库/其他设定` 目录。

**Estimated scope/risk**：小，约 2 小时。主要风险是模板文件名和输出文件名不完全一致，需要以 `init_project.py` 的真实映射为准。

### Phase 4 — placeholder guardrails / cross-volume anchor writeback（Issues #6 #7）

**Objective**：在 plan/write 边界发现 `[待...]`、`暂名`、`{占位}` 等漂移；同时把 Phase 7 定义为当前卷规划完成后的最小跨卷锚点写回：只在 `webnovel-plan` 已完成并验证当前卷规划 artifacts 后，向 `总纲.md` 写回 V+1 的卷名、核心冲突、卷末高潮，以及当前规划产物显式结构化给出的伏笔/开放环。Phase 7 不自动详细规划下一卷。

**Files to change**：
- `webnovel-writer/scripts/data_modules/placeholder_scanner.py`（新增）
- `webnovel-writer/scripts/webnovel.py`
- `webnovel-writer/skills/webnovel-plan/SKILL.md`
- `webnovel-writer/skills/webnovel-write/SKILL.md`
- `webnovel-writer/templates/output/大纲-总纲.md`
- plan/write/CLI 相关测试

**Concrete tasks**：
- 新增 `scan_placeholders(project_root)`，扫描 `大纲/*.md`、`设定集/*.md` 中的 `\[待[^\]]*\]`、`（暂名）`、`(暂名)`、`（待补充）`、全字段 `{占位}` / `<占位>`。
- 在 `webnovel.py` 增加 `placeholder-scan --project-root <dir>` CLI。
- 在 plan skill 开始/结束处加入占位扫描：plan 阶段警告但不阻断。
- 在 write skill 开始处加入章节相关实体的占位阻断：当前章涉及实体的设定文件仍有 pending 标记时阻断。
- 调整 `大纲-总纲.md` 模板：init 不预生成 V2-V20 空行。
- 在 `webnovel-plan` 成功完成当前卷规划且 artifacts 已完整/通过验证后，向 `总纲.md` 写回 V+1 的最小跨卷锚点：卷名、核心冲突、卷末高潮。
- 伏笔表只追加当前规划输出中显式结构化产出的新伏笔/持续开放环；禁止从自由文本推断伏笔。
- Phase 7 禁止生成下一卷详细大纲、beat sheet、timeline 或章节级规划。

**Dependencies**：
- 与 Phase 3 有模板交叉：如果两者并行，必须协调 `templates/output/大纲-总纲.md` 与 init 生成逻辑。
- write 阻断要复用 Phase 1 的 `chapter_directive.key_entities` 或现有章节实体解析结果，否则只能做全项目扫描，误报风险较高。

**Verification/tests**：
- `test_placeholder_scanner_finds_pending_marks` 覆盖 `[待...]`、`暂名`、`{占位}`。
- `test_write_chapter_blocks_on_pending_related_entity`：本章涉及实体有占位则阻断，不涉及则不阻断。
- `test_plan_flow_writes_minimal_next_volume_anchor`：完成并验证 V1 规划后只补 V2 的卷名、核心冲突、卷末高潮，不预填 V3-V20 空表，也不生成 V2 详细大纲/beat sheet/timeline。
- CLI smoke test：`webnovel.py placeholder-scan --project-root <fixture>` 返回结构化结果。

**Estimated scope/risk**：中，约 6 小时。主要风险是占位误报和“当前章节涉及实体”判定不准；应优先做警告/阻断分级。

### Phase 5 — anti-pattern feedback loop（Issue #8）

**Objective**：把 review 抓到的中高严重度 `ai_flavor` 问题回流到 `.story-system/anti_patterns.json`，让后续写章 brief 主动避开已发现句式。

**Files to change**：
- `webnovel-writer/scripts/data_modules/review_schema.py`
- `webnovel-writer/scripts/data_modules/story_system_engine.py`
- `webnovel-writer/skills/webnovel-review/SKILL.md`
- `webnovel-writer/skills/webnovel-write/SKILL.md`
- review/story-system 相关测试

**Concrete tasks**：
- 在审查报告持久化后增加 hook：`category == "ai_flavor"` 且 `severity in {"medium", "high", "critical"}` 时追加 anti-pattern。
- anti-pattern 记录包含 `text`、`source_table="review_extracted"`、`source_id="ch0002_issue_N"`、`category`、`added_at`。
- 增加去重：同一 evidence/text 不重复写入。
- brief 构建时读取 review-extracted anti-pattern，并在写章任务书中作为“避雷模式”输入。
- 更新 review/write skill 文档，明确哪些审查发现会进入后续写章约束。

**Dependencies**：
- 与 Phase 1 无硬依赖，但 Phase 1 的 prompt 排序完成后，anti-pattern 更容易被放到正确位置。
- 需要确认现有 `.story-system/anti_patterns.json` schema，不要破坏 reference 初始化写入的旧字段。

**Verification/tests**：
- `test_ai_flavor_review_issue_added_to_anti_patterns`：中等以上 ai_flavor 被追加。
- `test_anti_pattern_review_feedback_dedupes_evidence`：重复 evidence 不重复入库。
- brief fixture 验证 review-extracted text 能出现在后续写章避雷列表。

**Estimated scope/risk**：小，约 2 小时。主要风险是 review report 保存入口不唯一，需要先定位所有审查持久化路径。

---

## 3. 待办问题清单（速览）

| # | 优先级 | 问题 | 影响面 | 工作量 |
|---|--------|------|--------|--------|
| 1 | **P0** | chapter_brief 缺少详细大纲的丰富执行约束 | 每一章质量 | 中 |
| 2 | **P0** | 写章 prompt 中 dynamic_context 占主位、大纲约束次要 | 每一章质量 | 小 |
| 3 | P1 | reference 检索对真实章目标感知不足，导致题材路由压过章节语义 | 每一章 brief 相关性 | 中 |
| 4 | P1 | 设定模板批量生成空壳；单主角项目仍生成主角组文件 | init 后用户混淆、占位 noise | 小 |
| 5 | P1 | `chapter_focus` 字段被错塞 dynamic_context 里的无关 summary | brief 误导 | 小 |
| 6 | P2 | 设定文件缺少占位/漂移防护（卷纲 `[待...]` 占位 vs 主角卡已具名） | plan 阶段不一致 | 小 |
| 7 | P2 | 当前卷完成后缺少 V+1 最小锚点 / 结构化伏笔写回 | 跨卷承诺易脱节 | 中 |
| 8 | P2 | 审查抓到的 ai_flavor 模式不回流 `anti_patterns.json` | 同类问题反复出现 | 小 |

---

## 4. 详细修复 Spec

### Issue #1 — chapter_brief 缺少详细大纲的丰富执行约束（P0）

**现状证据**：

`D:\wk\xiaoshuo\灵石庄\.story-system\chapters\chapter_001.json`（136 行）字段构成：

```json
{
  "meta": {...},
  "override_allowed": {"chapter_focus": "<从 SP-087 卡片抄来>"},
  "dynamic_context": [SP-087, NR-077, SY-009, CH-072],
  "source_trace": [...],
  "reasoning": {...}
}
```

`D:\wk\xiaoshuo\灵石庄\大纲\第1卷-详细大纲.md` 的第 1 章节有完整的：目标 / 阻力 / 代价 / 时间锚点 / 倒计时状态 / Strand / 反派层级 / **CBN / CPNs / CEN / 必须覆盖节点 / 本章禁区 / 章末未闭合问题 / 钩子类型**。

其中 `CBN / CPNs / CEN / 必须覆盖节点 / 本章禁区` 已能被下游结构化解析器识别，但 **chapter brief 本身没有携带目标 / 阻力 / 代价 / 时间锚点 / 倒计时 / 章末未闭合问题等执行约束**。

**影响**：

- 第 2 章审查报告抓到的"孙旺说'明儿个' vs 时间线表 D-Day 傍晚"矛盾 —— 写作前没有把时间锚点、章内跨度、倒计时状态放进主任务书
- "本章禁区"可进入 review contract 的阻断规则，但没有作为写作前的正向执行约束展示，仍偏事后发现
- fulfillment_result.json 是事后履约校验，等正文写完才发现节点缺失

**根因**：

当前链路是分裂的：

- `webnovel-writer/scripts/chapter_outline_loader.py` 已能从详细大纲中提取 `cbn / cpns / cen / mandatory_nodes / prohibitions`
- `webnovel-writer/scripts/data_modules/runtime_contract_builder.py` 也已经消费了这些结构化字段，用于 `selected_scenes / must_check / blocking_rules`
- 但 `CHAPTER_BRIEF` 本身仍主要来自 story-system 的 `dynamic_context + chapter_focus`，没有把章节执行约束字段提升为写作主输入

问题不是“完全没读详细大纲”，而是**只读了结构化骨架，没有把章节执行指令注入 chapter brief 主链**。

**修复方案**：

1. 在现有 `webnovel-writer/scripts/chapter_outline_loader.py` 能力之上补一层执行约束提取，优先扩展同一模块或新增相邻 helper：
   - `load_chapter_execution_directive(project_root: Path, chapter: int) -> dict`
   - 从 `大纲/第{vol}卷-详细大纲.md` 解析章节 markdown 块，提取字段：
     - `goal` / `obstacles` / `cost` / `time_anchor` / `chapter_span` / `countdown`
     - `cbn` / `cpns: list[str]` / `cen`
     - `must_cover_nodes: list[str]`（"必须覆盖节点"行）
     - `forbidden_zones: list[str]`（"本章禁区"行，分号分隔）
     - `chapter_end_open_question` / `hook_type` / `hook_strength`
     - `key_entities: list[str]`、`strand`、`antagonist_tier`
   - 解析容错：找不到本章块返回 None，让上层走老路径

2. 修改 story-system 持久化的 chapter brief 组装链路（`webnovel-writer/scripts/story_system.py` + `webnovel-writer/scripts/data_modules/story_system_engine.py`，并让 `runtime_contract_builder.py` 继续负责 volume/review runtime contracts）：
   ```python
   from chapter_outline_loader import load_chapter_execution_directive
   directive = load_chapter_execution_directive(project_root, chapter)
   brief["chapter_directive"] = directive  # 新增顶层字段
   if directive and directive.get("goal"):
       # 用大纲目标覆盖从 dynamic_context 抄来的 chapter_focus
       brief["override_allowed"]["chapter_focus"] = directive["goal"]
   ```

3. 修改写章 skill (`webnovel-writer/skills/webnovel-write/SKILL.md`) 中 prompt 模板，把 `chapter_directive` 摆在 dynamic_context **之前**，并加显式约束语：
   ```markdown
   ## 本章硬性约束（优先级最高）
   - 时间锚点：{directive.time_anchor}
   - 倒计时状态：{directive.countdown}
   - 必须覆盖节点：{directive.must_cover_nodes}
   - 章末必须留下：{directive.chapter_end_open_question}
   - 本章禁区（违反即不通过）：{directive.forbidden_zones}
   ```

**验收标准**：

```python
def test_chapter_brief_contains_outline_directive(tmp_path):
    # 准备测试用大纲文件，写一章节块
    outline = tmp_path / "大纲" / "第1卷-详细大纲.md"
    outline.parent.mkdir(parents=True)
    outline.write_text(SAMPLE_OUTLINE_CHAPTER_1, encoding="utf-8")
    # 跑 brief 构建
    brief = build_chapter_brief(tmp_path, chapter=1)
    # 关键字段必须存在
    assert brief["chapter_directive"]["time_anchor"] == "D-Day 清晨"
    assert "杂役不能随意离开宗门" in brief["chapter_directive"]["forbidden_zones"][0]
    assert len(brief["chapter_directive"]["cpns"]) >= 1
    # chapter_focus 来自大纲目标，不是 SP 卡 summary
    assert "搞清楚" in brief["override_allowed"]["chapter_focus"]
```

**工作量**：中（约 4-6 小时含测试）

---

### Issue #2 — 写章 prompt 中 dynamic_context 占主位（P0）

**现状证据**：

`webnovel-writer/skills/webnovel-write/SKILL.md` 现在已经声明 `chapter_focus` 仅为 CSV 参考、本章目标以章纲为准，并要求有结构化节点时围绕 `CBN→CPNs→CEN` 展开。但任务书生成仍没有一个稳定的 `chapter_directive` 展示模板，`dynamic_context` 这类 reference 补充容易在实际任务书里比大纲执行约束更显眼。

**影响**：与 #1 联动 —— 即使 brief 里加了 directive，如果 context-agent / 写章任务书没有明确排序，LLM 仍可能把 generic reference 当成主要写作方向。

**修复方案**：

修改 `webnovel-writer/skills/webnovel-write/SKILL.md` prompt 段落顺序：

1. 标题：`# 第N章 写作任务`
2. **本章硬性约束**（来自 chapter_directive，新增）—— 列点形式
3. **本章必须覆盖节点**（CBN/CPNs/CEN）—— 编号列表
4. **本章禁区**（禁区项目）—— 显眼提示
5. **风格指引**（来自 reasoning + 主角卡 OOC 警戒，从 master setting 抽取）
6. **场景写法补充**（dynamic_context，**仅作风格参考**，明确标注非强制）

**验收标准**：

人工 review 一次写章 prompt，确认 directive > dynamic_context 顺序与权重。

**工作量**：小（约 1 小时，纯改 markdown 模板 + 1 个 fixture 测试）

---

### Issue #3 — reference 检索对真实章目标感知不足（P1）

**现状证据**：

第 1 章 brief 注入：
- SP-087 论道场景 → 第 1 章是杂役通铺醒来，无关
- NR-077 丹药法宝命名 → 第 1 章无丹药法宝
- SY-009 宗门经营 → 第 1 章不涉及组织经营
- CH-072 宗门天骄 → 第 1 章无天骄出场

当前产物表现为：被注入的卡片高度偏向“仙侠通用写法”，与第 1-2 章的借贷调查场景不对位。

**影响**：每章 brief 中无关知识占位，挤走真正有用的卡片（金融/经济类 reference 完全没出现），同时 `chapter_focus` 还会被这些误召回卡片继续放大偏差。

**根因**：

这是两个问题叠加：

1. **上游章目标注入不实**：现有 `webnovel-plan` / `webnovel-write` skill 文档把 story-system 调用写成 `story-system "{章纲目标}" ...`。灵石庄实际落盘的 `MASTER_SETTING.json` 里，`query` 甚至是字面量 `"第2章章纲目标"`，说明运行链路并未稳定传入真实章目标；`webnovel-review` 则在后续审查中暴露这种弱 query 带来的偏差。
2. **下游检索仍偏题材路由**：`webnovel-writer/scripts/data_modules/story_system_engine.py` 虽然会把 `query` 传给 `reference_search.py`，但检索入口先由题材路由决定推荐表，再做简化 BM25。上游 query 一旦是占位或泛词，结果就会退化为“题材对了、场景不对”。

所以根因不是“只按 genre_filter 检索”，而是**真实章目标没有稳定进入 query，且现有排序对章节语义的放大不够**。

**修复方案**：

1. 先修上游 query 真值：
   - 在 `webnovel-writer/skills/webnovel-plan/SKILL.md`、`webnovel-writer/skills/webnovel-write/SKILL.md` 中，把 `story-system "{章纲目标}" ...` 从占位提示升级为**必须先解析真实章目标再调用**
   - 在 `webnovel-writer/skills/webnovel-review/SKILL.md` 的审查输入说明里强调需对照真实 `chapter_brief.meta.query` / 大纲目标，避免把 placeholder query 产物当正常 brief
   - 在 `webnovel-writer/scripts/story_system.py` 入口附近补一层保护：若 query 仍是 `{章纲目标}`、`第N章章纲目标` 这类占位文本，给出诊断或降级警告
2. 在 reference 检索阶段加入 chapter-aware 信号：
   - 输入：`chapter_directive.goal`、`chapter_directive.key_entities`、`chapter_directive.strand`、`chapter_directive.antagonist_tier`
   - 提取关键词（jieba 分词或简单 split）
   - 与 reference 卡片的 `关键词` / `意图与同义词` / `适用场景` 字段做 token overlap 评分
3. 在 `webnovel-writer/scripts/data_modules/story_system_engine.py` 的 `_collect_tables` / `_expand_query` 周边补排序信号：
   - 最终排序：`(genre_match_score * 0.4) + (chapter_keyword_score * 0.6)`
   - 同 `priority_rank` 的卡片优先取章节关键词命中数高的
4. 保留 `reference_search.py` 的通用 BM25 能力，不另造不存在的 `genre_router.py`

**验收标准**：

```python
def test_story_system_reference_matching_prefers_chapter_keywords():
    directive = {"goal": "看穿借据条款的荒谬", "key_entities": ["借据", "利息", "复利"]}
    result = StorySystemEngine(csv_dir=fixture_csv_dir).build(
        query="看穿借据条款的荒谬 借据 利息 复利",
        genre="仙侠",
        chapter=1,
    )
    selected = result["chapter_brief"]["dynamic_context"]
    # 借贷类卡片应排在论道类前面
    assert selected[0]["编号"] == "FIN-001"
```

**工作量**：中（约 4 小时，含一份金融场景 reference 卡片样例）

---

### Issue #4 — 设定模板空壳 + 单主角误生成主角组（P1）

**现状证据**：

灵石庄设定集：
- `金手指设计.md`：全是 `{占位}` 和空字段（信息已在主角卡 §金手指 + 总纲 §创意约束）
- `复合题材-融合逻辑.md`：全空（信息已在世界观 §融合逻辑 + 总纲 §复合题材融合）
- `主角组.md`：全空，**且 `state.protagonist_structure="单主角+辅助视角"`**
- `爽点规划.md`：只有 1-5/6-10 两条示例（卷纲 §爽点密度规划已有更详细的）
- `角色库/{次要角色,反派角色,主要角色}/`、`物品库/`、`其他设定/`：全是空目录

**影响**：

- 用户 init 后看一堆空文件不知是该填还是不该填
- brief 里的 master setting 提取器若无脑读所有设定文件会误解空字段为"未确定"
- 项目目录显得冗余

**根因**：

`webnovel-writer/skills/webnovel-init/SKILL.md` 里硬编码生成所有模板，没按 `idea_bank` 的 `protagonist_structure / heroine_config / 复合题材标志` 做分支。

**修复方案**：

1. **删除以下"已被其他文件覆盖"的模板**（不再生成）：
   - `金手指设计.md` —— 信息在主角卡 §金手指
   - `复合题材-融合逻辑.md` —— 信息在世界观 §复合题材融合（且仅复合题材项目才有意义）
   - `爽点规划.md` —— 卷纲爽点表是单一事实源
2. **按项目配置条件生成**：
   - `主角组.md`：仅当 `protagonist_structure ∈ {"主角组", "双主角", "多主角"}` 时生成
   - `女主卡.md`：仅当 `heroine_config != "无女主"` 时生成
   - `反派设计.md` 始终生成（必填）
   - `复合题材-融合逻辑.md`（如果保留）：仅当 `genre_combination` 标记非空时生成
3. **空目录改为按需创建**：
   - `角色库/` 在第一次 `apply_entity_delta` 写新角色时创建（按 entity_type/tier 分子目录）
   - `物品库/`、`其他设定/` 同理

修改文件：`webnovel-writer/scripts/init_project.py`、`webnovel-writer/templates/output/` 中仍需保留的模板，以及 `webnovel-writer/skills/webnovel-init/SKILL.md` 生成步骤段。

**验收标准**：

```python
def test_init_skips_protagonist_group_for_single_protagonist(tmp_path):
    init_project(tmp_path, idea_bank={"protagonist_structure": "单主角+辅助视角", ...})
    assert not (tmp_path / "设定集" / "主角组.md").exists()
    assert (tmp_path / "设定集" / "主角卡.md").exists()

def test_init_skips_dead_template_files(tmp_path):
    init_project(tmp_path, idea_bank={...})
    for dead in ("金手指设计.md", "爽点规划.md"):
        assert not (tmp_path / "设定集" / dead).exists()
```

**工作量**：小（约 2 小时）

---

### Issue #5 — chapter_focus 字段被错塞无关 summary（P1）

**现状证据**：

`chapter_002.json:override_allowed.chapter_focus = "文斗场面的张力来自观点击中修行根基。"`

这是从 dynamic_context 里 SP-087 卡的 `核心摘要` 直抄。但第 2 章是"井边对话收集借贷情报"，不是文斗场景。

**影响**：当 LLM 把 chapter_focus 当本章核心方向时，会被误导走偏。

**根因**：

`webnovel-writer/scripts/data_modules/story_system_engine.py` 的 `_suggest_chapter_focus()` 在没有显式大纲目标时，直接返回第一条 `dynamic_context` 的 `核心摘要`。由于 reasoning 排序会把 SP-087 这类泛题材卡排到前面，`chapter_focus` 就被无关 summary 污染；这不是 `runtime_contract_builder.py` 的行为。

**修复方案**：

与 #1 联动 —— 一旦 chapter_directive 接入，chapter_focus 取自 `directive.goal`。**没有 directive 时取 query（前提是非占位）或留空，绝不从 dynamic_context 抄**。

**验收标准**：

`test_chapter_brief_contains_outline_directive` 已涵盖（chapter_focus 来自大纲目标）。补充：

```python
def test_chapter_focus_never_taken_from_dynamic_context_summary(tmp_path):
    brief = build_chapter_brief(tmp_path, chapter=99)  # 大纲没有第 99 章
    # 没大纲时 chapter_focus 应为空字符串或 None，而非 SP 卡 summary
    assert brief["override_allowed"].get("chapter_focus", "") == ""
```

**工作量**：小（约 30 分钟，包含在 #1 改动里）

---

### Issue #6 — 设定文件缺少占位/漂移防护（P2）

**现状证据**：

- 卷纲 `第1卷-卷纲.md:25` 行：`第一位女主（暂名） | 女主 | [待章纲拆分时具体设计] | 第20章起`
- 主角卡：`苏云——第一位女主...`
- 女主卡：完整 `苏云` 人物卡

主角卡 / 女主卡是 plan 阶段的某次产出后填好的，**卷纲表格停留在较早版本，没有后续一致性检查**。

主角卡里还有：
- `[待补充]：能打但脑子不够用的兄弟型角色（后续卷引入）`

未来发现这个角色没填，到写到该卷时才暴露。

**影响**：plan 阶段产出文件之间数据漂移；写章前没提醒导致用户没意识到未填项。

**根因**：

这不是某条“自动回填流水线失效”，而是**当前系统只有“增量写回现有设定集”的流程承诺，没有占位扫描、跨文件一致性检查、也没有写章前的 pending blocker**。`webnovel-plan/SKILL.md` 明确要求“卷纲完成后，把新增设定增量写回现有设定集”，但仓库里没有对应的 placeholder guardrail。

**修复方案**：

1. 新增 `webnovel-writer/scripts/data_modules/placeholder_scanner.py`：
   - 扫描 `大纲/*.md`、`设定集/*.md` 中匹配以下模式的占位：
     - `\[待[^\]]*\]`（如 `[待补充]`、`[待章纲拆分时具体设计]`、`[待定]`）
     - `（暂名）`、`(暂名)`、`（待补充）`
     - 全字段 `{占位}`、`<占位>` 块
   - 输出每条占位的 `{file, line, context, suggested_fill_phase}`

2. 在以下流程**前**加阻断检查：
   - `webnovel-plan` 开始或结束时：扫所有 plan 文件，警告但不阻断
   - `webnovel-write` 开始时：扫**当前章节涉及的实体**对应的设定文件，**有 `[待...]` 阻断写章**
3. 加一个 CLI 命令 `webnovel.py placeholder-scan --project-root <dir>` 让用户主动检查

**验收标准**：

```python
def test_placeholder_scanner_finds_pending_marks(tmp_path):
    f = tmp_path / "大纲" / "第1卷-卷纲.md"
    f.write_text("第一位女主（暂名）| [待章纲拆分时具体设计]")
    results = scan_placeholders(tmp_path)
    assert len(results) == 2
    assert any("（暂名）" in r["pattern"] for r in results)
    assert any("[待章纲" in r["pattern"] for r in results)

def test_write_chapter_blocks_on_pending_protagonist_card(tmp_path):
    # 主角卡有 [待补充] 但本章没涉及该实体 → 不阻断
    # 主角卡有 [待补充] 且本章涉及该实体 → 阻断
    ...
```

**工作量**：小（约 2 小时）

---

### Issue #7 — 当前卷完成后的最小跨卷锚点缺失（P2）

**现状证据**：

`大纲/总纲.md:46-62`：

```
| 卷号 | 卷名 | 章节范围 | 核心冲突 | 卷末高潮 |
| 1 | 阎王债 | 第1-50章 | <已填> | <已填> |
| 2 | | 第51-100章 | | |
...
| 20 | | 第951-1000章 | | |
```

伏笔表只有 2 条核心伏笔 + 1 行空。

**影响**：plan 阶段的 LLM 没有跨卷视野，每次开新卷都从头规划，可能跟前期承诺/伏笔脱节。

**修复方案**：

1. **不在 init 时就生成 V2-V20 行**（避免一堆空表格视觉污染）
2. 把 Phase 7 定义为当前卷规划成功后的**最小跨卷锚点写回**，不是自动详细规划下一卷：
   - 触发条件：`webnovel-plan` 已成功完成当前卷规划，且当前卷规划 artifacts 已完整/通过验证
   - 写回内容：只填 V+1 的 `卷名`、`核心冲突`、`卷末高潮`
   - 写入位置：`大纲/总纲.md` 的下一卷行；若不存在则追加一行
3. 伏笔表加受限追加：每次 `webnovel-plan` 完成一卷规划时，只追加当前规划输出中显式结构化产出的：
   - 新伏笔
   - 继续开放的 open loops
4. 明确禁止：
   - 不从自由文本推断伏笔或开放环
   - 不生成下一卷详细大纲
   - 不生成下一卷 beat sheet
   - 不生成下一卷 timeline
   - 不预生成 V2-V20 空行

这一路径最贴合写作流：当前卷规划完成时，系统只把必要的跨卷承诺固化到总纲，保留下一卷正式规划时的创作空间。

**验收标准**：

```python
def test_plan_flow_writes_minimal_next_volume_anchor(tmp_path):
    plan_volume(tmp_path, volume=1)  # 这里代表 webnovel-plan 完成 V1 规划
    # 完成后总纲应自动多了 V2 最小锚点（V3-V20 不存在/不预填）
    summary = (tmp_path / "大纲" / "总纲.md").read_text(encoding="utf-8")
    assert "| 2 |" in summary
    v2_line = [l for l in summary.split("\n") if l.startswith("| 2 |")][0]
    assert v2_line.count("|") == 6
    assert "第2卷-详细大纲.md" not in list_outline_files(tmp_path)
    assert "第2卷-beat-sheet.md" not in list_outline_files(tmp_path)
    assert "第2卷-timeline.md" not in list_outline_files(tmp_path)

def test_plan_flow_appends_only_structured_foreshadow_items(tmp_path):
    plan_volume(tmp_path, volume=1, structured_foreshadow=[
        {"type": "new_foreshadowing", "text": "债契背面的红印仍未解释"},
        {"type": "continued_open_loop", "text": "苏云身份与阎王债源头仍未闭合"},
    ])
    summary = (tmp_path / "大纲" / "总纲.md").read_text(encoding="utf-8")
    assert "债契背面的红印仍未解释" in summary
    assert "苏云身份与阎王债源头仍未闭合" in summary
```

**工作量**：中（约 4 小时，包含 `webnovel-plan` prompt / 流程改动）

---

### Issue #8 — 审查发现的 ai_flavor 模式不回流 anti_patterns（P2）

**现状证据**：

`灵石庄/.story-system/anti_patterns.json`：10 条全部来自 reference 库初始化（GR-011 / NR-077 / CH-072 / SP-087 / RS-002 的"毒点"字段）。

第 2 章审查报告 抓到的 ai_flavor：
- "唯一一个知道 X 的人" 三连排比
- "第一片 / 第二片 / 第三片" 三段式枚举

**这些模式没进入 anti_patterns.json**。后续章节没有"上一章已经被点出的句式不要重复"的避雷清单。

**修复方案**：

1. `webnovel-writer/scripts/data_modules/review_schema.py`（或对应处）在持久化审查报告后，新增 hook：
   ```python
   if issue["category"] == "ai_flavor" and issue["severity"] in ("medium", "high", "critical"):
       anti_patterns_store.append({
           "text": issue["evidence"][:200],
           "source_table": "review_extracted",
           "source_id": f"ch{chapter:04d}_issue_{issue_idx}",
           "category": issue["category"],
           "added_at": now_iso(),
       })
   ```
2. 写章前 brief 注入 anti_patterns 的 `text` 列表作为"避雷模式"
3. 加去重机制（同一 evidence 不重复入库）

**验收标准**：

```python
def test_ai_flavor_review_issue_added_to_anti_patterns(tmp_path):
    save_review_report(tmp_path, chapter=2, issues=[{
        "category": "ai_flavor",
        "severity": "medium",
        "evidence": "唯一一个知道复利公式的人。唯一一个知道..."
    }])
    patterns = json.loads((tmp_path / ".story-system" / "anti_patterns.json").read_text())
    assert any("唯一一个知道" in p["text"] for p in patterns)
    assert any(p["source_id"].startswith("ch0002_") for p in patterns)
```

**工作量**：小（约 2 小时）

---

## 5. 整体回归方案

完成后必须通过：

1. `pytest --no-cov` 全过（当前 508 个用例）
2. 灵石庄目录用 init 流程重跑（新建 `灵石庄2`）—— 验证：
   - 不生成 `主角组.md`、`金手指设计.md`、`复合题材-融合逻辑.md`、`爽点规划.md`
   - 不生成空目录
3. 第 1 章重写一次，对比 brief：
   - 必须包含 `chapter_directive` 字段
   - dynamic_context 至少有 1 条与"借贷/借据/利息"相关
   - chapter_focus 来自大纲第 1 章的 goal
4. 写完第 1 章后审查 → 故意制造 ai_flavor 句式 → 确认 anti_patterns.json 有新增
5. 完成并验证第 1 卷规划后 → 总纲只应写回 V2 的卷名 / 核心冲突 / 卷末高潮；不得生成 V2 详细大纲、beat sheet、timeline，也不得预填 V3-V20 空行
6. 当前卷规划输出含显式结构化的新伏笔 / 持续开放环时 → 总纲伏笔表追加这些结构化条目；自由文本里的暗示不得被推断追加
7. 主角卡保留 `[待补充]` 然后写第 50 章（涉及该角色） → 应阻断

---

## 6. 风险与已知限制

- #1 的 outline 解析依赖 markdown 章节块格式稳定。若 plan agent 输出格式变体（如标题用 `第1章` vs `第一章`），需要兼容。建议在 outline_extractor 用 regex 匹配 `第\s*\d+\s*章` 全部变体。
- #6 的占位扫描可能误报真实文本里的方括号（如"\[原文\]"）。建议白名单 + 模式严格化（必须含中文"待"字才算占位）。
- #7 的 V+1 写回必须保持最小锚点边界。建议只接受当前规划产物中结构化输出的卷名 / 核心冲突 / 卷末高潮，不从自由文本扩写，也不生成下一卷详细规划；真正的下一卷详细大纲、beat sheet、timeline 留到用户进入下一卷规划时再产出。
- 与已修的 protagonist_state 镜像（f58d657）可能在 #1 启用 directive 后有交互：directive 里有 time_anchor，state.protagonist_state 里有 location.last_chapter，两者不应被混写。需在 #1 实现时显式分离命名空间。

---

## 7. 不在本 spec 范围

| 类型 | 例子 | 责任 |
|------|------|------|
| LLM 输出本身的写作瑕疵 | AI 味、习惯动作过密、句式重复 | 审查 agent 已抓 |
| init 时用户未填的项目元数据 | heroine_names 空 | 用户责任，UI 应提示 |
| 模型选型 / 上下文窗口大小 | 一次生成不下大章 → 增量更新 | 模型/平台层 |
| 数据 agent 调用慢（NOT_FOUND 重试） | 见 `data_agent_timing.jsonl` | 已在 f58d657 修主因 |

---

## 8. 推荐实施顺序（可直接开工）

1. 在 `chapter_outline_loader.py` 先实现 `load_chapter_execution_directive()`，用最小 fixture 覆盖详细大纲第 1 章字段提取。
2. 把 `chapter_directive` 接入 `story_system.py` / `story_system_engine.py` 的 brief 组装，并让 `chapter_focus` 优先取 `directive.goal`。
3. 更新 `webnovel-write/SKILL.md`，把“本章硬性约束 / CBN-CPNs-CEN / 本章禁区”排到 `dynamic_context` 前。
4. 给 `story_system.py` 增加 placeholder query 诊断，并同步更新 `webnovel-plan`、`webnovel-write`、`webnovel-review` 三个 skill 的真实 query 要求。
5. 在 `story_system_engine.py` 增加 chapter-aware reference ranking，用 `goal/key_entities/strand/antagonist_tier` 影响排序。
6. 修改 `init_project.py` 与 `webnovel-init/SKILL.md` 的模板生成清单，先完成单主角 / 无女主 / 死模板剪枝测试。
7. 新增 `placeholder_scanner.py` 与 `webnovel.py placeholder-scan`，先实现全项目扫描，再接入 plan 警告。
8. 将 placeholder guardrail 接入 write 前检查，使用当前章节相关实体缩小阻断范围。
9. 修改总纲模板与 `webnovel-plan/SKILL.md`，移除 V2-V20 预生成空行；在当前卷规划完成并验证后，只写回 V+1 的卷名 / 核心冲突 / 卷末高潮，并仅追加当前规划输出显式结构化给出的新伏笔 / 持续开放环。
10. 在 `review_schema.py` 增加 `ai_flavor` → `anti_patterns.json` 回流与去重，再验证下一章 brief 能读到 review-extracted 避雷模式。
