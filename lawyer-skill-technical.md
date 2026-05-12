# Lawyer Skill 技术文档

> **版本**: v1.1.0  
> **文档类型**: 技术规格文档  
> **目标读者**: 开发者、Skill 维护者、系统集成人员

---

## 1. 概述

### 1.1 Skill 定位
Lawyer Skill 是一款面向中国法律领域的专业 AI 助手 Skill，提供法律法规查询、案例分析、合同审查、法律意见书起草、行为可行性评估等法律服务。核心原则是**"每一句话有法可依"**。

### 1.2 技术架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Lawyer Skill (v1.1)                     │
├─────────────────────────────────────────────────────────────┤
│  核心层  │  法律引用引擎 · 联网验证模块 · 风险评估引擎        │
├──────────┼──────────────────────────────────────────────────┤
│  功能层  │  +search-law · +search-case · +contract-review   │
│          │  +legal-opinion · +verify-validity · +legal-assess│
├──────────┼──────────────────────────────────────────────────┤
│  数据层  │  国家法律法规数据库 · 裁判文书网 · 北大法宝        │
│          │  司法解释库 · 指导性案例库                        │
├──────────┼──────────────────────────────────────────────────┤
│  工具层  │  WebSearch · WebFetch · Markdown 渲染            │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 元数据规格

### 2.1 Skill 元数据 (YAML Front Matter)

```yaml
---
name: lawyer
version: 1.1.0
description: "专业律师助手：提供中国法律法规查询、司法解释检索、案例分析、合同审查建议、法律意见书起草、行为可行性评估等法律服务。所有法律建议均基于现行有效法律法规，每一句话有法可依，支持联网查询最新法律动态和判例。"
metadata:
  requires:
    tools: ["WebSearch", "WebFetch"]
  capabilities: ["legal_research", "contract_review", "case_analysis", "legal_opinion", "legal_assess"]
---
```

### 2.2 字段说明

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | Skill 唯一标识符 |
| `version` | string | 语义化版本号 (MAJOR.MINOR.PATCH) |
| `description` | string | Skill 功能描述（用于 AI 路由决策） |
| `metadata.requires.tools` | array | 依赖的工具列表 |
| `metadata.capabilities` | array | 能力标签列表 |

---

## 3. 核心功能模块

### 3.1 功能矩阵

| Shortcut | 能力标签 | 依赖工具 | 复杂度 |
|----------|----------|----------|--------|
| `+search-law` | `legal_research` | WebSearch | 低 |
| `+search-case` | `case_analysis` | WebSearch | 中 |
| `+contract-review` | `contract_review` | WebSearch | 高 |
| `+legal-opinion` | `legal_opinion` | WebSearch | 高 |
| `+verify-validity` | `legal_research` | WebSearch | 低 |
| `+legal-assess` | `legal_assess` | WebSearch | 高 |

### 3.2 功能详细规格

#### 3.2.1 `+search-law` 法律法规检索

**输入参数**:
```typescript
interface SearchLawInput {
  keyword: string;           // 检索关键词
  lawName?: string;          // 特定法律名称（可选）
  articleNumber?: string;    // 条款编号（可选）
  effectiveOnly?: boolean;   // 是否仅查询现行有效（默认 true）
}
```

**输出格式**:
```typescript
interface SearchLawOutput {
  lawName: string;
  effectiveDate: string;
  articles: Array<{
    number: string;
    content: string;
    interpretation?: string;
  }>;
  relatedLaws: string[];
}
```

**工作流程**:
1. 解析用户查询意图
2. 构造 WebSearch 查询语句
3. 验证来源权威性（优先 flk.npc.gov.cn）
4. 提取并格式化法条内容
5. 标注效力层级和关联法条

---

#### 3.2.2 `+search-case` 案例检索

**输入参数**:
```typescript
interface SearchCaseInput {
  caseType?: string;         // 案由
  keywords: string[];        // 关键词列表
  courtLevel?: string;       // 法院层级（最高法/高院/中院/基层）
  timeRange?: {              // 时间范围
    start: string;
    end: string;
  };
}
```

**输出格式**:
```typescript
interface SearchCaseOutput {
  cases: Array<{
    title: string;
    caseNumber: string;
    court: string;
    date: string;
    result: string;
    holding: string;
    legalBasis: string[];
  }>;
  rules: string[];           // 裁判规则总结
}
```

---

#### 3.2.3 `+contract-review` 合同审查

**输入参数**:
```typescript
interface ContractReviewInput {
  contractType: string;      // 合同类型
  contractText: string;      // 合同文本
  focusAreas?: string[];     // 重点关注领域（可选）
}
```

**审查维度**:
| 维度 | 检查项 | 法律依据来源 |
|------|--------|-------------|
| 主体资格 | 民事行为能力、特殊资质 | 《民法典》总则编 |
| 合同形式 | 书面形式要求、签字盖章 | 《民法典》合同编 |
| 标的条款 | 合法性、明确性 | 相关特别法 |
| 价款条款 | 金额、支付方式、税费 | 《民法典》+ 税法 |
| 履行条款 | 期限、地点、方式 | 《民法典》合同编 |
| 违约责任 | 违约金、赔偿范围 | 《民法典》第585-594条 |
| 争议解决 | 管辖、仲裁/诉讼 | 《民事诉讼法》《仲裁法》 |

---

#### 3.2.4 `+legal-opinion` 法律意见书

**输入参数**:
```typescript
interface LegalOpinionInput {
  client: string;            // 委托方
  subject: string;           // 事项主题
  facts: string;             // 事实背景
  questions: string[];       // 法律问题列表
  materials?: string[];      // 参考材料
}
```

**输出结构**:
```
法律意见书
├── 引言（委托、材料、依据、声明）
├── 事实背景
├── 法律分析（逐条分析，每条附法律依据）
├── 结论意见
├── 风险提示
└── 建议
```

---

#### 3.2.5 `+verify-validity` 有效性验证

**输入参数**:
```typescript
interface VerifyValidityInput {
  lawName: string;           // 法规名称
  version?: string;          // 特定版本（可选）
}
```

**输出信息**:
- 当前状态（现行有效/已废止/已修订）
- 效力变迁时间线
- 替代法规（如适用）
- 验证来源和日期

---

#### 3.2.6 `+legal-assess` 行为可行性评估 ⭐

**核心创新点**: 采用**渐进式提问**模式，而非一次性分析

**状态机模型**:
```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  初始态  │ → │ 行为识别 │ → │ 主体提问 │ → │ 细节提问 │ → │ 联网验证 │ → │ 出具报告 │
└─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘    └─────────┘
     ↑                                                                            │
     └──────────────────────── 用户补充信息后可循环 ────────────────────────────────┘
```

**提问框架**:

| 阶段 | 提问维度 | 法律意义 |
|------|---------|---------|
| 主体与关系 | 自然人/法人、资质、代理关系 | 确定责任主体和适用法律 |
| 行为细节 | 时间、地点、方式、金额、目的 | 影响法律定性和管辖 |
| 合同/协议 | 书面/口头、主要条款 | 确定合同关系和内容 |
| 后果与争议 | 已发生后果、争议焦点 | 影响责任承担和救济途径 |

**输出模板**:
```typescript
interface LegalAssessOutput {
  behavior: string;          // 行为描述
  legalNature: string;       // 法律定性
  feasibility: {
    legal: string[];         // 合法部分
    caution: string[];       // 需注意部分
    illegal: string[];       // 违法部分
  };
  risks: Array<{
    level: 'high' | 'medium' | 'low';
    description: string;
    legalBasis: string;
    consequence: string;
  }>;
  supports: string[];        // 有利于用户的法律依据
  cases: Array<{
    title: string;
    caseNumber: string;
    holding: string;
  }>;
  recommendations: string[];
}
```

---

## 4. 法律引用规范

### 4.1 引用格式标准

```
【效力层级】《法律名称》（施行日期：XXXX年XX月XX日）第X条第X款："法条原文"
```

### 4.2 效力层级标注

| 标注 | 制定机关 | 示例 |
|------|---------|------|
| 【法律】 | 全国人大及其常委会 | 《民法典》《劳动合同法》 |
| 【行政法规】 | 国务院 | 《公司登记管理条例》 |
| 【司法解释】 | 最高法、最高检 | 《关于审理民间借贷案件适用法律若干问题的规定》 |
| 【部门规章】 | 国务院各部委 | 《市场监督管理行政处罚程序规定》 |
| 【地方性法规】 | 地方人大 | 《北京市生活垃圾管理条例》 |

### 4.3 引用验证流程

```
用户查询
    ↓
内部知识检索（训练数据）
    ↓
标注可能的时效性风险
    ↓
WebSearch 验证现行有效性
    ↓
确认或修正法律依据
    ↓
输出带引用的分析结果
```

---

## 5. 数据源配置

### 5.1 官方法律数据库

| 数据源 | URL | 优先级 | 用途 |
|--------|-----|--------|------|
| 国家法律法规数据库 | https://flk.npc.gov.cn/ | P0 | 法律、行政法规查询 |
| 中国裁判文书网 | https://wenshu.court.gov.cn/ | P0 | 案例检索 |
| 最高人民法院官网 | https://www.court.gov.cn/ | P1 | 司法解释、指导案例 |
| 司法部网站 | https://www.moj.gov.cn/ | P1 | 行政法规、部门规章 |
| 北大法宝 | https://www.pkulaw.com/ | P2 | 综合法律检索 |
| 中国法院网 | https://www.chinacourt.org/ | P2 | 法院新闻、案例 |

### 5.2 搜索策略

```javascript
// 法律法规搜索模板
const lawSearchTemplate = (lawName, article) => 
  `"${lawName}" "第${article}条" 现行有效 site:flk.npc.gov.cn`;

// 案例搜索模板
const caseSearchTemplate = (keywords) => 
  `"${keywords.join(' ')}" 裁判文书网 判决书`;

// 司法解释搜索模板
const judicialSearchTemplate = (topic) => 
  `"${topic}" 司法解释 最高人民法院 最新`;
```

---

## 6. 文件结构

```
lawyer/
├── SKILL.md                              # 主技能文件（元数据 + 功能索引）
└── references/                           # 参考文档目录
    ├── lawyer-search-law.md              # 法律法规检索指南
    ├── lawyer-search-case.md             # 案例检索指南
    ├── lawyer-contract-review.md         # 合同审查指南
    ├── lawyer-legal-opinion.md           # 法律意见书起草指南
    ├── lawyer-verify-validity.md         # 有效性验证指南
    └── lawyer-legal-assess.md            # 行为可行性评估指南 ⭐
```

### 6.1 文件命名规范

- 主文件: `SKILL.md`
- 参考文档: `{skill-name}-{shortcut-name}.md`
- 使用小写字母和连字符

### 6.2 参考文档结构

每个参考文档应包含:
1. 功能说明
2. 使用场景
3. 输入/输出规格
4. 工作流程
5. 示例
6. 注意事项

---

## 7. 约束与限制

### 7.1 法律免责声明

**强制性要求**: 所有输出必须包含以下免责声明:

```
【免责声明】本 Skill 提供的所有法律信息仅供参考，不构成正式法律意见。具体法律问题请咨询执业律师。
```

### 7.2 联网查询强制场景

以下情况**必须**使用 WebSearch:

| 场景 | 原因 |
|------|------|
| 2020年后的新法新规 | 《民法典》及相关配套规定变动频繁 |
| 司法解释最新修订 | 司法解释更新较快 |
| 特定地区地方法规 | 各地规定差异大 |
| 正在审议的法律草案 | 尚未生效，需确认状态 |
| 用户询问"最新""最近变化" | 时效性要求高 |

### 7.3 输出规范检查清单

- [ ] 每条法律依据标注出处和条款编号
- [ ] 区分强制性规定和任意性规定
- [ ] 提示法律适用的地域限制
- [ ] 使用【法律依据】【风险提示】【参考案例】标签
- [ ] 包含免责声明

---

## 8. 版本历史

| 版本 | 日期 | 变更内容 |
|------|------|---------|
| v1.0.0 | - | 初始版本，包含 5 个基础功能 |
| v1.1.0 | 2026-05-12 | 新增 `+legal-assess` 行为可行性评估功能 |

---

## 9. 附录

### 9.1 常见法律领域映射

| 领域 | 主要法律 | 典型场景 |
|------|---------|---------|
| 民事法律 | 《民法典》 | 合同、侵权、婚姻、继承 |
| 商事法律 | 《公司法》《商标法》《专利法》 | 公司事务、知识产权 |
| 劳动法律 | 《劳动合同法》《劳动法》 | 劳动合同、社保、工伤 |
| 刑事法律 | 《刑法》《刑事诉讼法》 | 罪名认定、量刑、辩护 |
| 行政法律 | 《行政处罚法》《行政许可法》 | 行政处罚、复议、诉讼 |

### 9.2 时效性关键节点

| 日期 | 事件 | 影响 |
|------|------|------|
| 2021-01-01 | 《民法典》施行 | 《合同法》等9部法律废止 |
| 2024-07-01 | 新《公司法》施行 | 公司法律制度重大调整 |

---

*文档结束*
