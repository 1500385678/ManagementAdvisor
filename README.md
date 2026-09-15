# ManagementAdvisor

> 30-管理-Management Level 行业 Web 项目 · 内部代号 ManagementAdvisor

## 项目说明
基于张勇的 36 行业架构,ManagementAdvisor 是 管理-Management Level 行业的 Web 端顾问产品。

## 同步
- GitHub: https://github.com/1500385678/ManagementAdvisor
- Gitee: https://gitee.com/architectzy/ManagementAdvisor

## 自动化
- T1 每日 02:50 自动巡检 + 落 .Log/巡检-管理-YYYYMMDD.md
- T4 每日 04:00 业务窗口(应急增量 · Phase 0 → Phase 1 桥接主动推进)
- T5 每日业务窗口 commit + push(gitee 优先,github 兜底)

## 项目进展(2026-09-16 视角)

### Phase 0 资产盘点(7/7 闭环 + 桥接资产 3 块扩展)
- [x] 第 1 项 · 主题速览 · `_index/主题速览.md`(8/26 commit 190a45c)
- [x] 第 3 项 · 资深管理者画像原型 · `_templates/资深管理者画像原型v0.1.md`(8/28 commit 5e2bdfb)
- [x] 第 5 项 · 飞书 bot 雏形指令识别 · `_templates/飞书bot指令识别设计v0.1.md`(9/1 commit 97b3b9d)
- [x] 第 6 项 · 健康度诊断规则 · `_rules/健康度诊断规则v0.1.md`(8/27 commit a2ea6e7)
- [x] 第 2 项 · 4.1 子项 · 会议纪要结构化提取模板 · `_templates/会议纪要结构化提取模板v0.1.md`(9/9 commit e34d7a1)
- [x] 第 4 项 · 4.2 子项 · 飞书会议 API 接入测试 · `_rules/飞书会议API接入测试v0.1.md`(9/10 commit f6eb47d)
- [x] 桥接 · Phase 0 → Phase 1 会议纪要数据流设计 · `_rules/Phase1会议纪要数据流设计v0.1.md`(9/11 commit 8aaa936)
- [x] 桥接扩展 · LLM 5 段式抽取 prompt 样例 · `_templates/LLM会议纪要5段式抽取prompt样例v0.1.md`(9/14 commit 9a1e288)
- [x] 桥接扩展 · 行动项 → 飞书任务映射 · `_rules/行动项转飞书任务映射v0.1.md`(9/15 commit dc2ceeb)
- [x] 桥接扩展 · Phase 1 第 1 项阻塞清单 · `_rules/Phase1第1项阻塞清单v0.1.md`(9/16 commit TBD)

### Phase 0 未做主项(2 项,T2/T3 决策)
- [ ] 第 2 项主任务 · 整理公司过去 1 年所有周会/月会纪要(需张勇签发公司纪要访问权限)
- [ ] 第 4 项主任务 · 飞书会议 API 真机凭证接入(需张勇签发 appId/appSecret + 租户管理员授权)

### Phase 1 MVP(0/6,等真机凭证)
- [ ] 上线会议纪要自动生成(周会/月会/项目会 3 类)
- [ ] 上线流程梳理器(口述→流程图,Mermaid 导出)
- [ ] 上线团队健康度诊断 v1(周报推送)
- [ ] 飞书任务打通(纪要行动项自动转任务)
- [ ] 管理仪表盘 Web 端 v1(团队/流程/会议三视图)
- [ ] 内部 dogfood · 3 个团队周会跑 4 周

### 桥接设计 · Phase 1 启动 6 步 checklist
详见 `_rules/Phase1会议纪要数据流设计v0.1.md` §六
1. 4.2 §4 真机凭证 6 步接入(预计 9/22 后)— **阻塞清单见 `_rules/Phase1第1项阻塞清单v0.1.md`**
2. 字段映射表在 4 API 真实响应上验证(依赖步骤 1)
3. 节点 4 说话人映射 API(directory.user.get)权限补充申请(依赖步骤 1)
4. 节点 6 LLM 5 段式抽取 prompt 调试 ✓ (9/14 commit 9a1e288)
5. 节点 8 行动项 → 飞书任务映射表沉淀 ✓ (9/15 commit dc2ceeb · `_rules/行动项转飞书任务映射v0.1.md`)
6. dogfood 3 团队周会跑 4 周(依赖步骤 1-5 + 真实测试租户)

**6 步进度**:5/6 完成(步骤 4/5 已落库,步骤 1-3 阻塞于真机凭证,步骤 6 阻塞于真实环境)

### Phase 1 第 1 项前置资产链(6 块齐备)
4.1 输入格式契约 + 4.2 数据来源契约 + 桥接 端到端数据流 + prompt 样例 + 行动项转任务映射 + 阻塞清单 = 6 件套齐备,真机凭证签发后 T2/T3 可直接落地编码

### Phase 1 第 1 项阻塞清单(9-16 入库)
详见 `_rules/Phase1第1项阻塞清单v0.1.md` §二
- 阻塞 1 · 飞书会议 API 真机凭证(张勇侧 appId/appSecret + 5 权限勾选 + 租户管理员授权)
- 阻塞 2 · directory.user.get 说话人映射 API 权限补充申请
- 阻塞 3 · 真实飞书测试租户(dogfood 4 周用)
- 关键路径总耗时:约 4-6 周(dogfood 4 周占大头)

## 关键路径
- Phase 0 第 1 项 → Phase 1 第 1 项「会议纪要自动生成」前置链路已闭环:
  4.1(输入格式契约) + 4.2(数据来源契约) + 桥接(端到端数据流) = 三件套齐备
- 阻塞:真机凭证(appId/appSecret + 租户授权),由张勇签发后 T2/T3 推进
- 阻塞详情:`_rules/Phase1第1项阻塞清单v0.1.md`(9-16 入库)

## 关联文档
- 项目立项与计划:`项目开发计划.md`
- 架构与方案设计:`管理顾问开发架构与计划.md`
- 巡检日志:`.Log/巡检-管理-YYYYMMDD.md`(8/24 立项以来连续 23 天)
