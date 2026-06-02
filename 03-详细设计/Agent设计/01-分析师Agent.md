# Analyst Agent (分析师) 详细设计

## 1. Agent概述

### 1.1 职责定义

分析师Agent是标书生成流程的第一个环节，负责深度解析招标文件，提取结构化信息，为后续大纲生成和内容编写提供基础数据。

### 1.2 核心能力

```
┌─────────────────────────────────────────────────────────────┐
│                    Analyst Agent 核心能力                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   📄 文档解析                                                │
│      - PDF文本提取                                          │
│      - Word文档解析                                         │
│      - 图片OCR识别                                          │
│                                                             │
│   🔍 信息提取                                                │
│      - 项目基本信息                                         │
│      - 技术要求                                             │
│      - 评分标准                                             │
│      - 资质要求                                             │
│                                                             │
│   📊 结构化输出                                              │
│      - JSON格式数据                                         │
│      - 标准化字段                                           │
│      - 验证和校验                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 接口设计

### 2.1 输入接口

```typescript
interface AnalystTaskInput {
  tender_doc: {
    id: string;
    file_name: string;
    file_type: 'pdf' | 'word' | 'image';
    file_path: string;
  };
  options?: {
    extract_images?: boolean;
    ocr_enabled?: boolean;
    language?: 'zh' | 'en';
  };
}
```

### 2.2 输出接口

```typescript
interface TenderAnalysis {
  id: string;
  tender_doc_id: string;
  
  // 项目基本信息
  project_name: string;
  project_overview: string;
  project_location?: string;
  project_budget?: string;
  project_duration?: string;
  
  // 技术要求
  technical_requirements: TechnicalRequirement[];
  
  // 评分标准
  scoring_criteria: ScoringCategory[];
  
  // 资质要求
  qualification_requirements: QualificationRequirement[];
  
  // 商务条件
  business_conditions: BusinessCondition[];
  
  // 关键要点
  key_points: string[];
  
  // 元数据
  metadata: {
    pages_count: number;
    extraction_time: number;
    confidence_score: number;
  };
}

interface TechnicalRequirement {
  id: string;
  category: string;
  requirement: string;
  description: string;
  is_mandatory: boolean;
}

interface ScoringCategory {
  id: string;
  category: string;
  total_score: number;
  items: ScoringItem[];
}

interface ScoringItem {
  id: string;
  item: string;
  score: number;
  description: string;
  scoring_standard?: string;
}

interface QualificationRequirement {
  id: string;
  type: 'company' | 'personnel' | 'certification' | 'experience';
  requirement: string;
  description: string;
  is_mandatory: boolean;
}

interface BusinessCondition {
  id: string;
  type: 'payment' | 'warranty' | 'delivery' | 'other';
  condition: string;
  description: string;
}
```

### 2.3 输出JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["project_name", "project_overview", "technical_requirements", "scoring_criteria"],
  "properties": {
    "project_name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 200
    },
    "project_overview": {
      "type": "string",
      "minLength": 50,
      "maxLength": 2000
    },
    "technical_requirements": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/TechnicalRequirement"
      }
    },
    "scoring_criteria": {
      "type": "array",
      "items": {
        "$ref": "#/definitions/ScoringCategory"
      }
    }
  }
}
```

---

## 3. 核心流程

### 3.1 处理流程图

```
┌─────────────────────────────────────────────────────────────┐
│                    Analyst Agent 处理流程                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   输入: 招标文件                                             │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ① 文档预处理                                        │  │
│   │    - 文件格式识别                                    │  │
│   │    - 文本提取 (PDF/Word/OCR)                        │  │
│   │    - 文本清洗和规范化                                │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ② 分段处理                                          │  │
│   │    - 按章节/标题分段                                 │  │
│   │    - 识别文档结构                                    │  │
│   │    - 标记关键段落                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ③ 信息提取 (AI调用)                                 │  │
│   │    - 提取项目基本信息                                │  │
│   │    - 提取技术要求                                    │  │
│   │    - 提取评分标准                                    │  │
│   │    - 提取资质要求                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ④ 结果合并                                          │  │
│   │    - 合并各段提取结果                                │  │
│   │    - 去重和排序                                      │  │
│   │    - 补充缺失信息                                    │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   ┌─────────────────────────────────────────────────────┐  │
│   │ ⑤ 验证和修正                                        │  │
│   │    - 格式验证                                        │  │
│   │    - 逻辑验证                                        │  │
│   │    - 自动修正                                        │  │
│   └─────────────────────────────────────────────────────┘  │
│      │                                                      │
│      ↓                                                      │
│   输出: TenderAnalysis                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. Prompt设计

### 4.1 系统提示词

```typescript
const ANALYST_SYSTEM_PROMPT = `你是一个专业的招标文件分析师，拥有丰富的招投标行业经验。

## 职责
1. 深入分析招标文件，提取关键信息
2. 识别技术要求和评分标准
3. 提取资质要求和商务条件
4. 总结项目概述和关键要点

## 分析原则
- 信息提取要完整，不遗漏重要信息
- 分类要准确，符合招投标行业规范
- 评分标准要精确提取，包括分值和评分细则
- 关键要点要突出，便于后续编写参考

## 输出要求
- 使用结构化JSON格式
- 字段命名规范、类型正确
- 数组内容有序、无重复
- 确保信息来源可追溯`;
```

### 4.2 项目信息提取提示词

```typescript
const PROJECT_INFO_PROMPT = `请从以下招标文件内容中提取项目基本信息：

{content}

请提取以下信息：
1. project_name: 项目名称（必须）
2. project_overview: 项目概述，包括项目背景、目标、范围（100-500字）
3. project_location: 项目地点
4. project_budget: 项目预算
5. project_duration: 项目工期

返回JSON格式：
{
  "project_name": "",
  "project_overview": "",
  "project_location": "",
  "project_budget": "",
  "project_duration": ""
}`;
```

### 4.3 技术要求提取提示词

```typescript
const TECHNICAL_REQUIREMENTS_PROMPT = `请从以下招标文件内容中提取技术要求：

{content}

请提取所有技术要求，每个要求包括：
1. id: 唯一标识（使用T1, T2, T3...格式）
2. category: 技术类别
3. requirement: 要求名称
4. description: 详细描述
5. is_mandatory: 是否为强制性要求

返回JSON格式：
{
  "technical_requirements": [
    {
      "id": "T1",
      "category": "",
      "requirement": "",
      "description": "",
      "is_mandatory": true
    }
  ]
}`;
```

### 4.4 评分标准提取提示词

```typescript
const SCORING_CRITERIA_PROMPT = `请从以下招标文件内容中提取评分标准：

{content}

请提取完整的评分标准，包括：
1. 评分大类（如技术方案、商务报价、项目经验等）
2. 每个大类下的评分细项
3. 每个细项的分值
4. 评分标准/评分细则

返回JSON格式：
{
  "scoring_criteria": [
    {
      "id": "S1",
      "category": "技术方案",
      "total_score": 60,
      "items": [
        {
          "id": "S1-1",
          "item": "技术路线",
          "score": 20,
          "description": "技术路线合理、先进",
          "scoring_standard": "优秀20分，良好15分，一般10分"
        }
      ]
    }
  ]
}`;
```

---

## 5. 实现代码

### 5.1 Agent类定义

```typescript
import { BaseAgent, AgentType, AgentStatus } from '../base/BaseAgent';
import { AIService } from '../../services/AIService';
import { SharedContext } from '../../context/SharedContext';
import { Logger } from '../../utils/Logger';

export class AnalystAgent extends BaseAgent {
  private documentParser: DocumentParser;
  
  constructor(
    aiService: AIService,
    sharedContext: SharedContext,
    logger: Logger
  ) {
    super({
      id: 'analyst-agent',
      type: AgentType.ANALYST,
      aiService,
      sharedContext,
      logger
    });
    
    this.documentParser = new DocumentParser();
  }
  
  protected getSystemPrompt(): string {
    return ANALYST_SYSTEM_PROMPT;
  }
  
  async execute(input: AnalystTaskInput): Promise<TenderAnalysis> {
    this.updateStatus(AgentStatus.BUSY);
    const startTime = Date.now();
    
    try {
      this.logger.info(`开始分析招标文件: ${input.tender_doc.file_name}`);
      
      // 1. 文档预处理
      const content = await this.preprocessDocument(input.tender_doc);
      this.logger.info(`文档预处理完成，文本长度: ${content.length}`);
      
      // 2. 分段处理
      const segments = this.segmentDocument(content);
      this.logger.info(`文档分段完成，共 ${segments.length} 个段落`);
      
      // 3. 并发提取信息
      const [projectInfo, techReqs, scoring, qualifications, business] = 
        await Promise.all([
          this.extractProjectInfo(segments),
          this.extractTechnicalRequirements(segments),
          this.extractScoringCriteria(segments),
          this.extractQualificationRequirements(segments),
          this.extractBusinessConditions(segments)
        ]);
      
      // 4. 合并结果
      const analysis: TenderAnalysis = {
        id: this.generateId(),
        tender_doc_id: input.tender_doc.id,
        ...projectInfo,
        technical_requirements: techReqs,
        scoring_criteria: scoring,
        qualification_requirements: qualifications,
        business_conditions: business,
        key_points: this.extractKeyPoints(content),
        metadata: {
          pages_count: this.countPages(content),
          extraction_time: Date.now() - startTime,
          confidence_score: this.calculateConfidence(projectInfo, techReqs, scoring)
        }
      };
      
      // 5. 验证结果
      this.validateAnalysis(analysis);
      
      this.updateStatus(AgentStatus.IDLE);
      this.logger.info(`招标文件分析完成，耗时: ${analysis.metadata.extraction_time}ms`);
      
      return analysis;
    } catch (error) {
      this.updateStatus(AgentStatus.ERROR);
      this.logger.error(`招标文件分析失败: ${error.message}`);
      throw error;
    }
  }
  
  private async preprocessDocument(doc: TenderDocumentInput): Promise<string> {
    switch (doc.file_type) {
      case 'pdf':
        return await this.documentParser.parsePDF(doc.file_path);
      case 'word':
        return await this.documentParser.parseWord(doc.file_path);
      case 'image':
        return await this.documentParser.parseImage(doc.file_path);
      default:
        throw new Error(`Unsupported file type: ${doc.file_type}`);
    }
  }
  
  private segmentDocument(content: string): string[] {
    // 按标题、章节进行分段
    const segments: string[] = [];
    const lines = content.split('\n');
    let currentSegment = '';
    
    for (const line of lines) {
      if (this.isHeading(line) && currentSegment.length > 100) {
        segments.push(currentSegment.trim());
        currentSegment = line;
      } else {
        currentSegment += '\n' + line;
      }
    }
    
    if (currentSegment.trim()) {
      segments.push(currentSegment.trim());
    }
    
    return segments;
  }
  
  private async extractProjectInfo(segments: string[]): Promise<Partial<TenderAnalysis>> {
    const content = segments.join('\n\n');
    const messages = [
      { role: 'user', content: PROJECT_INFO_PROMPT.replace('{content}', content) }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response);
  }
  
  private async extractTechnicalRequirements(segments: string[]): Promise<TechnicalRequirement[]> {
    const content = segments.join('\n\n');
    const messages = [
      { role: 'user', content: TECHNICAL_REQUIREMENTS_PROMPT.replace('{content}', content) }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response).technical_requirements;
  }
  
  private async extractScoringCriteria(segments: string[]): Promise<ScoringCategory[]> {
    const content = segments.join('\n\n');
    const messages = [
      { role: 'user', content: SCORING_CRITERIA_PROMPT.replace('{content}', content) }
    ];
    
    const response = await this.callAI(messages, { temperature: 0.1 });
    return JSON.parse(response).scoring_criteria;
  }
  
  private validateAnalysis(analysis: TenderAnalysis): void {
    if (!analysis.project_name) {
      throw new Error('项目名称不能为空');
    }
    
    if (!analysis.project_overview || analysis.project_overview.length < 50) {
      throw new Error('项目概述过于简短');
    }
    
    if (analysis.scoring_criteria.length === 0) {
      throw new Error('未提取到评分标准');
    }
  }
  
  private calculateConfidence(
    projectInfo: any,
    techReqs: any[],
    scoring: any[]
  ): number {
    let score = 0;
    
    if (projectInfo.project_name) score += 20;
    if (projectInfo.project_overview) score += 20;
    if (techReqs.length > 0) score += 30;
    if (scoring.length > 0) score += 30;
    
    return score;
  }
}
```

---

## 6. 错误处理

### 6.1 错误类型

```typescript
enum AnalystErrorType {
  DOCUMENT_PARSE_FAILED = 'DOCUMENT_PARSE_FAILED',
  EXTRACTION_FAILED = 'EXTRACTION_FAILED',
  VALIDATION_FAILED = 'VALIDATION_FAILED',
  AI_SERVICE_ERROR = 'AI_SERVICE_ERROR'
}

class AnalystError extends Error {
  constructor(
    public type: AnalystErrorType,
    message: string,
    public details?: any
  ) {
    super(message);
    this.name = 'AnalystError';
  }
}
```

### 6.2 重试策略

```typescript
// 文档解析失败：重试3次
// 信息提取失败：降级处理，返回部分结果
// AI服务错误：重试3次，指数退避
// 验证失败：自动修正后重试
```

---

## 7. 性能优化

### 7.1 并发处理

- 多个信息提取任务并发执行
- 大文档分段并行处理

### 7.2 缓存策略

- 相同文档缓存解析结果
- 分析结果支持增量更新

### 7.3 资源管理

- 控制并发AI调用数量
- 及时释放文档解析资源

---

*文档版本: v1.0*
*最后更新: 2026-06-02*
