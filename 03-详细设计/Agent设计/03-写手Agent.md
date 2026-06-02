# Writer Agent (写手) 详细设计

## 1. Agent概述

### 1.1 职责定义

写手Agent是标书内容生成的核心Agent，负责根据大纲、上下文和编排决策，生成高质量的标书正文内容。

### 1.2 核心能力

```
┌─────────────────────────────────────────────────────────────┐
│                    Writer Agent 核心能力                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ✍️ 内容生成                                                │
│      - 根据章节描述生成正文                                 │
│      - 支持多种内容格式（段落、列表、表格）                 │
│      - 遵循风格指南统一要求                                 │
│                                                             │
│   📚 上下文利用                                              │
│      - 使用术语表保持一致性                                 │
│      - 参考章节摘要避免重复                                 │
│      - 引用其他章节内容                                     │
│                                                             │
│   🔄 流式上下文更新                                          │
│      - 完成后生成摘要                                       │
│      - 提取新术语                                           │
│      - 广播给其他写手                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 接口设计

### 2.1 输入接口

```typescript
interface WriterTaskInput {
  chapter: {
    id: string;
    title: string;
    description: string;
    level: number;
    parent_id?: string;
  };
  context: SharedContextData;
  content_plan?: ContentPlan;
  options?: {
    temperature?: number;
    max_words?: number;
    regenerate?: boolean;
    regenerate_requirement?: string;
  };
}
```

### 2.2 输出接口

```typescript
interface ChapterContent {
  id: string;
  chapter_id: string;
  title: string;
  content: string;
  word_count: number;
  metadata: {
    generation_time: number;
    knowledge_used: string[];
    terminology_used: string[];
    references_made: Reference[];
  };
}
```

---

## 3. 核心流程

### 3.1 处理流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    Writer Agent 处理流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入: Chapter, SharedContext, ContentPlan                  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ① 构建上下文信息                                      │  │
│   │    - 提取项目概述                                      │  │
│   │    - 获取相关章节摘要                                  │  │
│   │    - 加载术语表                                        │  │
│   │    - 读取风格指南                                      │  │
│   │    - 加载知识库素材                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ② 构建Prompt                                          │  │
│   │    - 系统提示词                                        │  │
│   │    - 章节信息                                          │  │
│   │    - 上下文信息                                        │  │
│   │    - 编排决策                                          │  │
│   │    - 生成要求                                          │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ③ 调用AI生成内容                                      │  │
│   │    - 发送消息到AI服务                                  │  │
│   │    - 接收生成内容                                      │  │
│   │    - 流式处理（可选）                                  │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ④ 后处理                                              │  │
│   │    - 去除重复标题                                      │  │
│   │    - 格式规范化                                        │  │
│   │    - 去除Markdown标题语法                              │  │
│   │    - 统计字数                                          │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑤ 更新共享上下文                                      │  │
│   │    - 生成章节摘要                                      │  │
│   │    - 提取新术语                                        │  │
│   │    - 提取交叉引用                                      │  │
│   │    - 广播完成事件                                      │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   输出: ChapterContent                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Prompt设计

### 4.1 系统提示词

```typescript
const WRITER_SYSTEM_PROMPT = `你是一个专业的标书技术写手，拥有丰富的投标文件编写经验。

## 写作要求
1. 内容要专业、准确，与章节标题和描述保持一致
2. 这是技术方案，不是宣传报告，注意朴实无华，不要假大空
3. 语言要正式、规范，符合标书写作要求
4. 不要使用奇怪的连接词，不要让人觉得内容像是 AI 生成的
5. 内容要详细具体，避免空泛的描述
6. 注意避免与同级章节内容重复，保持内容的独特性和互补性

## 格式规范
1. 可以使用 Markdown 段落、列表和表格
2. 表格必须服务于内容表达，不要为了形式硬插
3. 严禁使用 Markdown 标题语法（#、##、###、####、#####、######）
4. 不要生成与当前章节同级或下级的伪目录标题
5. 如需在正文中分层表达，只能使用普通段落、列表、表格或加粗引导语
6. 严禁输出 Mermaid、PlantUML 等图表代码块

## 上下文使用规则
1. 使用术语表中的标准术语，保持术语一致性
2. 可以引用其他章节，使用标准格式：见第X章"章节名称"
3. 参考章节摘要，避免内容重复
4. 遵循风格指南的统一要求

## 知识库使用规则
1. 以下内容只作为可吸收的技术素材
2. 请改写为当前项目语境下的投标技术方案正文
3. 不要照抄，不要提到"知识库""历史文档""参考资料"或素材来源`;
```

### 4.2 内容生成提示词

```typescript
const CONTENT_GENERATION_PROMPT = `请为以下标书章节生成具体内容：

## 章节信息
- 章节ID: {chapter_id}
- 章节标题: {chapter_title}
- 章节描述: {chapter_description}

## 项目概述
{project_overview}

## 标准术语表
{terminology}

## 相关章节摘要
{related_summaries}

## 风格指南
{style_guide}

## 知识库素材（仅供参考，需要改写）
{knowledge_contents}

## 编排决策
{content_plan}

## 生成要求
1. 直接返回章节内容，不生成标题
2. 内容要详细具体，字数不少于{min_words}字
3. 注意与相关章节的内容互补，避免重复
4. 使用术语表中的标准术语

请生成内容：`;
```

---

## 5. 实现代码

```typescript
import { BaseAgent, AgentType, AgentStatus } from '../base/BaseAgent';
import { AIService } from '../../services/AIService';
import { SharedContext } from '../../context/SharedContext';
import { SummaryGenerator } from '../../services/SummaryGenerator';
import { TerminologyExtractor } from '../../services/TerminologyExtractor';
import { Logger } from '../../utils/Logger';

export class WriterAgent extends BaseAgent {
  private summaryGenerator: SummaryGenerator;
  private terminologyExtractor: TerminologyExtractor;
  
  constructor(
    aiService: AIService,
    sharedContext: SharedContext,
    logger: Logger
  ) {
    super({
      id: 'writer-agent',
      type: AgentType.WRITER,
      aiService,
      sharedContext,
      logger
    });
    
    this.summaryGenerator = new SummaryGenerator(aiService);
    this.terminologyExtractor = new TerminologyExtractor(aiService);
  }
  
  protected getSystemPrompt(): string {
    return WRITER_SYSTEM_PROMPT;
  }
  
  async execute(input: WriterTaskInput): Promise<ChapterContent> {
    this.updateStatus(AgentStatus.BUSY);
    const startTime = Date.now();
    
    try {
      const { chapter, context, content_plan, options } = input;
      
      this.logger.info(`开始生成章节: ${chapter.id} ${chapter.title}`);
      
      // 1. 构建上下文信息
      const contextInfo = this.buildContextInfo(chapter, context, content_plan);
      
      // 2. 构建消息
      const messages = this.buildMessages(chapter, contextInfo, options);
      
      // 3. 调用AI生成内容
      const rawContent = await this.callAI(messages, {
        temperature: options?.temperature || 0.7,
        maxTokens: 4096
      });
      
      // 4. 后处理
      const processedContent = this.postProcess(rawContent, chapter);
      
      // 5. 更新共享上下文
      await this.updateSharedContext(chapter, processedContent);
      
      // 6. 构建输出
      const result: ChapterContent = {
        id: this.generateId(),
        chapter_id: chapter.id,
        title: chapter.title,
        content: processedContent,
        word_count: this.countWords(processedContent),
        metadata: {
          generation_time: Date.now() - startTime,
          knowledge_used: content_plan?.knowledge_ids || [],
          terminology_used: this.extractUsedTerminology(processedContent, context.terminology),
          references_made: this.extractReferences(processedContent)
        }
      };
      
      this.updateStatus(AgentStatus.IDLE);
      this.logger.info(`章节生成完成: ${chapter.id}, 字数: ${result.word_count}`);
      
      return result;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      this.logger.error(`章节生成失败: ${error.message}`);
      throw error;
    }
  }
  
  private buildContextInfo(
    chapter: WriterTaskInput['chapter'],
    context: SharedContextData,
    contentPlan?: ContentPlan
  ): string {
    const parts: string[] = [];
    
    // 项目概述
    if (context.projectOverview) {
      parts.push(`## 项目概述\n${context.projectOverview}`);
    }
    
    // 术语表
    if (context.terminology.size > 0) {
      const terms = Array.from(context.terminology.entries())
        .map(([term, def]) => `- **${term}**: ${def}`)
        .join('\n');
      parts.push(`## 标准术语表\n${terms}`);
    }
    
    // 相关章节摘要
    const relatedSummaries = this.getRelatedSummaries(chapter, context);
    if (relatedSummaries.length > 0) {
      parts.push(`## 相关章节摘要\n${relatedSummaries.join('\n\n')}`);
    }
    
    // 风格指南
    if (context.styleGuide) {
      parts.push(`## 风格指南\n${JSON.stringify(context.styleGuide, null, 2)}`);
    }
    
    // 知识库素材
    if (contentPlan?.knowledge_ids?.length > 0) {
      const knowledgeContents = this.loadKnowledgeContents(contentPlan.knowledge_ids);
      if (knowledgeContents) {
        parts.push(`## 知识库素材（仅供参考）\n${knowledgeContents}`);
      }
    }
    
    return parts.join('\n\n');
  }
  
  private buildMessages(
    chapter: WriterTaskInput['chapter'],
    contextInfo: string,
    options?: WriterTaskInput['options']
  ): Message[] {
    const content = CONTENT_GENERATION_PROMPT
      .replace('{chapter_id}', chapter.id)
      .replace('{chapter_title}', chapter.title)
      .replace('{chapter_description}', chapter.description)
      .replace('{project_overview}', contextInfo)
      .replace('{min_words}', String(options?.max_words || 500));
    
    const messages: Message[] = [
      { role: 'user', content }
    ];
    
    // 如果是重新生成，添加额外要求
    if (options?.regenerate && options?.regenerate_requirement) {
      messages.push({
        role: 'user',
        content: `用户对本次重新生成的额外要求：\n${options.regenerate_requirement}`
      });
    }
    
    return messages;
  }
  
  private postProcess(content: string, chapter: WriterTaskInput['chapter']): string {
    let processed = content;
    
    // 1. 去除重复的章节标题
    processed = this.stripRepeatedChapterTitle(processed, chapter.title);
    
    // 2. 去除Markdown标题语法
    processed = this.stripMarkdownHeadings(processed);
    
    // 3. 规范化换行
    processed = this.normalizeLineBreaks(processed);
    
    // 4. 去除首尾空白
    processed = processed.trim();
    
    return processed;
  }
  
  private async updateSharedContext(
    chapter: WriterTaskInput['chapter'],
    content: string
  ): Promise<void> {
    try {
      // 1. 生成摘要
      const summary = await this.summaryGenerator.generate(content, 200);
      
      // 2. 提取术语
      const terms = await this.terminologyExtractor.extract(content);
      
      // 3. 提取引用
      const references = this.extractReferences(content);
      
      // 4. 更新共享上下文
      await this.sharedContext.update({
        chapterId: chapter.id,
        summary: {
          chapter_id: chapter.id,
          summary: summary,
          key_points: this.extractKeyPoints(content),
          generated_at: new Date()
        },
        terms: terms,
        references: references
      });
      
      // 5. 广播事件
      this.eventBus.emit('CHAPTER_COMPLETED', {
        chapterId: chapter.id,
        summary,
        terms,
        references
      });
      
      this.logger.debug(`共享上下文已更新: ${chapter.id}`);
    } catch (error) {
      this.logger.warn(`更新共享上下文失败: ${error.message}`);
    }
  }
  
  private getRelatedSummaries(
    chapter: WriterTaskInput['chapter'],
    context: SharedContextData
  ): string[] {
    const summaries: string[] = [];
    
    // 获取父章节摘要
    if (chapter.parent_id) {
      const parentSummary = context.chapterSummaries.get(chapter.parent_id);
      if (parentSummary) {
        summaries.push(`**父章节摘要：**\n${parentSummary.summary}`);
      }
    }
    
    // 获取同级章节摘要
    const siblingIds = this.getSiblingChapterIds(chapter.id);
    for (const siblingId of siblingIds) {
      const siblingSummary = context.chapterSummaries.get(siblingId);
      if (siblingSummary) {
        summaries.push(`**同级章节 ${siblingId} 摘要：**\n${siblingSummary.summary}`);
      }
    }
    
    return summaries;
  }
  
  private stripRepeatedChapterTitle(content: string, title: string): string {
    const lines = content.split('\n');
    const firstLine = lines[0]?.trim();
    
    // 如果第一行与章节标题相同或相似，去除它
    if (firstLine && (
      firstLine === title ||
      firstLine === `# ${title}` ||
      firstLine === `## ${title}` ||
      firstLine.replace(/^[一二三四五六七八九十]+[、.．]\s*/, '') === title
    )) {
      return lines.slice(1).join('\n').trim();
    }
    
    return content;
  }
  
  private stripMarkdownHeadings(content: string): string {
    return content.replace(/^#{1,6}\s+(.+)$/gm, '**$1**');
  }
}
```

---

## 6. 分组串行策略

### 6.1 分组算法

```typescript
interface ChapterGroup {
  id: string;
  parentId: string;
  chapters: Chapter[];
  mode: 'serial';
}

function groupChaptersByParent(outline: Outline): ChapterGroup[] {
  const groups: ChapterGroup[] = [];
  const topLevelItems = outline.items;
  
  for (const item of topLevelItems) {
    const leafChapters = collectLeafChapters(item);
    
    if (leafChapters.length > 0) {
      groups.push({
        id: `group-${item.id}`,
        parentId: item.id,
        chapters: leafChapters,
        mode: 'serial'
      });
    }
  }
  
  return groups;
}

function collectLeafChapters(item: OutlineItem): Chapter[] {
  if (!item.children || item.children.length === 0) {
    return [convertToChapter(item)];
  }
  
  const chapters: Chapter[] = [];
  for (const child of item.children) {
    chapters.push(...collectLeafChapters(child));
  }
  return chapters;
}
```

### 6.2 串行生成

```typescript
async function generateGroupSerial(
  group: ChapterGroup,
  writerAgent: WriterAgent,
  sharedContext: SharedContext
): Promise<ChapterContent[]> {
  const results: ChapterContent[] = [];
  
  for (const chapter of group.chapters) {
    // 获取当前上下文快照
    const context = await sharedContext.getSnapshot();
    
    // 生成内容
    const content = await writerAgent.execute({
      chapter,
      context
    });
    
    results.push(content);
    
    // 等待一小段时间，确保上下文更新完成
    await delay(100);
  }
  
  return results;
}
```

---

## 7. 错误处理

### 7.1 错误类型

```typescript
enum WriterErrorType {
  CONTEXT_BUILD_FAILED = 'CONTEXT_BUILD_FAILED',
  AI_GENERATION_FAILED = 'AI_GENERATION_FAILED',
  POST_PROCESS_FAILED = 'POST_PROCESS_FAILED',
  CONTEXT_UPDATE_FAILED = 'CONTEXT_UPDATE_FAILED'
}
```

### 7.2 重试策略

- AI生成失败：重试3次，指数退避
- 上下文更新失败：记录日志，不影响主流程
- 后处理失败：返回原始内容

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
