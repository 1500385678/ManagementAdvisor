# LLM 会议纪要 5 段式抽取 prompt 样例 v0.1

> Phase 1 启动 6 步 checklist 第 4 步 · 2026-09-14 · T4 业务窗口主动响应
> 触发来源:`_rules/Phase1会议纪要数据流设计v0.1.md` §六 步骤 4
> 范围:节点 6 LLM 5 段式抽取 prompt 设计(系统提示词 + 用户模板 + few-shot + JSON schema + 降级指令 + 调试方法)
> 前置资产:4.1 [[../_templates/会议纪要结构化提取模板v0.1]] §二 5 段式 + 4.2 [[../_rules/飞书会议API接入测试v0.1]] §五 边界预研 + 桥接 [[../_rules/Phase1会议纪要数据流设计v0.1]] §三 字段映射
> 形态:prompt 样例(可入库)+ few-shot 3 类会议(周会/月会/项目会)+ JSON Schema + 调试方法
> 不依赖:Claude Sonnet 4.5 API 真机调用 + 飞书真机凭证 + 真实公司纪要

---

## 一、设计原则

1. **5 段式严格沿用 4.1**:背景 / 讨论要点 / 决策 / 行动项 / 风险,字段名不允许偏离
2. **字段契约严格沿用桥接 §三**:所有字段名 / 数据类型 / 必填规则与 `_rules/Phase1会议纪要数据流设计v0.1.md` §三 完全一致
3. **JSON 优先输出**:LLM 输出标准化 JSON,节点 7 按 4.1 §五 Markdown 模板渲染,避免二次解析
4. **降级指令显式化**:用户提示词必须接收 4 类降级指令(全功能 / 警告标签 / 仅元信息 / 失败兜底),不允许 LLM 自行决定降级
5. **说话人映射必做**:用户提示词中 `speaker_id → 真实姓名` 映射表由节点 4 注入,LLM 必须使用映射后姓名
6. **few-shot 3 类会议**:周会 / 月会 / 项目会各 1 个示例,沿用 4.1 §三 差异化字段(周会 5-10 议题 / 月会 OKR 进展 / 项目会 评审意见)
7. **可调试可回归**:所有示例数据使用占位转写(不依赖真实公司纪要),便于 Phase 1 启动后用真机数据回归测试

---

## 二、系统提示词(System Prompt)

```text
你是 30-管理 行业顾问的会议纪要抽取助手。你的任务是基于"会议转写 + 说话人映射表 + 会议元信息"3 个输入,抽取 5 段式结构化纪要。

# 5 段式骨架(严格沿用 4.1 模板 §二)
1. 背景(7 字段):会议名称 / 会议类别 / 会议时间 / 参会人 / 缺席人 / 会议目的 / 关联项目
2. 讨论要点(动态字段):议题 1-N / 共识 / 分歧
3. 决策(3 字段):决策内容 / 决策人 / 反对意见
4. 行动项(4 字段必填 + 2 字段可选):责任人 / 截止时间 / 优先级 / 状态 / 关联决策(可选)/ 验收标准(可选)
5. 风险(动态字段):风险 1-N / 风险等级

# 行为规范
- 字段名严格使用中文,不允许翻译成英文或缩写
- 行动项"状态"字段新建时统一填"待开始",不允许填"已完成"或"进行中"
- 决策"反对意见"字段若无人明确反对,填"无",不允许省略
- 风险"风险等级"高/中/低,缺省填"中"
- 输出必须是 JSON,不允许输出 Markdown 表格或散文
- 不允许在纪要中编造未在转写中出现的姓名 / 日期 / 数字

# 降级指令处理(由用户提示词传入降级等级)
- 全功能:抽取所有 5 段
- 警告标签:抽取所有 5 段,但在 JSON 顶层加 `warnings: [...]` 字段
- 仅元信息:仅抽取段 1 背景 7 字段,其余段填空数组
- 失败兜底:返回 `{"error": "<reason>", "meeting_id": "<id>"}`,不抽取任何字段
```

**设计说明**:
- 第 1 段"5 段式骨架"与 4.1 §二完全对应,LLM 不需要重新理解结构
- 第 2 段"行为规范"对应桥接 §三 字段映射表的"必填规则"和"缺省值"
- "降级指令处理"对应桥接 §四 降级策略表的 4 类行为

---

## 三、用户提示词模板(User Prompt Template)

```text
# 会议元信息(来自节点 1 / 节点 2)
```json
{{meeting_meta}}
```

# 说话人映射表(来自节点 4,speaker_id → 真实姓名)
```json
{{speaker_map}}
```

# 会议转写(来自节点 3,已按时间排序)
```
{{transcript}}
```

# 会议类别判定(来自节点 5)
- 类别: {{meeting_category}}  # 周会 / 月会 / 项目会 / 1v1 / 临时会
- 参会人数: {{participant_count}}
- 方言占比: {{dialect_ratio}}  # 0.0 ~ 1.0
- 英文占比: {{english_ratio}}
- 降级等级: {{degradation_level}}  # 全功能 / 警告标签 / 仅元信息 / 失败兜底

# 抽取任务
请基于以上输入,按 5 段式骨架抽取结构化纪要,输出 JSON。

# 特别提醒
- 行动项"责任人"必须使用"说话人映射表"中的真实姓名,不允许保留 speaker_id
- 行动项"截止时间"必须标准化为 ISO 8601 日期(YYYY-MM-DD),不允许"下周""尽快"等模糊表达
- 行动项"优先级"判定:含"紧急/P0/ASAP"→ P0;含"重要/P1/这周"→ P1;其他 → P2
- 段 5 风险至少抽取 1 条,若转写中无明确风险信号,从行动项中"延期风险""依赖阻塞"反向推导
- 降级等级 = "仅元信息" 时,只输出段 1 背景,段 2-5 字段为空数组
- 降级等级 = "失败兜底" 时,返回 `{"error": "...", "meeting_id": "..."}`
```

**设计说明**:
- 4 个输入块(meeting_meta / speaker_map / transcript / 会议类别判定)与桥接 §二 节点 1-5 一一对应
- "特别提醒"段是"行为规范"在用户提示词中的强化提示(LLM 容易遗忘边界规则,需要显式提醒)
- 模板用 `{{var}}` 占位符,Phase 1 编码时由节点 6 渲染函数填充

---

## 四、Few-shot 示例(3 类会议)

### 4.1 周会示例

**输入摘要**:
- 会议元信息:`{"meeting_name": "产品组周会-2026-W37", "start_time": "2026-09-14T10:00:00+08:00", "end_time": "2026-09-14T11:00:00+08:00", "purpose": "同步本周进展 + 识别阻塞"}`
- 说话人映射:`{"spk_001": "张三", "spk_002": "李四", "spk_003": "王五"}`
- 转写(节选):
  > 张三:这周登录页改版上线了,转化率提升 12%。
  > 李四:我这边搜索接口出了点问题,可能影响明天的灰度。
  > 王五:测试那边已经准备好回归用例了,周三可以走完。
  > 张三:好,那搜索接口的事李四这周内修一下,P1。
- 会议类别:周会 / 3 人 / 方言 0.0 / 英文 0.0 / 全功能

**期望输出**:
```json
{
  "background": {
    "meeting_name": "产品组周会-2026-W37",
    "meeting_category": "周会",
    "meeting_time": "2026-09-14T10:00:00+08:00 ~ 2026-09-14T11:00:00+08:00",
    "participants": ["张三", "李四", "王五"],
    "absentees": [],
    "purpose": "同步本周进展 + 识别阻塞",
    "related_projects": ["登录页改版", "搜索接口优化"]
  },
  "discussion_points": [
    {
      "topic": "登录页改版上线效果",
      "main_view": "转化率提升 12%",
      "key_arguments": ["改版 9/12 上线,数据稳定"],
      "consensus": "效果符合预期",
      "disagreement": "无"
    },
    {
      "topic": "搜索接口故障",
      "main_view": "可能影响 9/15 灰度",
      "key_arguments": ["接口报错率上升", "测试用例已就绪"],
      "consensus": "本周内修复",
      "disagreement": "无"
    }
  ],
  "decisions": [
    {
      "decision_content": "搜索接口故障本周内修复",
      "decision_maker": "张三",
      "objection": "无"
    }
  ],
  "action_items": [
    {
      "owner": "李四",
      "deadline": "2026-09-20",
      "priority": "P1",
      "status": "待开始",
      "related_decision": "搜索接口故障本周内修复",
      "acceptance_criteria": "搜索接口报错率 < 0.1%"
    },
    {
      "owner": "王五",
      "deadline": "2026-09-17",
      "priority": "P2",
      "status": "待开始",
      "related_decision": null,
      "acceptance_criteria": "搜索接口回归用例全绿"
    }
  ],
  "risks": [
    {
      "risk_description": "搜索接口修复延期影响灰度发布",
      "owner": "李四",
      "mitigation": "提前与测试对齐回归时间",
      "risk_level": "中"
    }
  ],
  "warnings": [],
  "metadata": {
    "extraction_version": "v0.1",
    "extraction_time": "<运行时填充>",
    "model": "claude-sonnet-4.5"
  }
}
```

### 4.2 月会示例(摘要)

**输入摘要**:
- 会议元信息:`{"meeting_name": "研发中心月会-202609", "purpose": "月度复盘 + 下月规划"}`
- 说话人映射:5 人
- 转写含 OKR 进展 / 资源调配 / 下月 Roadmap
- 会议类别:月会 / 5 人 / 全功能

**期望输出结构差异**:
- `background.purpose` 后追加 `monthly_review: "9 月 OKR 完成率 78%"`
- `discussion_points` 中至少 1 个 topic 字段为 `okr_progress: "OKR-3: 用户增长 +15%,达成"`
- `action_items` 数量 5-15 条(周会示例只有 2 条)

### 4.3 项目会示例(摘要)

**输入摘要**:
- 会议元信息:`{"meeting_name": "订单中心项目评审-2026-09-14", "purpose": "技术方案评审"}`
- 说话人映射:4 人
- 转写含架构选型 / 风险评估 / 排期
- 会议类别:项目会 / 4 人 / 全功能

**期望输出结构差异**:
- `discussion_points` 中至少 1 个 topic 字段为 `review_opinion: "P0 风险:数据迁移一致性,需 2 人日压测"`
- `decisions` 至少 1 条 `decision_content: "采用方案 B(Event Sourcing + 读模型 CQRS)"`

**完整示例不展开**:Phase 1 T2 算法调试时按 4.1 周会示例同结构扩展。

---

## 五、JSON 输出 Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "MeetingMinutes5Sections",
  "type": "object",
  "required": ["background", "discussion_points", "decisions", "action_items", "risks", "metadata"],
  "properties": {
    "background": {
      "type": "object",
      "required": ["meeting_name", "meeting_category", "meeting_time", "participants", "absentees", "purpose", "related_projects"],
      "properties": {
        "meeting_name": {"type": "string"},
        "meeting_category": {"enum": ["周会", "月会", "项目会", "1v1", "临时会"]},
        "meeting_time": {"type": "string", "format": "date-time"},
        "participants": {"type": "array", "items": {"type": "string"}},
        "absentees": {"type": "array", "items": {"type": "string"}},
        "purpose": {"type": "string"},
        "related_projects": {"type": "array", "items": {"type": "string"}}
      }
    },
    "discussion_points": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["topic", "main_view", "consensus", "disagreement"],
        "properties": {
          "topic": {"type": "string"},
          "main_view": {"type": "string"},
          "key_arguments": {"type": "array", "items": {"type": "string"}},
          "consensus": {"type": "string"},
          "disagreement": {"type": "string"},
          "okr_progress": {"type": "string", "description": "月会专用"},
          "review_opinion": {"type": "string", "description": "项目会专用"}
        }
      }
    },
    "decisions": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["decision_content", "decision_maker", "objection"],
        "properties": {
          "decision_content": {"type": "string"},
          "decision_maker": {"type": "string"},
          "objection": {"type": "string"}
        }
      }
    },
    "action_items": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["owner", "deadline", "priority", "status"],
        "properties": {
          "owner": {"type": "string"},
          "deadline": {"type": "string", "format": "date"},
          "priority": {"enum": ["P0", "P1", "P2"]},
          "status": {"enum": ["待开始", "进行中", "已完成", "已取消"]},
          "related_decision": {"type": ["string", "null"]},
          "acceptance_criteria": {"type": "string"}
        }
      }
    },
    "risks": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["risk_description", "owner", "mitigation", "risk_level"],
        "properties": {
          "risk_description": {"type": "string"},
          "owner": {"type": "string"},
          "mitigation": {"type": "string"},
          "risk_level": {"enum": ["高", "中", "低"]}
        }
      }
    },
    "warnings": {
      "type": "array",
      "items": {"type": "string"},
      "description": "降级等级 = 警告标签时填充"
    },
    "metadata": {
      "type": "object",
      "required": ["extraction_version", "model"],
      "properties": {
        "extraction_version": {"type": "string"},
        "extraction_time": {"type": "string", "format": "date-time"},
        "model": {"type": "string"}
      }
    }
  }
}
```

**设计说明**:
- 必填字段严格沿用 4.1 模板 + 桥接 §三 字段映射表
- `discussion_points.items.properties` 中的 `okr_progress` / `review_opinion` 是 3 类会议差异化字段(JSON Schema 不强制 required,允许 LLM 按会议类别选择性输出)
- `warnings` 字段仅降级等级 = 警告标签时填充,全功能时为空数组

---

## 六、降级指令集(4 类降级)

| 降级等级 | 触发条件(沿用桥接 §四) | 行为 |
|---|---|---|
| 全功能 | 普通话 + 2-15 人 + 静音正常 | 抽取所有 5 段,warnings = [] |
| 警告标签 | 6-15 人 / 粤英混说 / 多人讨论 | 抽取所有 5 段,warnings 填具体警告 |
| 仅元信息 | 16-50 人 / 纯英文 / 高噪音 | 仅段 1 背景 7 字段,段 2-5 字段 = [] |
| 失败兜底 | 凭证缺失 / API 报错 / 边界外 | 返回 `{"error": "...", "meeting_id": "..."}` |

**警告标签填充规则**(LLM 必须按以下规则填充 `warnings`):
- 6-15 人会议:`"多人会议说话人标记仅供参考"`
- 粤英混说:`"转写含方言,部分关键词识别可能偏差"`
- 高噪音但可生成:`"会议背景噪音较大,转写置信度 < 80%"`

**失败兜底触发条件**:
- 会议凭证缺失:`{"error": "missing_credentials", "meeting_id": "..."}`
- API 调用失败:`{"error": "api_failure: <endpoint>", "meeting_id": "..."}`
- 转写为空:`{"error": "empty_transcript", "meeting_id": "..."}`

---

## 七、调试方法(5 步)

**步骤 1 · 单元调试(不依赖 Claude API)**
- 把 §四 周会示例的"输入摘要"作为 fixture,人工撰写期望输出 JSON
- 校验期望输出符合 §五 JSON Schema

**步骤 2 · Claude API dry-run(需 API key)**
- 用 §四 3 个示例分别调用 Claude Sonnet 4.5
- 对比 LLM 实际输出与期望 JSON 差异
- 重点检查:`meeting_category` 是否落入 enum / `deadline` 是否标准化 ISO / `priority` 判定是否符合规则

**步骤 3 · 降级路径 dry-run**
- 模拟 4 类降级等级输入,验证 LLM 输出符合 §六 行为表
- 重点检查:仅元信息降级时段 2-5 是否为空数组 / 失败兜底是否仅返回 error 字段

**步骤 4 · 说话人映射 dry-run**
- 构造 speaker_id 未在映射表中的转写,验证 LLM 是否保留 speaker_id 或拒绝输出
- 期望:LLM 遇到未映射 speaker 时,在 `warnings` 中追加 `"unmapped_speaker: spk_005"`,不静默编造姓名

**步骤 5 · 真实转写回归(Phase 1 启动后)**
- 用 dogfood 3 个团队周会跑 4 周的真机转写(桥接 §六 步骤 6)
- 与人工纪要对比 5 段式准确率
- 验收标准:背景 7 字段 ≥ 95% / 决策 / 行动项 ≥ 90% / 风险 ≥ 80%

---

## 八、关联文档

- 4.1 输入格式契约:[[../_templates/会议纪要结构化提取模板v0.1]] §二 5 段式 + §三 3 类会议差异化 + §五 Markdown 模板
- 4.2 数据来源契约:[[../_rules/飞书会议API接入测试v0.1]] §二 4 API + §五 边界预研 + §三.5 字段映射草案
- 桥接数据流设计:[[../_rules/Phase1会议纪要数据流设计v0.1]] §三 字段映射 + §四 降级策略 + §六 启动 checklist 步骤 4
- 行动项 → 飞书任务映射(checklist 步骤 5,本文档不涉及):待 Phase 1 启动时 T2 起草
- 项目立项:[[../项目开发计划]] §5 Phase 0 桥接 → §6 Phase 1 第 1 项

---

## 九、不做什么

- **不调 Claude API**:本文档只设计 prompt 样例,实际调用留到 Phase 1 启动步骤 4 dry-run 阶段
- **不依赖真实公司纪要**:所有示例使用占位转写数据,不消耗张勇侧资源
- **不替代 T2 算法调试**:本文档是"prompt 设计样例",实际 prompt 调优(温度 / max_tokens / 提示词微调)由 Phase 1 T2 算法完成
- **不涉及行动项 → 飞书任务映射**:checklist 步骤 5 单独起草,本文档仅完成步骤 4
- **不写降级兜底话术**:沿用 `_templates/飞书bot指令识别设计v0.1` §三 兜底话术 3 条 + 桥接 §四 降级兜底话术,不在本文档重复

---

## 十、变更记录

| 日期 | 版本 | 变更 | 作者 |
|---|---|---|---|
| 2026-09-14 | v0.1 | 初版入库 · Phase 1 启动 6 步 checklist 第 4 步 · prompt 样例 + few-shot 3 类会议 + JSON Schema + 降级指令 + 5 步调试方法 | T4 业务窗口(03:50 cron · .plan/20260914.md 缺失 · 应急增量) |
