# Outliner Agent (大纲师) 详细设计

## 1. Agent概述

### 1.1 职责定义

大纲师Agent负责根据招标分析结果，生成符合评分标准的标书目录结构，确保目录与评分点完全对齐。

### 1.2 核心能力

```
┌─────────────────────────────────────────────────────────────┐
│                    Outliner Agent 核心能力                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   📋 评分点分析                                              │
│      - 识别评分大类                                         │
│      - 提取评分细项                                         │
│      - 计算分值权重                                         │
│                                                             │
│   🌳 目录生成                                                │
│      - 一级目录与评分大类对齐                               │
│      - 二三级目录覆盖评分细项                               │
│      - 自动生成章节编号                                     │
│                                                             │
│   ✅ 目录审核                                                │
│      - 完整性检查                                           │
│      - 一致性验证                                           │
│      - 格式规范化                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 接口设计

### 2.1 输入接口

```typescript
interface OutlinerTaskInput {
  analysis: TenderAnalysis;
  options?: {
    max_depth?: number;           // 最大目录层级，默认4
    align_with_scoring?: boolean; // 是否与评分点对齐，默认true
    user_outline?: OutlineItem[]; // 用户自定义大纲（可选）
  };
}
```

### 2.2 输出接口

```typescript
interface Outline {
  id: string;
  bid_id: string;
  items: OutlineItem[];
  version: number;
  metadata: OutlineMetadata;
}

interface OutlineItem {
  id: string;                    // 章节编号，如 "1", "1.1", "1.1.1"
  parent_id?: string;            // 父节点ID
  title: string;                 // 章节标题
  description: string;           // 章节描述
  level: number;                 // 层级 (1-4)
  order: number;                 // 排序序号
  requirement_ids: string[];     // 对应的评分点ID
  children?: OutlineItem[];      // 子节点
}

interface OutlineMetadata {
  total_chapters: number;
  max_depth: number;
  requirement_coverage: number;  // 评分点覆盖率
  created_at: Date;
}
```

---

## 3. 核心流程

### 3.1 处理流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    Outliner Agent 处理流程                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入: TenderAnalysis                                      │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ① 解析评分标准                                        │  │
│   │    - 提取评分大类                                      │  │
│   │    - 计算分值权重                                      │  │
│   │    - 确定优先级                                        │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ② 生成一级目录                                        │  │
│   │    - 与评分大类一一对应                                │  │
│   │    - 标题与评分大类名称一致                            │  │
│   │    - 记录对应的评分点ID                                │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ③ 并发生成二级目录                                    │  │
│   │    - 为每个一级目录生成二级目录                        │  │
│   │    - 覆盖对应评分大类的所有细项                        │  │
│   │    - 保持逻辑清晰                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ④ 并发生成三级目录                                    │  │
│   │    - 为每个二级目录生成三级目录                        │  │
│   │    - 内容具体、可执行                                  │  │
│   │    - 便于后续内容生成                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑤ 合并目录树                                          │  │
│   │    - 生成唯一编号                                      │  │
│   │    - 设置层级关系                                      │  │
│   │    - 排序和规范化                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑥ 审核目录                                            │  │
│   │    - 检查评分点覆盖率                                  │  │
│   │    - 验证目录结构合理性                                │  │
│   │    - 提出修改建议                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   输出: Outline                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Prompt设计

### 4.1 系统提示词

```typescript
const OUTLINER_SYSTEM_PROMPT = `你是一个专业的标书大纲设计专家，拥有丰富的投标文件编写经验。

## 职责
1. 根据招标文件的技术要求和评分标准，设计标书目录结构
2. 确保一级目录与评分大类对齐
3. 确保二级、三级目录覆盖所有评分细项
4. 保持目录层级清晰、逻辑合理

## 设计原则
- 一级目录名称应与评分大类一致
- 每个评分点都应有对应的目录章节
- 目录层级不超过4级
- 章节编号规范统一
- 目录标题专业、准确、简洁

## 输出要求
- 使用结构化JSON格式
- 包含完整的层级关系
- 记录对应的评分点ID
- 只返回JSON，不要输出其他内容`;
```

### 4.2 一级目录生成提示词

```typescript
const TOP_LEVEL_OUTLINE_PROMPT = `根据以下评分标准，生成标书的一级目录：

评分标准：
{scoring_criteria}

项目概述：
{project_overview}

要求：
1. 一级目录必须与评分大类一一对应
2. 目录名称与评分大类名称保持一致
3. 记录每个目录对应的评分点ID
4. 只返回一级目录，不要生成下级目录

返回JSON格式：
{
  "outline": [
    {
      "id": "1",
      "title": "目录标题",
      "description": "目录描述",
      "requirement_ids": ["S1"]
    }
  ]
}`;
```

### 4.3 子目录生成提示词

```typescript
const CHILDREN_OUTLINE_PROMPT = `为以下一级目录生成二级和三级目录：

一级目录：
- ID: {parent_id}
- 标题: {parent_title}
- 描述: {parent_description}

对应的评分细项：
{scoring_items}

项目概述：
{project_overview}

要求：
1. 二级目录要覆盖该评分大类下的所有细项
2. 三级目录要具体、可执行
3. 编号以父级ID为前缀（如父级是1，则二级从1.1开始）
4. 只返回当前一级目录下的子目录

返回JSON格式：
{
  "children": [
    {
      "id": "1.1",
      "title": "二级目录标题",
      "description": "二级目录描述",
      "requirement_ids": ["S1-1"],
      "children": [
        {
          "id": "1.1.1",
          "title": "三级目录标题",
          "description": "三级目录描述",
          "requirement_ids": ["S1-1"]
        }
      ]
    }
  ]
}`;
```

### 4.4 目录审核提示词

```typescript
const OUTLINE_REVIEW_PROMPT = `请审核以下标书目录是否符合要求：

目录结构：
{outline_json}

评分标准：
{scoring_criteria}

审核要求：
1. 检查是否所有评分点都有对应的目录章节
2. 检查一级目录是否与评分大类一致
3. 检查目录层级是否合理
4. 检查目录标题是否专业准确

返回JSON格式：
{
  "passed": true/false,
  "score": 0-100,
  "issues": [
    {
      "type": "missing_coverage|structure|naming",
      "description": "问题描述",
      "suggestion": "修改建议"
    }
  ],
  "requirement_coverage": 0.95
}`;
```

---

## 5. 实现代码

```typescript
import { BaseAgent, AgentType, AgentStatus } from '../base/BaseAgent';
import { AIService } from '../../services/AIService';
import { SharedContext } from '../../context/SharedContext';
import { Logger } from '../../utils/Logger';

export class OutlinerAgent extends BaseAgent {
  constructor(
    aiService: AIService,
    sharedContext: SharedContext,
    logger: Logger
  ) {
    super({
      id: 'outliner-agent',
      type: AgentType.OUTLINER,
      aiService,
      sharedContext,
      logger
    });
  }
  
  protected getSystemPrompt(): string {
    return OUTLINER_SYSTEM_PROMPT;
  }
  
  async execute(input: OutlinerTaskInput): Promise<Outline> {
    this.updateStatus(AgentStatus.BUSY);
    const startTime = Date.now();
    
    try {
      this.logger.info('开始生成大纲');
      
      const { analysis, options } = input;
      const maxDepth = options?.max_depth || 4;
      
      // 1. 解析评分标准
      const scoringGroups = this.parseScoringCriteria(analysis.scoring_criteria);
      this.logger.info(`解析到 ${scoringGroups.length} 个评分大类`);
      
      // 2. 生成一级目录
      const topLevel = await this.generateTopLevel(analysis, scoringGroups);
      this.logger.info(`生成 ${topLevel.length} 个一级目录`);
      
      // 3. 并发生成子目录
      const outlineItems = await this.generateChildrenRecursive(
        topLevel,
        analysis,
        maxDepth
      );
      
      // 4. 合并目录树
      const outline: Outline = {
        id: this.generateId(),
        bid_id: analysis.tender_doc_id,
        items: outlineItems,
        version: 1,
        metadata: {
          total_chapters: this.countChapters(outlineItems),
          max_depth: this.calculateMaxDepth(outlineItems),
          requirement_coverage: this.calculateCoverage(outlineItems, analysis),
          created_at: new Date()
        }
      };
      
      // 5. 审核目录
      const reviewResult = await this.reviewOutline(outline, analysis);
      if (!reviewResult.passed) {
        this.logger.warn(`目录审核未通过，问题: ${reviewResult.issues.length} 个`);
        // 可以选择自动修正或返回让用户确认
      }
      
      this.updateStatus(AgentStatus.IDLE);
      this.logger.info(`大纲生成完成，耗时: ${Date.now() - startTime}ms`);
      
      return outline;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      this.logger.error(`大纲生成失败: ${error.message}`);
      throw error;
    }
  }
  
  private async generateTopLevel(
    analysis: TenderAnalysis,
    scoringGroups: ScoringCategory[]
  ): Promise<OutlineItem[]> {
    const messages = [
      {
        role: 'user',
        content: TOP_LEVEL_OUTLINE_PROMPT
          .replace('{scoring_criteria}', JSON.stringify(scoringGroups, null, 2))
          .replace('{project_overview}', analysis.project_overview)
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.2 });
    return JSON.parse(response).outline;
  }
  
  private async generateChildrenRecursive(
    items: OutlineItem[],
    analysis: TenderAnalysis,
    maxDepth: number,
    currentLevel: number = 1
  ): Promise<OutlineItem[]> {
    if (currentLevel >= maxDepth) {
      return items;
    }
    
    // 并发为每个节点生成子目录
    const results = await Promise.all(
      items.map(async (item) => {
        const scoringItems = this.getScoringItemsForCategory(
          item.requirement_ids,
          analysis.scoring_criteria
        );
        
        if (scoringItems.length === 0 && currentLevel >= 2) {
          return item; // 没有对应的评分细项，不需要继续展开
        }
        
        const children = await this.generateChildren(
          item,
          scoringItems,
          analysis
        );
        
        if (children.length > 0) {
          item.children = await this.generateChildrenRecursive(
            children,
            analysis,
            maxDepth,
            currentLevel + 1
          );
        }
        
        return item;
      })
    );
    
    return results;
  }
  
  private async generateChildren(
    parent: OutlineItem,
    scoringItems: ScoringItem[],
    analysis: TenderAnalysis
  ): Promise<OutlineItem[]> {
    const messages = [
      {
        role: 'user',
        content: CHILDREN_OUTLINE_PROMPT
          .replace('{parent_id}', parent.id)
          .replace('{parent_title}', parent.title)
          .replace('{parent_description}', parent.description)
          .replace('{scoring_items}', JSON.stringify(scoringItems, null, 2))
          .replace('{project_overview}', analysis.project_overview)
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.3 });
    return JSON.parse(response).children || [];
  }
  
  private async reviewOutline(
    outline: Outline,
    analysis: TenderAnalysis
  ): Promise<OutlineReviewResult> {
    const messages = [
      {
        role: 'user',
        content: OUTLINE_REVIEW_PROMPT
          .replace('{outline_json}', JSON.stringify(outline.items, null, 2))
          .replace('{scoring_criteria}', JSON.stringify(analysis.scoring_criteria, null, 2))
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
}
```

---

## 6. 目录编号规则

### 6.1 编号格式

```
一级目录: 1, 2, 3, ...
二级目录: 1.1, 1.2, 1.3, ...
三级目录: 1.1.1, 1.1.2, 1.1.3, ...
四级目录: 1.1.1.1, 1.1.1.2, ...
```

### 6.2 编号生成算法

```typescript
function generateChapterId(parentId: string | null, index: number): string {
  if (!parentId) {
    return String(index + 1);
  }
  return `${parentId}.${index + 1}`;
}
```

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
