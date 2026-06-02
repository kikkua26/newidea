# Polisher Agent (润色师) 详细设计

## 1. Agent概述

### 1.1 职责定义

润色Agent负责对审核通过的内容进行最终润色，统一风格、优化语言、消除AI痕迹，确保输出高质量的标书正文。

### 1.2 润色维度

```
┌─────────────────────────────────────────────────────────────┐
│                    Polisher Agent 润色维度                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   📝 语言优化                                                │
│      - 修正语法错误                                         │
│      - 优化句子结构                                         │
│      - 增强表达准确性                                       │
│                                                             │
│   🎨 风格统一                                                │
│      - 统一术语使用                                         │
│      - 统一语气风格                                         │
│      - 统一格式规范                                         │
│                                                             │
│   🤖 消除AI痕迹                                              │
│      - 去除模板化表达                                       │
│      - 增加自然过渡                                         │
│      - 使语言更人性化                                       │
│                                                             │
│   🔗 增强连贯性                                              │
│      - 优化段落过渡                                         │
│      - 增强逻辑连接                                         │
│      - 统一全文节奏                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 接口设计

### 2.1 输入接口

```typescript
interface PolisherTaskInput {
  chapters: ChapterContent[];
  review_report: ReviewReport;
  context: SharedContextData;
  options?: {
    polish_level: 'light' | 'medium' | 'heavy';  // 润色程度
    preserve_structure: boolean;                   // 保持结构
    focus_areas?: string[];                        // 重点关注领域
  };
}
```

### 2.2 输出接口

```typescript
interface PolishedContent {
  id: string;
  chapter_id: string;
  title: string;
  original_content: string;
  polished_content: string;
  changes: ContentChange[];
  metadata: {
    polish_time: number;
    change_count: number;
    word_count_before: number;
    word_count_after: number;
  };
}

interface ContentChange {
  type: 'add' | 'modify' | 'delete';
  location: string;
  original: string;
  modified: string;
  reason: string;
}
```

---

## 3. 核心流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Polisher Agent 处理流程                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入: Chapters[], ReviewReport, SharedContext              │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ① 分析审核报告                                        │  │
│   │    - 提取需要修正的问题                                │  │
│   │    - 确定润色重点                                      │  │
│   │    - 制定润色策略                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ② 修正审核问题                                        │  │
│   │    - 根据审核报告修正具体问题                          │  │
│   │    - 补充缺失内容                                      │  │
│   │    - 修正逻辑错误                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ③ 统一术语                                            │  │
│   │    - 替换非标准术语                                    │  │
│   │    - 统一术语表达                                      │  │
│   │    - 确保术语表一致                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ④ 优化语言                                            │  │
│   │    - 修正语法错误                                      │  │
│   │    - 优化句子结构                                      │  │
│   │    - 增强表达准确性                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑤ 消除AI痕迹                                          │  │
│   │    - 去除模板化表达                                    │  │
│   │    - 增加自然过渡                                      │  │
│   │    - 使语言更人性化                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑥ 增强连贯性                                          │  │
│   │    - 优化段落过渡                                      │  │
│   │    - 增强逻辑连接                                      │  │
│   │    - 统一全文节奏                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   输出: PolishedContent[]                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Prompt设计

### 4.1 系统提示词

```typescript
const POLISHER_SYSTEM_PROMPT = `你是一个专业的标书润色专家，拥有丰富的文书润色经验。

## 润色原则
1. 保持原意不变，不添加新内容
2. 使语言更自然流畅，符合正式文书规范
3. 统一术语和风格，确保全文一致
4. 消除AI生成痕迹，使内容更人性化
5. 增强可读性和专业性

## 润色要求
- 修正语法错误
- 优化句子结构
- 统一术语使用
- 调整段落节奏
- 增强逻辑连贯性
- 去除模板化表达

## 输出要求
- 返回润色后的完整内容
- 记录所做的修改
- 保持原有格式和结构`;
```

### 4.2 修正问题提示词

```typescript
const FIX_ISSUES_PROMPT = `请根据以下审核报告修正标书内容：

## 审核问题
{issues}

## 原始内容
{content}

## 修正要求
1. 根据审核问题逐一修正
2. 保持原有结构和格式
3. 不要添加新的内容
4. 只修正指出的问题

请返回修正后的内容：`;
```

### 4.3 消除AI痕迹提示词

```typescript
const REMOVE_AI痕迹_PROMPT = `请润色以下标书内容，消除AI生成痕迹：

## 原始内容
{content}

## 润色要求
1. 去除模板化、套路化的表达
2. 增加自然的过渡和连接
3. 使用更人性化的语言
4. 保持专业性和准确性
5. 不要改变原意

## 常见AI痕迹
- "首先...其次...最后..." 的机械排列
- "综上所述"、"总而言之" 等套话
- 过于工整的句式结构
- 缺乏自然过渡的段落

请返回润色后的内容：`;
```

---

## 5. 实现代码

```typescript
import { BaseAgent, AgentType, AgentStatus } from '../base/BaseAgent';
import { AIService } from '../../services/AIService';
import { TerminologyReplacer } from '../../services/TerminologyReplacer';
import { Logger } from '../../utils/Logger';

export class PolisherAgent extends BaseAgent {
  private terminologyReplacer: TerminologyReplacer;
  
  constructor(
    aiService: AIService,
    logger: Logger
  ) {
    super({
      id: 'polisher-agent',
      type: AgentType.POLISHER,
      aiService,
      logger
    });
    
    this.terminologyReplacer = new TerminologyReplacer();
  }
  
  protected getSystemPrompt(): string {
    return POLISHER_SYSTEM_PROMPT;
  }
  
  async execute(input: PolisherTaskInput): Promise<PolishedContent[]> {
    this.updateStatus(AgentStatus.BUSY);
    const startTime = Date.now();
    
    try {
      this.logger.info('开始润色标书内容');
      
      const { chapters, review_report, context, options } = input;
      const results: PolishedContent[] = [];
      
      for (const chapter of chapters) {
        this.logger.info(`润色章节: ${chapter.chapter_id}`);
        
        // 1. 修正审核问题
        let polished = await this.fixIssues(chapter.content, review_report, chapter.chapter_id);
        
        // 2. 统一术语
        polished = await this.unifyTerminology(polished, context.terminology);
        
        // 3. 消除AI痕迹
        polished = await this.removeAITrace(polished);
        
        // 4. 增强连贯性
        polished = await this.enhanceCoherence(polished);
        
        // 5. 记录变更
        const changes = this.detectChanges(chapter.content, polished);
        
        results.push({
          id: this.generateId(),
          chapter_id: chapter.chapter_id,
          title: chapter.title,
          original_content: chapter.content,
          polished_content: polished,
          changes,
          metadata: {
            polish_time: Date.now() - startTime,
            change_count: changes.length,
            word_count_before: this.countWords(chapter.content),
            word_count_after: this.countWords(polished)
          }
        });
      }
      
      this.updateStatus(AgentStatus.IDLE);
      this.logger.info(`润色完成，共处理 ${results.length} 个章节`);
      
      return results;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      this.logger.error(`润色失败: ${error.message}`);
      throw error;
    }
  }
  
  private async fixIssues(
    content: string,
    reviewReport: ReviewReport,
    chapterId: string
  ): Promise<string> {
    const chapterIssues = reviewReport.issues.filter(i => i.chapter_id === chapterId);
    
    if (chapterIssues.length === 0) {
      return content;
    }
    
    const messages = [
      {
        role: 'user',
        content: FIX_ISSUES_PROMPT
          .replace('{issues}', JSON.stringify(chapterIssues, null, 2))
          .replace('{content}', content)
      }
    ];
    
    return await this.callAI(messages, { temperature: 0.3 });
  }
  
  private async unifyTerminology(
    content: string,
    terminology: Map<string, any>
  ): Promise<string> {
    return await this.terminologyReplacer.replace(content, terminology);
  }
  
  private async removeAITrace(content: string): Promise<string> {
    const messages = [
      {
        role: 'user',
        content: REMOVE_AI痕迹_PROMPT.replace('{content}', content)
      }
    ];
    
    return await this.callAI(messages, { temperature: 0.5 });
  }
  
  private async enhanceCoherence(content: string): Promise<string> {
    // 如果内容较短，不需要增强连贯性
    if (content.length < 500) {
      return content;
    }
    
    const messages = [
      {
        role: 'user',
        content: `请优化以下内容的段落过渡和逻辑连接：

${content}

要求：
1. 优化段落间的过渡
2. 增强逻辑连接词
3. 保持原有内容不变
4. 使文章更流畅自然

请返回优化后的内容：`
      }
    ];
    
    return await this.callAI(messages, { temperature: 0.4 });
  }
  
  private detectChanges(original: string, polished: string): ContentChange[] {
    const changes: ContentChange[] = [];
    
    // 简单的变更检测：按段落对比
    const originalParagraphs = original.split('\n\n');
    const polishedParagraphs = polished.split('\n\n');
    
    for (let i = 0; i < Math.max(originalParagraphs.length, polishedParagraphs.length); i++) {
      const orig = originalParagraphs[i] || '';
      const pol = polishedParagraphs[i] || '';
      
      if (orig !== pol) {
        if (!orig && pol) {
          changes.push({
            type: 'add',
            location: `段落 ${i + 1}`,
            original: '',
            modified: pol,
            reason: '补充内容'
          });
        } else if (orig && !pol) {
          changes.push({
            type: 'delete',
            location: `段落 ${i + 1}`,
            original: orig,
            modified: '',
            reason: '删除内容'
          });
        } else {
          changes.push({
            type: 'modify',
            location: `段落 ${i + 1}`,
            original: orig,
            modified: pol,
            reason: '润色优化'
          });
        }
      }
    }
    
    return changes;
  }
}
```

---

## 6. 润色策略

### 6.1 轻度润色

- 修正明显语法错误
- 统一术语使用
- 保持原文风格

### 6.2 中度润色

- 修正语法错误
- 统一术语使用
- 优化句子结构
- 消除部分AI痕迹

### 6.3 重度润色

- 修正语法错误
- 统一术语使用
- 优化句子结构
- 消除AI痕迹
- 增强连贯性
- 重新组织段落

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
