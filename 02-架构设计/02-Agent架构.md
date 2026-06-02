# Agent架构

## 1. Agent架构概览

### 1.1 架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            多Agent协作架构                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      Orchestrator (编排引擎)                         │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │   │
│  │  │  FlowController│  │ TaskScheduler │  │  StateManager │           │   │
│  │  │   (流程控制)   │  │  (任务调度)   │  │  (状态管理)   │           │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ↓                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                       Agent Pool (Agent池)                          │   │
│  │                                                                     │   │
│  │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │   │
│  │   │ Analyst  │    │ Outliner │    │  Writer  │    │ Reviewer │    │   │
│  │   │ (分析师) │    │ (大纲师) │    │ (写手)   │    │ (审核员) │    │   │
│  │   └────┬─────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    │   │
│  │        │               │               │               │           │   │
│  │        └───────────────┴───────────────┴───────────────┘           │   │
│  │                                    │                                │   │
│  │                              ┌─────┴─────┐                         │   │
│  │                              │ Polisher  │                         │   │
│  │                              │ (润色师)   │                         │   │
│  │                              └───────────┘                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                      │
│                                      ↓                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    SharedContext (共享上下文池)                      │   │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐      │   │
│  │  │ Terminology│ │ Summaries  │ │ StyleGuide │ │ References │      │   │
│  │  │  (术语表)  │ │ (章节摘要) │ │ (风格指南) │ │ (引用表)   │      │   │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Agent类型定义

### 2.1 Agent基类

```typescript
// Agent类型枚举
enum AgentType {
  ANALYST = 'analyst',       // 分析师
  OUTLINER = 'outliner',    // 大纲师
  WRITER = 'writer',         // 写手
  REVIEWER = 'reviewer',     // 审核员
  POLISHER = 'polisher',     // 润色师
  ILLUSTRATOR = 'illustrator' // 配图师
}

// Agent状态枚举
enum AgentStatus {
  IDLE = 'idle',
  BUSY = 'busy',
  ERROR = 'error',
  PAUSED = 'paused'
}

// Agent基类
abstract class BaseAgent {
  protected id: string;
  protected type: AgentType;
  protected status: AgentStatus;
  protected aiService: AIService;
  protected sharedContext: SharedContext;
  protected logger: Logger;
  
  constructor(config: AgentConfig) {
    this.id = config.id;
    this.type = config.type;
    this.status = AgentStatus.IDLE;
    this.aiService = config.aiService;
    this.sharedContext = config.sharedContext;
    this.logger = config.logger;
  }
  
  // 执行任务（子类实现）
  abstract execute<T>(task: AgentTask): Promise<T>;
  
  // 获取系统提示词（子类实现）
  protected abstract getSystemPrompt(): string;
  
  // 调用AI服务
  protected async callAI(messages: Message[], options?: AIOptions): Promise<string> {
    const systemPrompt = this.getSystemPrompt();
    const fullMessages = [
      { role: 'system', content: systemPrompt },
      ...messages
    ];
    
    return await this.aiService.chat(fullMessages, options);
  }
  
  // 获取共享上下文
  protected getContext(): SharedContextData {
    return this.sharedContext.getSnapshot();
  }
  
  // 更新状态
  protected updateStatus(status: AgentStatus): void {
    this.status = status;
    this.logger.info(`Agent ${this.id} status changed to ${status}`);
  }
}
```

---

## 3. 各Agent详细设计

### 3.1 Analyst Agent (分析师)

**职责**: 解析招标文件，提取结构化信息

```typescript
class AnalystAgent extends BaseAgent {
  protected getSystemPrompt(): string {
    return `你是一个专业的招标文件分析师。

职责：
1. 深入分析招标文件，提取关键信息
2. 识别技术要求和评分标准
3. 提取资质要求和商务条件
4. 总结项目概述和关键要点

输出要求：
- 使用结构化JSON格式
- 确保信息完整准确
- 标注信息来源位置`;
  }
  
  async execute<TenderAnalysis>(task: AgentTask): Promise<TenderAnalysis> {
    this.updateStatus(AgentStatus.BUSY);
    
    try {
      const { tenderDoc } = task.input;
      
      // 1. 文档预处理
      const text = await this.preprocessDocument(tenderDoc);
      
      // 2. 提取结构化信息
      const analysis = await this.extractInformation(text);
      
      // 3. 验证和补充
      const validated = await this.validateAnalysis(analysis);
      
      this.updateStatus(AgentStatus.IDLE);
      return validated;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      throw error;
    }
  }
  
  private async extractInformation(text: string): Promise<TenderAnalysis> {
    const messages = [
      {
        role: 'user',
        content: `请分析以下招标文件内容，提取结构化信息：

${text}

请返回以下JSON格式：
{
  "project_name": "项目名称",
  "project_overview": "项目概述",
  "technical_requirements": ["技术要求1", "技术要求2"],
  "scoring_criteria": [
    {
      "category": "评分大类",
      "items": [
        {"item": "评分细项", "score": 分值, "description": "评分标准"}
      ]
    }
  ],
  "qualification_requirements": ["资质要求1", "资质要求2"],
  "key_points": ["关键要点1", "关键要点2"]
}`
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
}
```

---

### 3.2 Outliner Agent (大纲师)

**职责**: 根据分析结果生成标书大纲

```typescript
class OutlinerAgent extends BaseAgent {
  protected getSystemPrompt(): string {
    return `你是一个专业的标书大纲设计专家。

职责：
1. 根据招标文件的技术要求和评分标准，设计标书目录结构
2. 确保一级目录与评分大类对齐
3. 确保二级、三级目录覆盖所有评分细项
4. 保持目录层级清晰、逻辑合理

设计原则：
- 一级目录名称应与评分大类一致
- 每个评分点都应有对应的目录章节
- 目录层级不超过4级
- 章节编号规范统一`;
  }
  
  async execute<Outline>(task: AgentTask): Promise<Outline> {
    this.updateStatus(AgentStatus.BUSY);
    
    try {
      const { analysis } = task.input;
      
      // 1. 生成一级目录
      const topLevel = await this.generateTopLevel(analysis);
      
      // 2. 并发生成各一级目录的子目录
      const children = await Promise.all(
        topLevel.map(item => this.generateChildren(item, analysis))
      );
      
      // 3. 合并成完整大纲
      const outline = this.mergeOutline(topLevel, children);
      
      // 4. 审核大纲
      const reviewed = await this.reviewOutline(outline, analysis);
      
      this.updateStatus(AgentStatus.IDLE);
      return reviewed;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      throw error;
    }
  }
  
  private async generateTopLevel(analysis: TenderAnalysis): Promise<OutlineItem[]> {
    const messages = [
      {
        role: 'user',
        content: `根据以下技术要求和评分标准，生成标书的一级目录：

技术要求：
${analysis.technical_requirements.join('\n')}

评分标准：
${JSON.stringify(analysis.scoring_criteria, null, 2)}

请返回JSON格式的一级目录列表：
{
  "outline": [
    {
      "id": "1",
      "title": "目录标题",
      "description": "目录描述",
      "requirement_ids": ["R1", "R2"]
    }
  ]
}`
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.2 });
    return JSON.parse(response).outline;
  }
}
```

---

### 3.3 Writer Agent (写手)

**职责**: 根据大纲和上下文生成章节内容

```typescript
class WriterAgent extends BaseAgent {
  protected getSystemPrompt(): string {
    return `你是一个专业的标书技术写手。

写作要求：
1. 内容要专业、准确，与章节标题和描述保持一致
2. 语言要正式、规范，符合标书写作要求
3. 内容要详细具体，避免空泛的描述
4. 注意避免与同级章节内容重复
5. 可以使用Markdown段落、列表和表格
6. 严禁使用Markdown标题语法（#、##、###等）
7. 如需分层表达，只能使用加粗引导语

上下文使用规则：
- 使用术语表中的标准术语
- 可以引用其他章节，使用标准格式：见第X章"章节名称"
- 参考章节摘要，避免内容重复
- 遵循风格指南的统一要求`;
  }
  
  async execute<ChapterContent>(task: AgentTask): Promise<ChapterContent> {
    this.updateStatus(AgentStatus.BUSY);
    
    try {
      const { chapter, context } = task.input;
      
      // 1. 构建上下文信息
      const contextInfo = this.buildContextInfo(chapter, context);
      
      // 2. 生成内容
      const content = await this.generateContent(chapter, contextInfo);
      
      // 3. 后处理
      const processed = this.postProcess(content, chapter);
      
      this.updateStatus(AgentStatus.IDLE);
      return processed;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      throw error;
    }
  }
  
  private buildContextInfo(chapter: Chapter, context: SharedContextData): string {
    const parts = [];
    
    // 项目概述
    if (context.projectOverview) {
      parts.push(`项目概述：\n${context.projectOverview}`);
    }
    
    // 术语表
    if (context.terminology.size > 0) {
      const terms = Array.from(context.terminology.entries())
        .map(([term, def]) => `- ${term}: ${def}`)
        .join('\n');
      parts.push(`标准术语表：\n${terms}`);
    }
    
    // 相关章节摘要
    const relatedSummaries = this.getRelatedSummaries(chapter, context);
    if (relatedSummaries.length > 0) {
      parts.push(`相关章节摘要：\n${relatedSummaries.join('\n')}`);
    }
    
    // 风格指南
    if (context.styleGuide) {
      parts.push(`风格指南：\n${JSON.stringify(context.styleGuide, null, 2)}`);
    }
    
    return parts.join('\n\n');
  }
  
  private async generateContent(chapter: Chapter, contextInfo: string): Promise<string> {
    const messages = [
      {
        role: 'user',
        content: `请为以下章节生成内容：

章节信息：
- 章节ID: ${chapter.id}
- 章节标题: ${chapter.title}
- 章节描述: ${chapter.description}

${contextInfo}

请直接返回章节内容，不要包含章节标题。`
      }
    ];
    
    return await this.callAI(messages, { temperature: 0.7 });
  }
}
```

---

### 3.4 Reviewer Agent (审核员)

**职责**: 审核生成内容的质量

```typescript
class ReviewerAgent extends BaseAgent {
  protected getSystemPrompt(): string {
    return `你是一个严格的标书质量审核员。

审核维度：
1. 内容完整性 - 是否覆盖所有评分点，内容是否充实
2. 逻辑一致性 - 章节间是否矛盾，论证是否完整
3. 风格统一性 - 术语使用是否一致，语气是否统一
4. 格式规范性 - 是否符合标书格式要求
5. 语言质量 - 表达是否准确，是否有AI痕迹

审核要求：
- 对每个维度给出评分（0-100）
- 列出具体问题和修改建议
- 给出总体评价和通过/不通过结论`;
  }
  
  async execute<ReviewReport>(task: AgentTask): Promise<ReviewReport> {
    this.updateStatus(AgentStatus.BUSY);
    
    try {
      const { content, outline, context } = task.input;
      
      // 1. 内容完整性审核
      const completeness = await this.reviewCompleteness(content, outline);
      
      // 2. 逻辑一致性审核
      const consistency = await this.reviewConsistency(content);
      
      // 3. 风格统一性审核
      const style = await this.reviewStyle(content, context);
      
      // 4. 格式规范性审核
      const format = await this.reviewFormat(content);
      
      // 5. 语言质量审核
      const language = await this.reviewLanguage(content);
      
      // 6. 生成审核报告
      const report = this.generateReport({
        completeness,
        consistency,
        style,
        format,
        language
      });
      
      this.updateStatus(AgentStatus.IDLE);
      return report;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      throw error;
    }
  }
  
  private async reviewCompleteness(content: string, outline: Outline): Promise<ReviewResult> {
    const messages = [
      {
        role: 'user',
        content: `请审核以下内容的完整性：

大纲要求：
${JSON.stringify(outline, null, 2)}

生成内容：
${content}

请检查：
1. 是否所有大纲章节都有对应内容
2. 内容是否覆盖了所有评分点
3. 内容是否充实，有无遗漏

返回JSON格式：
{
  "score": 85,
  "issues": [
    {"type": "missing", "description": "缺少xxx内容", "location": "章节ID"}
  ],
  "suggestions": ["建议1", "建议2"]
}`
      }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
}
```

---

### 3.5 Polisher Agent (润色师)

**职责**: 润色和优化生成内容

```typescript
class PolisherAgent extends BaseAgent {
  protected getSystemPrompt(): string {
    return `你是一个专业的标书润色专家。

润色原则：
1. 保持原意不变
2. 使语言更自然流畅
3. 统一术语和风格
4. 消除AI生成痕迹
5. 增强可读性和专业性

润色要求：
- 修正语法错误
- 优化句子结构
- 统一术语使用
- 调整段落节奏
- 增强逻辑连贯性`;
  }
  
  async execute<PolishedContent>(task: AgentTask): Promise<PolishedContent> {
    this.updateStatus(AgentStatus.BUSY);
    
    try {
      const { content, reviewReport, context } = task.input;
      
      // 1. 根据审核报告修正问题
      let polished = await this.fixIssues(content, reviewReport);
      
      // 2. 统一术语
      polished = await this.unifyTerminology(polished, context.terminology);
      
      // 3. 优化语言
      polished = await this.optimizeLanguage(polished);
      
      // 4. 增强连贯性
      polished = await this.enhanceCoherence(polished);
      
      this.updateStatus(AgentStatus.IDLE);
      return polished;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      throw error;
    }
  }
  
  private async fixIssues(content: string, report: ReviewReport): Promise<string> {
    if (report.issues.length === 0) {
      return content;
    }
    
    const messages = [
      {
        role: 'user',
        content: `请根据以下审核报告修正内容：

审核问题：
${report.issues.map(issue => `- ${issue.description}`).join('\n')}

原始内容：
${content}

请返回修正后的内容，保持原有格式和结构。`
      }
    ];
    
    return await this.callAI(messages, { temperature: 0.3 });
  }
}
```

---

## 4. Agent通信机制

### 4.1 事件驱动通信

```typescript
// 事件类型定义
enum AgentEvent {
  CHAPTER_COMPLETED = 'chapter_completed',
  CONTEXT_UPDATED = 'context_updated',
  REVIEW_COMPLETED = 'review_completed',
  POLISH_COMPLETED = 'polish_completed',
  ERROR_OCCURRED = 'error_occurred'
}

// 事件总线
class EventBus {
  private handlers: Map<string, Function[]>;
  
  on(event: string, handler: Function): void {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event).push(handler);
  }
  
  emit(event: string, data: any): void {
    const handlers = this.handlers.get(event) || [];
    handlers.forEach(handler => handler(data));
  }
}

// 写手完成后触发事件
async function onChapterCompleted(chapter: Chapter, content: string) {
  // 1. 生成摘要
  const summary = await generateSummary(content);
  
  // 2. 提取术语
  const terms = await extractTerminology(content);
  
  // 3. 更新共享上下文
  sharedContext.update({
    chapterId: chapter.id,
    summary,
    terms
  });
  
  // 4. 广播事件
  eventBus.emit(AgentEvent.CHAPTER_COMPLETED, {
    chapterId: chapter.id,
    summary,
    terms
  });
}
```

### 4.2 共享上下文访问

```typescript
class SharedContext {
  private data: SharedContextData;
  private lock: ReadWriteLock;
  
  // 读取上下文（支持并发读）
  async getSnapshot(): Promise<SharedContextData> {
    await this.lock.acquireRead();
    try {
      return { ...this.data };
    } finally {
      this.lock.releaseRead();
    }
  }
  
  // 更新上下文（独占写）
  async update(update: ContextUpdate): Promise<void> {
    await this.lock.acquireWrite();
    try {
      // 更新术语表
      if (update.terms) {
        update.terms.forEach(term => {
          this.data.terminology.set(term.name, term);
        });
      }
      
      // 更新章节摘要
      if (update.chapterId && update.summary) {
        this.data.chapterSummaries.set(update.chapterId, update.summary);
      }
      
      // 持久化
      await this.persist();
    } finally {
      this.lock.releaseWrite();
    }
  }
}
```

---

## 5. Agent生命周期管理

### 5.1 生命周期状态

```
┌─────────────────────────────────────────────────────────────┐
│                    Agent生命周期状态机                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌──────┐    create    ┌──────┐    start    ┌──────┐      │
│   │ 无   │ ──────────→ │ 空闲 │ ─────────→ │ 忙碌 │      │
│   └──────┘             └──────┘             └──────┘      │
│                             ↑                   │           │
│                             │ complete          │ error     │
│                             └───────────────────↓           │
│                                             ┌──────┐       │
│                                             │ 错误 │       │
│                                             └──────┘       │
│                                                 │           │
│                                                 │ retry     │
│                                                 ↓           │
│                                             ┌──────┐       │
│                                             │ 空闲 │       │
│                                             └──────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 资源管理

```typescript
class AgentPool {
  private agents: Map<AgentType, BaseAgent[]>;
  private maxAgents: Map<AgentType, number>;
  
  // 获取可用Agent
  async acquire(type: AgentType): Promise<BaseAgent> {
    const agents = this.agents.get(type) || [];
    const idle = agents.find(a => a.status === AgentStatus.IDLE);
    
    if (idle) {
      idle.updateStatus(AgentStatus.BUSY);
      return idle;
    }
    
    // 如果没有空闲Agent，等待或创建新的
    if (agents.length < this.maxAgents.get(type)) {
      return await this.createAgent(type);
    }
    
    // 等待Agent空闲
    return await this.waitForIdle(type);
  }
  
  // 释放Agent
  release(agent: BaseAgent): void {
    agent.updateStatus(AgentStatus.IDLE);
  }
}
```

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
