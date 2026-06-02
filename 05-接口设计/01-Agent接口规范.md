# Agent接口规范

## 1. 概述

### 1.1 设计原则

- **一致性**: 所有Agent遵循统一的接口规范
- **可扩展**: 支持新增Agent类型
- **类型安全**: 使用TypeScript类型定义
- **错误处理**: 统一的错误处理机制

---

## 2. 基础接口

### 2.1 Agent接口

```typescript
/**
 * Agent基础接口
 */
interface IAgent {
  /** Agent唯一标识 */
  readonly id: string;
  
  /** Agent类型 */
  readonly type: AgentType;
  
  /** 当前状态 */
  readonly status: AgentStatus;
  
  /**
   * 执行任务
   * @param task 任务输入
   * @returns 任务输出
   */
  execute<T>(task: AgentTask): Promise<T>;
  
  /**
   * 获取Agent能力
   */
  getCapabilities(): string[];
  
  /**
   * 健康检查
   */
  healthCheck(): Promise<boolean>;
}

/**
 * Agent类型枚举
 */
enum AgentType {
  ANALYST = 'analyst',       // 分析师
  OUTLINER = 'outliner',    // 大纲师
  WRITER = 'writer',         // 写手
  REVIEWER = 'reviewer',     // 审核员
  POLISHER = 'polisher',     // 润色师
  ILLUSTRATOR = 'illustrator' // 配图师
}

/**
 * Agent状态枚举
 */
enum AgentStatus {
  IDLE = 'idle',           // 空闲
  BUSY = 'busy',           // 忙碌
  ERROR = 'error',         // 错误
  PAUSED = 'paused',       // 暂停
  STOPPED = 'stopped'      // 停止
}

/**
 * Agent任务
 */
interface AgentTask {
  /** 任务ID */
  id: string;
  
  /** 任务类型 */
  type: TaskType;
  
  /** 任务输入 */
  input: Record<string, any>;
  
  /** 任务选项 */
  options?: AgentTaskOptions;
}

/**
 * Agent任务选项
 */
interface AgentTaskOptions {
  /** 超时时间(ms) */
  timeout?: number;
  
  /** 重试次数 */
  retries?: number;
  
  /** 优先级 */
  priority?: number;
  
  /** 是否流式输出 */
  stream?: boolean;
}
```

---

## 3. 各Agent接口定义

### 3.1 Analyst Agent接口

```typescript
/**
 * 分析师Agent任务输入
 */
interface AnalystTaskInput {
  /** 招标文件 */
  tender_doc: {
    id: string;
    file_name: string;
    file_type: 'pdf' | 'word' | 'image';
    file_path: string;
  };
  
  /** 分析选项 */
  options?: {
    extract_images?: boolean;
    ocr_enabled?: boolean;
    language?: 'zh' | 'en';
  };
}

/**
 * 分析师Agent任务输出
 */
interface AnalystTaskOutput {
  /** 分析结果ID */
  id: string;
  
  /** 关联的招标文件ID */
  tender_doc_id: string;
  
  /** 项目名称 */
  project_name: string;
  
  /** 项目概述 */
  project_overview: string;
  
  /** 技术要求 */
  technical_requirements: TechnicalRequirement[];
  
  /** 评分标准 */
  scoring_criteria: ScoringCategory[];
  
  /** 资质要求 */
  qualification_requirements: QualificationRequirement[];
  
  /** 关键要点 */
  key_points: string[];
  
  /** 元数据 */
  metadata: {
    pages_count: number;
    extraction_time: number;
    confidence_score: number;
  };
}

/**
 * 技术要求
 */
interface TechnicalRequirement {
  id: string;
  category: string;
  requirement: string;
  description: string;
  is_mandatory: boolean;
}

/**
 * 评分大类
 */
interface ScoringCategory {
  id: string;
  category: string;
  total_score: number;
  items: ScoringItem[];
}

/**
 * 评分细项
 */
interface ScoringItem {
  id: string;
  item: string;
  score: number;
  description: string;
  scoring_standard?: string;
}
```

### 3.2 Outliner Agent接口

```typescript
/**
 * 大纲师Agent任务输入
 */
interface OutlinerTaskInput {
  /** 分析结果 */
  analysis: AnalystTaskOutput;
  
  /** 大纲选项 */
  options?: {
    max_depth?: number;
    align_with_scoring?: boolean;
    user_outline?: OutlineItem[];
  };
}

/**
 * 大纲师Agent任务输出
 */
interface OutlinerTaskOutput {
  /** 大纲ID */
  id: string;
  
  /** 关联的标书ID */
  bid_id: string;
  
  /** 大纲条目 */
  items: OutlineItem[];
  
  /** 版本号 */
  version: number;
  
  /** 元数据 */
  metadata: {
    total_chapters: number;
    max_depth: number;
    requirement_coverage: number;
  };
}

/**
 * 大纲条目
 */
interface OutlineItem {
  /** 章节编号 */
  id: string;
  
  /** 父节点ID */
  parent_id?: string;
  
  /** 章节标题 */
  title: string;
  
  /** 章节描述 */
  description: string;
  
  /** 层级 */
  level: number;
  
  /** 排序序号 */
  order: number;
  
  /** 对应的评分点ID */
  requirement_ids: string[];
  
  /** 子节点 */
  children?: OutlineItem[];
}
```

### 3.3 Writer Agent接口

```typescript
/**
 * 写手Agent任务输入
 */
interface WriterTaskInput {
  /** 章节信息 */
  chapter: {
    id: string;
    title: string;
    description: string;
    level: number;
    parent_id?: string;
  };
  
  /** 共享上下文 */
  context: SharedContextData;
  
  /** 编排决策 */
  content_plan?: ContentPlan;
  
  /** 生成选项 */
  options?: {
    temperature?: number;
    max_words?: number;
    regenerate?: boolean;
    regenerate_requirement?: string;
  };
}

/**
 * 写手Agent任务输出
 */
interface WriterTaskOutput {
  /** 内容ID */
  id: string;
  
  /** 章节ID */
  chapter_id: string;
  
  /** 章节标题 */
  title: string;
  
  /** 生成的内容 */
  content: string;
  
  /** 字数统计 */
  word_count: number;
  
  /** 元数据 */
  metadata: {
    generation_time: number;
    knowledge_used: string[];
    terminology_used: string[];
    references_made: Reference[];
  };
}

/**
 * 编排决策
 */
interface ContentPlan {
  chapter_id: string;
  use_table: boolean;
  table_purpose?: string;
  use_image: boolean;
  image_style?: string;
  image_prompt?: string;
  use_mermaid: boolean;
  mermaid_code?: string;
  knowledge_ids: string[];
}

/**
 * 共享上下文数据
 */
interface SharedContextData {
  projectOverview: string;
  technicalRequirements: string[];
  scoringCriteria: ScoringCategory[];
  terminology: Map<string, TermEntry>;
  chapterSummaries: Map<string, ChapterSummary>;
  styleGuide: StyleGuide;
  references: Reference[];
  requirementCoverage: Map<string, RequirementCoverage>;
}

/**
 * 术语条目
 */
interface TermEntry {
  term: string;
  definition: string;
  aliases: string[];
  usage_count: number;
  first_used_in: string;
}

/**
 * 章节摘要
 */
interface ChapterSummary {
  chapter_id: string;
  summary: string;
  key_points: string[];
  word_count: number;
  generated_at: Date;
}
```

### 3.4 Reviewer Agent接口

```typescript
/**
 * 审核员Agent任务输入
 */
interface ReviewerTaskInput {
  /** 章节内容列表 */
  chapters: WriterTaskOutput[];
  
  /** 大纲 */
  outline: OutlinerTaskOutput;
  
  /** 共享上下文 */
  context: SharedContextData;
  
  /** 审核选项 */
  options?: {
    strict_mode?: boolean;
    auto_fix?: boolean;
    min_score?: number;
  };
}

/**
 * 审核员Agent任务输出
 */
interface ReviewerTaskOutput {
  /** 报告ID */
  id: string;
  
  /** 关联的标书ID */
  bid_id: string;
  
  /** 总体评分 */
  overall_score: number;
  
  /** 是否通过 */
  passed: boolean;
  
  /** 各维度评分 */
  dimensions: ReviewDimension[];
  
  /** 问题列表 */
  issues: ReviewIssue[];
  
  /** 改进建议 */
  suggestions: string[];
}

/**
 * 审核维度
 */
interface ReviewDimension {
  name: string;
  score: number;
  weight: number;
  issues: string[];
}

/**
 * 审核问题
 */
interface ReviewIssue {
  id: string;
  chapter_id: string;
  type: IssueType;
  severity: 'high' | 'medium' | 'low';
  description: string;
  location?: string;
  suggestion: string;
}

/**
 * 问题类型
 */
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

### 3.5 Polisher Agent接口

```typescript
/**
 * 润色师Agent任务输入
 */
interface PolisherTaskInput {
  /** 章节内容列表 */
  chapters: WriterTaskOutput[];
  
  /** 审核报告 */
  review_report: ReviewerTaskOutput;
  
  /** 共享上下文 */
  context: SharedContextData;
  
  /** 润色选项 */
  options?: {
    polish_level: 'light' | 'medium' | 'heavy';
    preserve_structure: boolean;
    focus_areas?: string[];
  };
}

/**
 * 润色师Agent任务输出
 */
interface PolisherTaskOutput {
  /** 润色结果列表 */
  results: PolishedContent[];
}

/**
 * 润色后的内容
 */
interface PolishedContent {
  /** 内容ID */
  id: string;
  
  /** 章节ID */
  chapter_id: string;
  
  /** 章节标题 */
  title: string;
  
  /** 原始内容 */
  original_content: string;
  
  /** 润色后内容 */
  polished_content: string;
  
  /** 变更记录 */
  changes: ContentChange[];
  
  /** 元数据 */
  metadata: {
    polish_time: number;
    change_count: number;
    word_count_before: number;
    word_count_after: number;
  };
}

/**
 * 内容变更
 */
interface ContentChange {
  type: 'add' | 'modify' | 'delete';
  location: string;
  original: string;
  modified: string;
  reason: string;
}
```

---

## 4. 错误处理规范

### 4.1 错误类型

```typescript
/**
 * Agent错误基类
 */
class AgentError extends Error {
  constructor(
    public code: string,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'AgentError';
  }
}

/**
 * 分析错误
 */
class AnalysisError extends AgentError {
  constructor(message: string, details?: any) {
    super('ANALYSIS_ERROR', message, details);
  }
}

/**
 * 大纲生成错误
 */
class OutlineError extends AgentError {
  constructor(message: string, details?: any) {
    super('OUTLINE_ERROR', message, details);
  }
}

/**
 * 内容生成错误
 */
class ContentGenerationError extends AgentError {
  constructor(message: string, details?: any) {
    super('CONTENT_GENERATION_ERROR', message, details);
  }
}

/**
 * 审核错误
 */
class ReviewError extends AgentError {
  constructor(message: string, details?: any) {
    super('REVIEW_ERROR', message, details);
  }
}

/**
 * 润色错误
 */
class PolishError extends AgentError {
  constructor(message: string, details?: any) {
    super('POLISH_ERROR', message, details);
  }
}
```

### 4.2 错误码定义

```typescript
enum ErrorCode {
  // 通用错误
  UNKNOWN_ERROR = 'UNKNOWN_ERROR',
  TIMEOUT_ERROR = 'TIMEOUT_ERROR',
  CANCELLED_ERROR = 'CANCELLED_ERROR',
  
  // 文档解析错误
  DOCUMENT_PARSE_FAILED = 'DOCUMENT_PARSE_FAILED',
  DOCUMENT_NOT_FOUND = 'DOCUMENT_NOT_FOUND',
  DOCUMENT_FORMAT_UNSUPPORTED = 'DOCUMENT_FORMAT_UNSUPPORTED',
  
  // AI服务错误
  AI_SERVICE_ERROR = 'AI_SERVICE_ERROR',
  AI_RATE_LIMIT = 'AI_RATE_LIMIT',
  AI_INVALID_RESPONSE = 'AI_INVALID_RESPONSE',
  
  // 内容生成错误
  CONTENT_GENERATION_FAILED = 'CONTENT_GENERATION_FAILED',
  CONTENT_TOO_SHORT = 'CONTENT_TOO_SHORT',
  CONTENT_INVALID_FORMAT = 'CONTENT_INVALID_FORMAT',
  
  // 上下文错误
  CONTEXT_LOAD_FAILED = 'CONTEXT_LOAD_FAILED',
  CONTEXT_UPDATE_FAILED = 'CONTEXT_UPDATE_FAILED',
  
  // 验证错误
  VALIDATION_FAILED = 'VALIDATION_FAILED',
  MISSING_REQUIRED_FIELD = 'MISSING_REQUIRED_FIELD',
  INVALID_FIELD_VALUE = 'INVALID_FIELD_VALUE'
}
```

---

## 5. 事件规范

### 5.1 事件类型

```typescript
enum AgentEvent {
  // 生命周期事件
  AGENT_STARTED = 'agent:started',
  AGENT_STOPPED = 'agent:stopped',
  AGENT_ERROR = 'agent:error',
  
  // 任务事件
  TASK_SUBMITTED = 'task:submitted',
  TASK_STARTED = 'task:started',
  TASK_COMPLETED = 'task:completed',
  TASK_FAILED = 'task:failed',
  TASK_CANCELLED = 'task:cancelled',
  TASK_PAUSED = 'task:paused',
  TASK_RESUMED = 'task:resumed',
  
  // 上下文事件
  CONTEXT_UPDATED = 'context:updated',
  CHAPTER_COMPLETED = 'chapter:completed',
  TERM_ADDED = 'term:added',
  SUMMARY_UPDATED = 'summary:updated'
}

/**
 * 事件数据
 */
interface EventData {
  event: AgentEvent;
  timestamp: Date;
  source: string;
  data: any;
}
```

---

## 6. 配置规范

### 6.1 Agent配置

```typescript
interface AgentConfig {
  /** Agent ID */
  id: string;
  
  /** Agent类型 */
  type: AgentType;
  
  /** AI服务配置 */
  ai: {
    provider: string;
    model: string;
    temperature: number;
    max_tokens: number;
  };
  
  /** 超时配置 */
  timeout: {
    task: number;
    ai_call: number;
  };
  
  /** 重试配置 */
  retry: {
    max_attempts: number;
    delay: number;
    backoff: number;
  };
}

/**
 * 全局配置
 */
interface GlobalConfig {
  /** 并发配置 */
  concurrency: {
    max_agents: number;
    agent_limits: Map<AgentType, number>;
  };
  
  /** 存储配置 */
  storage: {
    database_path: string;
    file_storage_path: string;
  };
  
  /** 日志配置 */
  logging: {
    level: string;
    file: string;
  };
}
```

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
