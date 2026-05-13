# 科研级文献智能系统设计规范 v1

本文档把“文献自动调研 + 专业知识库构建 + 严格来源引用 + 零无源断言策略”固化为工程可执行规范。目标不是让系统回答更多问题，而是让每个最终结论都可复核、可审计、可回归评测。

## 1. 固化结论

当前设计可以固化为 **v1 工程基线**，原因如下：

1. 已明确产品边界：面向科研和专业知识库，不做泛用笔记工具。
2. 已明确核心差异化：严格证据绑定、断言级验证、冲突识别、审计可复现。
3. 已明确发布门禁：P0 指标不达标不得发布科研可用版本。
4. 已明确数据骨架：文献、span、claim、evidence、conflict、audit trace。
5. 已明确近期路线：先实现证据闭环和评测闭环，再扩展高级推理能力。

v1 固化不代表设计永久不变，而是代表团队可以停止发散，进入实现、评测和专家复核阶段。

## 2. 设计原则

1. **Evidence-first**：任何最终回答必须先形成证据包，再生成结论。
2. **No Unsourced Claim**：最终输出中的每条断言必须绑定至少一条有效证据。
3. **Citable by Construction**：引用不是后处理装饰，而是 claim 生成、验证、渲染的必需字段。
4. **Abstain over Hallucinate**：证据不足、冲突过强、引用不完整时必须拒答或降级为证据列表。
5. **Reproducible Research**：同一输入、同一资料版本、同一策略版本应可复现结果。
6. **Expert-in-the-loop**：高风险结论、冲突结论、低置信结论需要专家复核入口。

## 3. 系统范围

### 3.1 必做范围

- 多源文献接入与去重。
- PDF/HTML/摘要文本解析，保留页码、章节、字符范围和版本信息。
- 混合检索和证据重排。
- claim 抽取、claim-evidence 绑定和 citation entailment 评分。
- 无源断言阻断、证据不足拒答、冲突证据标记。
- 结构化专业知识库：paper、span、claim、evidence、conflict、review。
- 全链路 audit trace、发布门禁和专家复核闭环。

### 3.2 暂不纳入 v1 范围

- 追求通用 Notebook 或聊天产品体验。
- 无证据开放式创作。
- 未经评测的多代理复杂辩论。
- 没有 citation span 的长篇报告自动发布。
- 对外承诺“绝对杜绝所有幻觉”。

## 4. P0 发布门禁指标

P0 指标是科研可用版本发布前必须达标的硬门槛。任何一项失败，都不应对外宣称“严格引用”或“低幻觉”。

| 指标 | 定义 | 建议阈值 | 失败处置 |
|---|---|---:|---|
| Unsourced Claim Rate | 最终输出中无有效 citation 的 claim 占比 | < 0.5% | 阻断发布，开启无源断言错误复盘 |
| Citation Precision | citation span 是否真的支持对应 claim | > 98% | 阻断发布，修复 citation entailment |
| Critical Claim Dual-source Coverage | 关键结论是否被两个独立来源支持 | > 90% | 关键结论降级为“证据不足” |
| Retrieval Recall@20 | 金标证据是否出现在召回 Top 20 | > 85% | 调整检索、重排、query expansion |
| Conflict Recall | 金标冲突证据是否被识别 | > 90% | 冲突问题禁止单边结论 |
| Abstention Accuracy | 证据不足题是否正确拒答 | > 95% | 收紧生成前证据门控 |
| Trace Completeness | 输出是否具备完整 trace_id 与步骤日志 | 100% | 输出降级为不可发布草案 |
| Reproducibility | 同版本重复运行的结构化 claim 一致率 | > 95% | 固定模型、提示词、索引与策略版本 |

## 5. 核心数据模型

### 5.1 文献与片段

```sql
create table papers (
  id uuid primary key,
  source text not null,
  external_id text,
  doi text,
  title text not null,
  authors jsonb not null default '[]'::jsonb,
  journal text,
  published_at date,
  ingested_at timestamptz not null default now(),
  content_hash text not null,
  version integer not null default 1,
  metadata jsonb not null default '{}'::jsonb,
  unique (source, external_id, version)
);

create table paper_spans (
  id uuid primary key,
  paper_id uuid not null references papers(id),
  section text,
  page_start integer,
  page_end integer,
  char_start integer not null,
  char_end integer not null,
  text text not null,
  text_hash text not null,
  parse_quality jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now(),
  check (char_end > char_start)
);
```

### 5.2 Claim、证据与冲突

```sql
create table claims (
  id uuid primary key,
  project_id uuid not null,
  trace_id uuid not null,
  claim_text text not null,
  claim_type text not null check (claim_type in ('fact', 'causal', 'comparative', 'methodological', 'recommendation')),
  importance text not null check (importance in ('critical', 'normal', 'minor')),
  status text not null check (status in ('supported', 'conflicted', 'insufficient', 'rejected')),
  confidence numeric not null check (confidence >= 0 and confidence <= 1),
  created_by text not null,
  created_at timestamptz not null default now()
);

create table claim_evidence (
  claim_id uuid not null references claims(id),
  span_id uuid not null references paper_spans(id),
  stance text not null check (stance in ('support', 'refute', 'neutral')),
  entailment_score numeric not null check (entailment_score >= 0 and entailment_score <= 1),
  source_quality_score numeric not null check (source_quality_score >= 0 and source_quality_score <= 1),
  rationale text not null,
  primary key (claim_id, span_id, stance)
);

create table claim_conflicts (
  id uuid primary key,
  project_id uuid not null,
  claim_a uuid not null references claims(id),
  claim_b uuid not null references claims(id),
  conflict_type text not null check (conflict_type in ('direct_contradiction', 'scope_mismatch', 'methodological_disagreement', 'temporal_drift')),
  severity numeric not null check (severity >= 0 and severity <= 1),
  explanation text not null,
  created_at timestamptz not null default now()
);
```

### 5.3 审计日志

```sql
create table audit_events (
  id uuid primary key,
  trace_id uuid not null,
  project_id uuid not null,
  step text not null check (step in ('plan', 'retrieve', 'rank', 'extract_claims', 'verify_claims', 'detect_conflicts', 'render', 'expert_review')),
  input_hash text not null,
  output_hash text not null,
  model_name text,
  model_version text,
  prompt_version text,
  index_version text,
  policy_version text not null,
  latency_ms integer,
  token_usage jsonb not null default '{}'::jsonb,
  metrics jsonb not null default '{}'::jsonb,
  created_at timestamptz not null default now()
);
```

## 6. P0 指标 SQL 口径模板

以下 SQL 是指标口径模板，团队可按真实 schema 微调。

### 6.1 无源断言率

```sql
select
  count(*) filter (where ce.claim_id is null)::numeric / nullif(count(*), 0) as unsourced_claim_rate,
  count(*) filter (where ce.claim_id is null) as unsourced_claims,
  count(*) as total_claims
from claims c
left join claim_evidence ce
  on ce.claim_id = c.id
 and ce.stance = 'support'
 and ce.entailment_score >= 0.80
where c.status in ('supported', 'conflicted')
  and c.created_at >= now() - interval '7 days';
```

### 6.2 引用准确率

```sql
select
  avg(case when human_label = 'supports' then 1 else 0 end) as citation_precision,
  count(*) as reviewed_citations
from citation_review_labels
where reviewed_at >= now() - interval '7 days'
  and sampled_from = 'release_gate';
```

### 6.3 关键结论双来源覆盖率

```sql
with support_sources as (
  select
    c.id as claim_id,
    count(distinct ps.paper_id) filter (
      where ce.stance = 'support' and ce.entailment_score >= 0.80
    ) as independent_sources
  from claims c
  left join claim_evidence ce on ce.claim_id = c.id
  left join paper_spans ps on ps.id = ce.span_id
  where c.importance = 'critical'
    and c.created_at >= now() - interval '7 days'
  group by c.id
)
select
  count(*) filter (where independent_sources >= 2)::numeric / nullif(count(*), 0) as dual_source_coverage,
  count(*) filter (where independent_sources < 2) as under_supported_critical_claims,
  count(*) as critical_claims
from support_sources;
```

### 6.4 Trace 完整率

```sql
with required_steps(step) as (
  values ('plan'), ('retrieve'), ('rank'), ('extract_claims'), ('verify_claims'), ('render')
), traces as (
  select distinct trace_id
  from claims
  where created_at >= now() - interval '7 days'
), trace_step_counts as (
  select
    t.trace_id,
    count(distinct ae.step) filter (where ae.step in (select step from required_steps)) as present_steps
  from traces t
  left join audit_events ae on ae.trace_id = t.trace_id
  group by t.trace_id
)
select
  count(*) filter (where present_steps = (select count(*) from required_steps))::numeric / nullif(count(*), 0) as trace_completeness,
  count(*) filter (where present_steps < (select count(*) from required_steps)) as incomplete_traces,
  count(*) as total_traces
from trace_step_counts;
```

## 7. 每周评测看板字段

### 7.1 可靠性看板

| 字段 | 说明 | 负责人 |
|---|---|---|
| Unsourced Claim Rate | 无源断言率，按产品线和领域拆分 | Research QA Lead |
| Citation Precision | 人工抽检 citation 是否支持 claim | Research QA Lead |
| Citation Entailment Score p50/p90 | 自动引用蕴含评分分布 | Evidence Engineer |
| Critical Dual-source Coverage | 关键结论双来源覆盖率 | Product Owner |
| Abstention Accuracy | 证据不足时是否拒答 | Research QA Lead |

### 7.2 检索看板

| 字段 | 说明 | 负责人 |
|---|---|---|
| Recall@5/10/20 | 金标证据召回率 | Evidence Engineer |
| nDCG@10 | 证据排序质量 | Evidence Engineer |
| Source Diversity@10 | 前 10 条结果来源多样性 | Evidence Engineer |
| Freshness SLA | 新文献入库后可检索耗时 | Backend Lead |
| Parser Warning Rate | OCR、页码、表格解析警告率 | Data Pipeline Lead |

### 7.3 专家复核看板

| 字段 | 说明 | 负责人 |
|---|---|---|
| Expert Acceptance Rate | 专家无需修改直接采纳比例 | Domain Editor |
| Mean Edit Distance | 专家修订幅度 | Domain Editor |
| High-risk Escalation Rate | 高风险结论进入专家复核比例 | Product Owner |
| Error Taxonomy Top 10 | 本周错误类型排行 | Research QA Lead |
| Regression Count | 相比上一版本退化用例数 | QA Lead |

## 8. 发布流程

1. **冻结版本**：固定模型、提示词、检索索引、policy 与数据快照版本。
2. **离线回归**：运行 100-300 题金标集，生成指标报告。
3. **红队测试**：覆盖伪造引用、过期证据、冲突证据、提示词注入。
4. **专家抽检**：至少抽检 20 个高风险或高影响输出。
5. **发布裁决**：P0 全绿才可发布；P0 任一失败必须回滚或降级。

## 9. 错误分级与处置

| 等级 | 示例 | 处置 |
|---|---|---|
| S0 | 编造不存在 DOI、文献、页码并作为关键依据 | 立即阻断发布，创建事故复盘 |
| S1 | citation 存在但不支持关键 claim | 阻断发布，修复验证器或重排策略 |
| S2 | 证据不足却输出确定性结论 | 阻断高风险场景，调整拒答策略 |
| S3 | 引用格式错误但可人工定位来源 | 不阻断发布，进入修复队列 |
| S4 | 摘要风格、表达冗余、排序体验问题 | 常规迭代 |

## 10. 30 天最佳执行计划

| 周期 | 目标 | 交付物 |
|---|---|---|
| 第 1 周 | 锁定评测与日志口径 | P0 指标 SQL、审计日志 schema、100 题金标模板 |
| 第 2 周 | 打通证据绑定闭环 | claim extractor、citation binder、无源断言检测 |
| 第 3 周 | 增强可靠性门控 | citation entailment、冲突状态、拒答策略 |
| 第 4 周 | 建立发布门禁 | 周报看板、红队用例、专家抽检流程 |

## 11. 对外表述建议

建议使用：

> 面向科研与专业知识库构建的证据约束型文献智能系统。系统执行零无源断言策略，关键结论可追溯到明确文献片段，并通过检索、引用、冲突和审计指标持续评测。

避免使用绝对化承诺：

> 彻底杜绝所有幻觉。

更准确的承诺是：

> 系统性降低幻觉，并使无源断言可检测、可阻断、可审计、可复盘。
