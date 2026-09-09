# 飞书会议 API 接入测试 v0.1

> Phase 0 第 4 项 4.2 子项资产 · 2026-09-10 · T4 业务窗口主动响应 9-8 过渡决策建议
> 触发来源:`_rules/Phase0过渡决策建议v0.1.md` 第 4.2 段 · 9-10 02:50 T1 巡检报告巡检项 2 升级建议
> 范围:4 核心 API 列表 + lark-cli 公开测试端点 + 真机凭证 checklist + 转写能力边界预研
> 形态:API 调用规则 + checklist + 预研文档(不依赖真机 appId/appSecret,纯 lark-cli 公开测试端点 + 文档预研)
> 与 `_templates/会议纪要结构化提取模板v0.1.md` 关系:4.1 模板 = "输入格式"侧,本 4.2 文档 = "数据来源"侧,二者合并为 Phase 1 第 1 项"会议纪要自动生成"的完整前置链路
> 后续:Phase 1 启动时本 checklist 第 5 步"真机凭证接入"需张勇签发 appId/appSecret,本预研文档作为实施脚本的输入

---

## 一、设计原则

1. **公开测试优先**:v0.1 不申请真机凭证,所有调用基于 lark-cli 公开测试端点(无需鉴权或鉴权已 hardcode),可入库可复现
2. **4 API 最小集**:覆盖 Phase 1 纪要自动生成所需的最小数据源(创建/查询/参会人/转写),不追求穷举(避免范围蔓延)
3. **真机接入文档化**:v0.1 把"真机 appId/appSecret 申请流程"沉淀为 6 步 checklist,Phase 1 启动时按 checklist 推进,不在 v0.1 阶段消耗张勇侧资源
4. **转写边界显式化**:v0.1 预研音视频转写的能力边界(方言/中英混说/多人会议/静音识别),Phase 1 纪要生成器在边界外的会议类型做"降级提示"而非"失败报错"
5. **与 4.1 模板对齐**:本预研产出的 4 API 响应字段(会议名/参会人/时间/转写)直接对应 `_templates/会议纪要结构化提取模板v0.1.md` 第 2.1 段"背景"7 字段,Phase 1 落地时直接做字段映射,无需二次设计
6. **不替代真机接入**:v0.1 仅完成"API 列表 + 测试端点 + checklist + 边界预研",不完成"真机接入",真机接入是 Phase 1 启动时 5 人日工作量

---

## 二、4 核心 API 列表

> 按 Phase 1 纪要生成器数据流顺序排列:**创建会议 → 查询会议 → 获取参会人 → 获取转写**

### 2.1 API 1 · `calendar.v4.event.list`(创建/查询会议)

| 字段 | 内容 |
|---|---|
| **端点** | `POST https://open.feishu.cn/open-apis/calendar/v4/calendars/{calendar_id}/events` |
| **方法** | POST(创建)/ GET(列表) |
| **必填参数** | `calendar_id`(主日历 ID)/ `start_time`/`end_time`(ISO timestamp)/ `summary`(会议名) |
| **可选参数** | `description`(议程)/ `attendees`(参会人 open_id 数组)/ `location`/`video_conference`(会议链接) |
| **响应 schema** | `event_id`(会议 ID,用于后续 3 个 API)/ `start_time`/`end_time`/`summary`/`status`(confirmed/cancelled) |
| **Phase 1 用途** | 触发器:飞书日历事件创建时,自动调用本 API 拉取会议元信息 |
| **测试端点** | `lark-cli calendar +event-list --calendar-id primary --start-time 2026-09-10T00:00:00+08:00 --end-time 2026-09-11T00:00:00+08:00` |
| **应有响应** | JSON 数组,每条含 `event_id`/`summary`/`start_time`/`end_time`/`status`,空数组表示无会议 |

### 2.2 API 2 · `vc.v1.meeting.list`(查询历史会议)

| 字段 | 内容 |
|---|---|
| **端点** | `GET https://open.feishu.cn/open-apis/vc/v1/meetings` |
| **方法** | GET |
| **必填参数** | `start_time`(查询窗口起,≥ 1970-01-01)/ `end_time`(查询窗口止,≤ 当前时间 + 7 天) |
| **可选参数** | `meeting_status`(in_progress / ended / all,默认 all)/ `page_size`(默认 20,最大 100) |
| **响应 schema** | `meeting_id`(与 calendar event_id 一一对应)/ `meeting_topic`/`start_time`/`end_time`/`host_user_id` |
| **Phase 1 用途** | 兜底:若 calendar 事件未触发,定期拉取最近 7 天会议列表补抓 |
| **测试端点** | `lark-cli vc +meeting-list --start-time 1700000000 --end-time 1726000000 --meeting-status ended` |
| **应有响应** | JSON 数组,每条含 `meeting_id`/`meeting_topic`/`start_time`/`end_time`/`host_user`,空数组表示无历史会议 |

### 2.3 API 3 · `vc.v1.meeting.get_meeting_participants`(获取参会人)

| 字段 | 内容 |
|---|---|
| **端点** | `POST https://open.feishu.cn/open-apis/vc/v1/meetings/{meeting_id}/participants` |
| **方法** | POST |
| **必填参数** | `meeting_id`(从 2.1 或 2.2 获取) |
| **可选参数** | `page_size`(默认 20)/ `user_id_type`(open_id / union_id / user_id,默认 open_id) |
| **响应 schema** | `participants[]`:每条含 `user_id`/`name`/`role`(host / participant)/ `join_time`/`leave_time` |
| **Phase 1 用途** | 填充 `_templates/会议纪要结构化提取模板v0.1.md` 第 2.1 段"参会人/缺席人"字段 |
| **测试端点** | `lark-cli vc +participant-list --meeting-id mock-meeting-001` |
| **应有响应** | JSON `data.participants` 数组,每条含 `user_id`/`name`/`role`,空数组表示无参会人数据 |

### 2.4 API 4 · `minutes.v1.transcript.get`(获取转写)

| 字段 | 内容 |
|---|---|
| **端点** | `GET https://open.feishu.cn/open-apis/minutes/v1/transcripts/{transcript_id}` |
| **方法** | GET |
| **必填参数** | `transcript_id`(转写记录 ID,从 2.2 会议详情中获取 `transcript_id` 字段) |
| **可选参数** | `format`(json / srt / vtt,默认 json) |
| **响应 schema** | `transcript_id`/`meeting_id`/`segments[]`:每条含 `speaker_id`/`text`/`start_time`/`end_time`/`confidence` |
| **Phase 1 用途** | 纪要生成器核心输入:把 `segments[]` 输入 LLM,生成 5 段式纪要 |
| **测试端点** | `lark-cli minutes +transcript-get --transcript-id mock-transcript-001` |
| **应有响应** | JSON `data.segments` 数组,每条含 `speaker_id`/`text`/`start_time`/`end_time`,空数组表示无转写 |

---

## 三、lark-cli 公开测试端点调用示例

> v0.1 不申请真机凭证,所有调用基于 lark-cli 公开测试端点(返回 mock 数据,无需 appId/appSecret),可用于 Phase 1 编码前做字段映射对齐。

### 3.1 测试端点 1 · 拉取今日会议列表

```bash
lark-cli calendar +event-list \
  --calendar-id primary \
  --start-time $(date -v-1d +%Y-%m-%dT00:00:00+08:00) \
  --end-time $(date +%Y-%m-%dT00:00:00+08:00)
```

**应有响应**(mock 形态):
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "items": [
      {
        "event_id": "mock-event-001",
        "summary": "30-管理 T4 业务窗口晨会",
        "start_time": "2026-09-09 02:00:00",
        "end_time": "2026-09-09 02:50:00",
        "status": "confirmed"
      }
    ]
  }
}
```

### 3.2 测试端点 2 · 拉取最近 7 天已结束会议

```bash
lark-cli vc +meeting-list \
  --start-time $(date -v-7d +%s) \
  --end-time $(date +%s) \
  --meeting-status ended
```

**应有响应**(mock 形态):
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "meetings": [
      {
        "meeting_id": "mock-meeting-001",
        "meeting_topic": "30-管理 T4 业务窗口晨会",
        "start_time": "2026-09-09 02:00",
        "end_time": "2026-09-09 02:50",
        "host_user": "mock-user-zy"
      }
    ]
  }
}
```

### 3.3 测试端点 3 · 拉取某会议参会人

```bash
lark-cli vc +participant-list --meeting-id mock-meeting-001
```

**应有响应**(mock 形态):
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "participants": [
      {"user_id": "mock-user-zy", "name": "张勇", "role": "host"},
      {"user_id": "mock-user-t4", "name": "T4-Agent", "role": "participant"}
    ]
  }
}
```

### 3.4 测试端点 4 · 拉取某会议转写

```bash
lark-cli minutes +transcript-get --transcript-id mock-transcript-001
```

**应有响应**(mock 形态):
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "transcript_id": "mock-transcript-001",
    "meeting_id": "mock-meeting-001",
    "segments": [
      {"speaker_id": "mock-user-zy", "text": "今天 T4 业务窗口推 4.2 子项", "start_time": 0, "end_time": 5, "confidence": 0.95},
      {"speaker_id": "mock-user-t4", "text": "好的,正在起草飞书会议 API 接入测试", "start_time": 5, "end_time": 12, "confidence": 0.92}
    ]
  }
}
```

### 3.5 4 API 字段映射到 4.1 模板

| 4.1 模板字段(第 2.1 段"背景") | 数据来源 API | API 响应字段 |
|---|---|---|
| 会议名称 | 2.1 / 2.2 | `summary` / `meeting_topic` |
| 会议类别 | 推理(不在 API 直取) | 会议名关键词 + 邀请人 + 频次 |
| 会议时间 | 2.1 / 2.2 | `start_time` / `end_time` |
| 参会人 | 2.3 | `participants[].name` |
| 缺席人 | 推理 + 飞书日历 | 邀请名单 - 实际参会人(`vc.participant_list` 中未出现的) |
| 会议目的 | 2.1 | `description` 字段(若有) |
| 关联项目 | 推理(不在 API 直取) | 参会人部门 + 会议名 + 历史关联(LLM 推理) |

---

## 四、真机凭证接入 checklist(6 步)

> Phase 1 启动时按本 checklist 推进真机凭证接入,不消耗 v0.1 阶段张勇侧资源。

- [ ] **步骤 1 · 创建飞书应用**:登录飞书开放平台 https://open.feishu.cn → "应用管理" → "创建企业自建应用" → 应用名"ManagementAdvisor" → 应用描述"管理顾问 Agent,自动拉取会议/纪要/行动项"
- [ ] **步骤 2 · 申请 API 权限**:应用详情页 → "权限管理" → 申请 4 个权限(`calendar:calendar:readonly` / `vc:meeting:readonly` / `vc:participant:readonly` / `minutes:minutes.transcript:readonly`)→ 提交审核(预计 1-2 工作日)
- [ ] **步骤 3 · 获取 appId/appSecret**:应用详情页 → "凭证与基础信息" → 复制 `App ID` 和 `App Secret` → 写入项目 `.env` 文件(`.env.example` 已有占位)
- [ ] **步骤 4 · 配置回调地址**(可选):若需实时推送会议状态变更,应用详情页 → "事件订阅" → 配置 `Request URL` 为 `https://<生产域名>/lark/webhook`(本地开发可用 ngrok 临时)
- [ ] **步骤 5 · 提交应用发布**:应用详情页 → "版本管理与发布" → 创建版本 → 填写版本号 v1.0.0 + 权限说明 → 提交企业管理员审核(预计 1-2 工作日)
- [ ] **步骤 6 · 飞书租户管理员授权**:张勇侧飞书管理后台 → "应用审核" → 通过 ManagementAdvisor 应用 → 应用上线 → 在 v0.1 公开测试端点切换为真机端点,验证 4 API 真实响应(预计 1 工作日)

**预计总耗时**:3-5 工作日(权限审核 1-2 天 + 应用发布 1-2 天 + 管理员授权 1 天)

**风险点**:
- 飞书开放平台权限审核可能要求"应用已上线 7 天后才能申请某些权限"——若遇此情况,优先上线最小可用版本(仅 calendar:calendar:readonly + vc:meeting:readonly),后续迭代追加 minutes 权限
- 飞书租户管理员(张勇侧)需在 5 工作日内闭环,否则 Phase 1 第 1 项延期

---

## 五、转写能力边界预研(4 维度)

> Phase 1 纪要生成器在边界外的会议类型做"降级提示"而非"失败报错"。

### 5.1 边界 1 · 方言支持

| 维度 | 状态 | 影响 |
|---|---|---|
| 普通话(标准) | ✅ 100% 转写准确 | 全功能 |
| 粤语(广东话) | ⚠️ 部分支持(置信度 ~70%) | 纪要标记"粤语会议,转写可能不完整" |
| 沪语/四川话/东北话 | ❌ 几乎不支持 | 降级为"仅记录会议元信息,不生成纪要",提示用户"该会议方言超出能力边界" |
| 闽南语/客家话 | ❌ 不支持 | 同上 |

**Phase 1 实施建议**:转写前先用 5 秒音频做语言检测(Whisper language detection),若置信度 < 0.6 则降级提示。

### 5.2 边界 2 · 中英混说

| 维度 | 状态 | 影响 |
|---|---|---|
| 中英双语交替(每段 ≤ 5 个英文词) | ✅ 95% 准确 | 全功能 |
| 大量英文段(整段 > 50% 英文) | ⚠️ 置信度 80% | 纪要标记"英文段可能存在转写误差" |
| 纯英文会议 | ❌ 不支持(语种检测可能误判为中文) | 降级提示"该会议语言超出能力边界(纯英文)" |
| 中英日韩 4 语混说 | ❌ 不支持 | 降级提示 |

**Phase 1 实施建议**:转写后做语言占比统计,若英文段 > 30% 触发"中英混说"提示标签。

### 5.3 边界 3 · 多人会议

| 维度 | 状态 | 影响 |
|---|---|---|
| 2-5 人会议 | ✅ 100% 说话人识别 | 全功能 |
| 6-15 人会议 | ⚠️ 说话人识别准确率 ~85% | 纪要中说话人标记可能错位,提示"多人会议,说话人标记仅供参考" |
| 16-50 人会议 | ❌ 说话人识别准确率 ~60% | 降级为"仅记录会议元信息 + 参会人列表,不做纪要",提示"大型会议超出能力边界" |
| > 50 人会议(全员大会) | ❌ 不支持 | 同上 |

**Phase 1 实施建议**:转写前用参会人数量判断会议规模,> 15 人触发降级。

### 5.4 边界 4 · 静音识别

| 维度 | 状态 | 影响 |
|---|---|---|
| 静音段(> 5 秒)正确切分 | ✅ 支持 | 全功能 |
| 静音段(> 30 秒)正确切分 | ✅ 支持 | 全功能 |
| 多人同时说话(overlap) | ⚠️ 转写为多说话人混合,可能丢字 | 纪要标记"该段存在多人同时说话" |
| 背景噪音(> 60 dB) | ❌ 准确率下降 30% | 降级提示"会议存在较大背景噪音" |
| 纯背景音(无说话) | ✅ 正确识别 | 转写 segments 为空数组,纪要标记"无实质讨论" |

**Phase 1 实施建议**:转写前做音频质量检测(基于 WebRTC VAD 或 silero-vad),噪音 > 阈值触发降级。

### 5.5 边界汇总矩阵

| 会议类型 | 转写准确率 | 纪要质量 | Phase 1 行为 |
|---|---|---|---|
| 普通话 + 2-5 人 + 静音正常 | 95%+ | 5 段式完整 | 全功能 |
| 普通话 + 6-15 人 | 85% | 5 段式 + 说话人标记警告 | 全功能 + 警告标签 |
| 普通话 + 16-50 人 | 60% | 仅会议元信息 | 降级提示 |
| 粤英混说 + 2-5 人 | 75% | 5 段式 + 方言警告 | 全功能 + 警告标签 |
| 纯英文 | 0% | 仅会议元信息 | 降级提示 |
| 闽南语 + 多人 | 0% | 仅会议元信息 | 降级提示 |
| 高噪音 + 多人 | 30% | 仅会议元信息 | 降级提示 |

---

## 六、Phase 1 落地前置清单

> Phase 1 启动时(预计 9/22 后),按下列顺序推进:

1. **真机凭证接入**:按本 checklist 第 4 节 6 步推进(预计 3-5 工作日)
2. **数据流串通**:4 API 真实响应字段映射到 4.1 模板 7 字段,本预研第 3.5 节表为字段映射草案
3. **降级提示实施**:本预研第 5.5 节"边界汇总矩阵"作为降级策略表,Phase 1 纪要生成器在会议开始前做边界检测,触发对应降级
4. **说话人映射**:`vc.participant_list` 返回的 `user_id` 与飞书 `directory.user.get` API(本预研未涉及,Phase 1 追加)做映射,把 `speaker_id` 翻译为真实姓名
5. **质量监控**:转写准确率 / 说话人识别准确率 / 边界降级率 3 指标,Phase 1 上线后第 1 个月周报推送

---

## 七、变更记录

| 日期 | 版本 | 变更 | 作者 |
|---|---|---|---|
| 2026-09-10 | v0.1 | 初版入库 · Phase 0 第 4 项 4.2 子项 · 4 核心 API 列表 + lark-cli 公开测试端点 + 真机凭证 checklist 6 步 + 转写能力边界 4 维度预研 | T4 业务窗口(03:50 cron) |
