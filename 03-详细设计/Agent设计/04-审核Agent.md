# Reviewer Agent (审核员) 详细设计

## 1. Agent概述

### 1.1 职责定义

审核员Agent负责对生成的标书内容进行多维度质量审核，确保内容完整性、逻辑一致性、风格统一性。

### 1.2 审核维度

```
┌─────────────────────────────────────────────────────────────┐
│                    Reviewer Agent 审核维度                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   📊 内容完整性 (30%)                                        │
│      - 评分点覆盖率                                         │
│      - 内容充实度                                           │
│      - 关键信息遗漏检查                                     │
│                                                             │
│   🔗 逻辑一致性 (25%)                                        │
│      - 章节间逻辑连贯                                       │
│      - 论证完整性                                           │
│      - 无矛盾冲突                                           │
│                                                             │
│   🎨 风格统一性 (20%)                                        │
│      - 术语使用一致                                         │
│      - 语气风格统一                                         │
│      - 格式规范一致                                         │
│                                                             │
│   📐 格式规范性 (15%)                                        │
│      - 标书格式要求                                         │
│      - 排版规范                                             │
│      - 编号正确                                             │
│                                                             │
│   ✍️ 语言质量 (10%)                                          │
│      - 表达准确                                             │
│      - 无AI痕迹                                             │
│      - 专业术语正确                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 接口设计

### 2.1 输入接口

```typescript
interface ReviewerTaskInput {
  chapters: ChapterContent[];
  outline: Outline;
  context: SharedContextData;
  options?: {
    strict_mode?: boolean;        // 严格模式
    auto_fix?: boolean;           // 自动修复
    min_score?: number;           // 最低通过分数，默认70
  };
}
```

### 2.2 输出接口

```typescript
interface ReviewReport {
  id: string;
  bid_id: string;
  overall_score: number;          // 总体评分 0-100
  passed: boolean;                // 是否通过
  dimensions: ReviewDimension[];  // 各维度评分
  issues: ReviewIssue[];          // 问题列表
  suggestions: string[];          // 改进建议
  created_at: Date;
}

interface ReviewDimension {
  name: string;                   // 维度名称
  score: number;                  // 维度评分 0-100
  weight: number;                 // 权重
  issues: string[];               // 该维度的问题
}

interface ReviewIssue {
  id: string;
  chapter_id: string;
  type: IssueType;
  severity: 'high' | 'medium' | 'low';
  description: string;
  location?: string;              // 问题位置
  suggestion: string;             // 修改建议
}

enum IssueType {
  MISSING_CONTENT = 'missing_content',
  INCONSISTENT_LOGIC = 'inconsistent_logic',
  STYLE_MISMATCH = 'style_mismatch',
  FORMAT_ERROR = 'format_error',
  LANGUAGE_ISSUE = 'language_issue',
  TERMINOLOGY_ERROR = 'terminology_error',
  REFERENCE_ERROR = 'reference_error'
}
```

---

## 3. 核心流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Reviewer Agent 处理流程                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入: Chapters[], Outline, SharedContext                   │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ① 内容完整性审核                                      │  │
│   │    - 检查评分点覆盖率                                  │  │
│   │    - 检查内容充实度                                    │  │
│   │    - 识别关键信息遗漏                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ② 逻辑一致性审核                                      │  │
│   │    - 检查章节间逻辑连贯                                │  │
│   │    - 识别矛盾冲突                                      │  │
│   │    - 验证论证完整性                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ③ 风格统一性审核                                      │  │
│   │    - 检查术语使用一致性                                │  │
│   │    - 检查语气风格统一                                  │  │
│   │    - 检查格式规范一致                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ④ 格式规范性审核                                      │  │
│   │    - 检查标书格式要求                                  │  │
│   │    - 检查排版规范                                      │  │
│   │    - 检查编号正确性                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑤ 语言质量审核                                        │  │
│   │    - 检查表达准确性                                    │  │
│   │    - 识别AI生成痕迹                                    │  │
│   │    - 检查专业术语正确性                                │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑥ 生成审核报告                                        │  │
│   │    - 计算各维度评分                                    │  │
│   │    - 汇总问题列表                                      │  │
│   │    - 生成改进建议                                      │  │
│   │    - 判断是否通过                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   输出: ReviewReport                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Prompt设计

### 4.1 系统提示词

```typescript
const REVIEWER_SYSTEM_PROMPT = `你是一个严格的标书质量审核员，拥有丰富的投标文件审核经验。

## 审核原则
- 审核要客观、公正、全面
- 问题要具体、可定位、可修改
- 建议要实用、可执行
- 评分要合理、有依据

## 审核维度
1. 内容完整性 (30%) - 评分点覆盖、内容充实度
2. 逻辑一致性 (25%) - 章节间逻辑、论证完整性
3. 风格统一性 (20%) - 术语一致、语气统一
4. 格式规范性 (15%) - 标书格式、排版规范
5. 语言质量 (10%) - 表达准确、无AI痕迹

## 输出要求
- 使用结构化JSON格式
- 问题描述要具体，包含位置信息
- 修改建议要可执行
- 评分要合理有依据`;
```

### 4.2 内容完整性审核提示词

```typescript
const COMPLETENESS_REVIEW_PROMPT = `请审核以下标书内容的完整性：

## 大纲要求
{outline_json}

## 评分标准
{scoring_criteria}

## 生成内容
{chapters_content}

请检查：
1. 是否所有大纲章节都有对应内容
2. 内容是否覆盖了所有评分点
3. 内容是否充实，有无遗漏
4. 各章节字数是否合理

返回JSON格式：
{
  "score": 85,
  "coverage_rate": 0.95,
  "issues": [
    {
      "chapter_id": "1.1",
      "type": "missing_content",
      "severity": "high",
      "description": "缺少xxx内容",
      "suggestion": "建议补充xxx"
    }
  ]
}`;
```

### 4.3 逻辑一致性审核提示词

```typescript
const CONSISTENCY_REVIEW_PROMPT = `请审核以下标书内容的逻辑一致性：

## 章节内容
{chapters_content}

## 项目概述
{project_overview}

请检查：
1. 章节间逻辑是否连贯
2. 是否存在矛盾或冲突
3. 论证是否完整
4. 引用是否正确

返回JSON格式：
{
  "score": 90,
  "issues": [
    {
      "chapter_ids": ["1.1", "2.1"],
      "type": "inconsistent_logic",
      "severity": "medium",
      "description": "xxx与yyy存在矛盾",
      "suggestion": "建议统一为zzz"
    }
  ]
}`;
```

### 4.4 风格统一性审核提示词

```typescript
const STYLE_REVIEW_PROMPT = `请审核以下标书内容的风格统一性：

## 章节内容
{chapters_content}

## 术语表
{terminology}

## 风格指南
{style_guide}

请检查：
1. 术语使用是否一致
2. 语气风格是否统一
3. 格式是否规范一致
4. 是否存在AI生成痕迹

返回JSON格式：
{
  "score": 88,
  "terminology_issues": [
    {
      "term": "xxx",
      "locations": ["1.1", "2.1"],
      "description": "术语使用不一致"
    }
  ],
  "style_issues": [
    {
      "chapter_id": "1.1",
      "description": "语气过于口语化"
    }
  ]
}`;
```

---

## 5. 实现代码

```typescript
import { BaseAgent, AgentType, AgentStatus } from '../base/BaseAgent';
import { AIService } from '../../services/AIService';
import { Logger } from '../../utils/Logger';

export class ReviewerAgent extends BaseAgent {
  constructor(
    aiService: AIService,
    logger: Logger
  ) {
    super({
      id: 'reviewer-agent',
      type: AgentType.REVIEWER,
      aiService,
      logger
    });
  }
  
  protected getSystemPrompt(): string {
    return REVIEWER_SYSTEM_PROMPT;
  }
  
  async execute(input: ReviewerTaskInput): Promise<ReviewReport> {
    this.updateStatus(AgentStatus.BUSY);
    const startTime = Date.now();
    
    try {
      this.logger.info('开始审核标书内容');
      
      const { chapters, outline, context, options } = input;
      const minScore = options?.min_score || 70;
      
      // 1. 并发执行各维度审核
      const [completeness, consistency, style, format, language] = 
        await Promise.all([
          this.reviewCompleteness(chapters, outline, context),
          this.reviewConsistency(chapters, context),
          this.reviewStyle(chapters, context),
          this.reviewFormat(chapters),
          this.reviewLanguage(chapters)
        ]);
      
      // 2. 汇总问题
      const allIssues = [
        ...completeness.issues,
        ...consistency.issues,
        ...style.issues,
        ...format.issues,
        ...language.issues
      ];
      
      // 3. 计算总体评分
      const overallScore = this.calculateOverallScore({
        completeness: completeness.score,
        consistency: consistency.score,
        style: style.score,
        format: format.score,
        language: language.score
      });
      
      // 4. 生成改进建议
      const suggestions = this.generateSuggestions(allIssues);
      
      // 5. 判断是否通过
      const passed = overallScore >= minScore && 
                     !allIssues.some(i => i.severity === 'high');
      
      const report: ReviewReport = {
        id: this.generateId(),
        bid_id: outline.bid_id,
        overall_score: overallScore,
        passed,
        dimensions: [
          { name: '内容完整性', score: completeness.score, weight: 0.3, issues: completeness.issueDescriptions },
          { name: '逻辑一致性', score: consistency.score, weight: 0.25, issues: consistency.issueDescriptions },
          { name: '风格统一性', score: style.score, weight: 0.2, issues: style.issueDescriptions },
          { name: '格式规范性', score: format.score, weight: 0.15, issues: format.issueDescriptions },
          { name: '语言质量', score: language.score, weight: 0.1, issues: language.issueDescriptions }
        ],
        issues: allIssues,
        suggestions,
        created_at: new Date()
      };
      
      this.updateStatus(AgentStatus.IDLE);
      this.logger.info(`审核完成，总分: ${overallScore}, 是否通过: ${passed}`);
      
      return report;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      this.logger.error(`审核失败: ${error.message}`);
      throw error;
    }
  }
  
  private async reviewCompleteness(
    chapters: ChapterContent[],
    outline: Outline,
    context: SharedContextData
  ): Promise<DimensionResult> {
    const messages = [
      {
        role: 'user',
        content: COMPLETENESS_REVIEW_PROMPT
          .replace('{outline_json}', JSON.stringify(outline.items, null, 2))
          .replace('{scoring_criteria}', JSON.stringify(context.scoringCriteria, null, 2))
          .replace('{chapters_content}', this.formatChaptersForPrompt(chapters))
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
  
  private async reviewConsistency(
    chapters: ChapterContent[],
    context: SharedContextData
  ): Promise<DimensionResult> {
    const messages = [
      {
        role: 'user',
        content: CONSISTENCY_REVIEW_PROMPT
          .replace('{chapters_content}', this.formatChaptersForPrompt(chapters))
          .replace('{project_overview}', context.projectOverview || '')
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
  
  private async reviewStyle(
    chapters: ChapterContent[],
    context: SharedContextData
  ): Promise<DimensionResult> {
    const messages = [
      {
        role: 'user',
        content: STYLE_REVIEW_PROMPT
          .replace('{chapters_content}', this.formatChaptersForPrompt(chapters))
          .replace('{terminology}', JSON.stringify(Array.from(context.terminology.entries()), null, 2))
          .replace('{style_guide}', JSON.stringify(context.styleGuide, null, 2))
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
  
  private calculateOverallScore(scores: Record<string, number>): number {
    const weights = {
      completeness: 0.3,
      consistency: 0.25,
      style: 0.2,
      format: 0.15,
      language: 0.1
    };
    
    let total = 0;
    for (const [dimension, score] of Object.entries(scores)) {
      total += score * (weights[dimension] || 0);
    }
    
    return Math.round(total);
  }
  
  private generateSuggestions(issues: ReviewIssue[]): string[] {
    const suggestions: string[] = [];
    
    // 按严重程度排序
    const sortedIssues = issues.sort((a, b) => {
      const severityOrder = { high: 0, medium: 1, low: 2 };
      return severityOrder[a.severity] - severityOrder[b.severity];
    });
    
    // 生成建议
    for (const issue of sortedIssues.slice(0, 10)) {
      suggestions.push(`[${issue.chapter_id}] ${issue.suggestion}`);
    }
    
    return suggestions;
  }
  
  private formatChaptersForPrompt(chapters: ChapterContent[]): string {
    return chapters.map(ch => 
      `### ${ch.chapter_id} ${ch.title}\n${ch.content}`
    ).join('\n\n---\n\n');
  }
}
```

---

## 6. 审核结果处理

### 6.1 通过条件

```typescript
function isReviewPassed(report: ReviewReport, minScore: number): boolean {
  // 总分必须达到最低要求
  if (report.overall_score < minScore) {
    return false;
  }
  
  // 不能有高严重度问题
  const highSeverityIssues = report.issues.filter(i => i.severity === 'high');
  if (highSeverityIssues.length > 0) {
    return false;
  }
  
  return true;
}
```

### 6.2 反馈给写手

```typescript
interface WriterFeedback {
  chapter_id: string;
  issues: ReviewIssue[];
  suggestions: string[];
  regenerate_requirement: string;
}

function generateFeedbackForWriter(
  report: ReviewReport,
  chapterId: string
): WriterFeedback {
  const chapterIssues = report.issues.filter(i => i.chapter_id === chapterId);
  
  const requirement = chapterIssues.map(i => i.suggestion).join('；');
  
  return {
    chapter_id: chapterId,
    issues: chapterIssues,
    suggestions: report.suggestions,
    regenerate_requirement: requirement
  };
}
```

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
