# GetRec、保险公司政策文档和CARC/RARC的RAG索引关系详解

## 📋 目录
1. [整体关系图](#1-整体关系图)
2. [CARC/RARC代码的双重存储](#2-carcrarc代码的双重存储)
3. [RAG索引的构建过程](#3-rag索引的构建过程)
4. [GetRec如何使用RAG索引](#4-getrec如何使用rag索引)
5. [完整数据流](#5-完整数据流)
6. [索引与RAG构建的关系](#6-索引与rag构建的关系)

---

## 1. 整体关系图

### 🔗 核心组件关系

```
┌─────────────────────────────────────────────────────────────┐
│                    CARC/RARC代码定义                         │
│                                                              │
│  方式1: Dataverse表 (rhail_codedefinitions)                │
│  ├─ 结构化数据                                              │
│  ├─ 代码 + 定义文本                                          │
│  └─ 用于查询和关联                                           │
│                                                              │
│  方式2: RAG索引文档 (filerecs-autocreate)                  │
│  ├─ 文档形式（PDF/DOCX/TXT）                                │
│  ├─ 包含代码定义和处理指南                                   │
│  └─ 用于AI检索和生成推荐                                     │
└─────────────────────────────────────────────────────────────┘
                        │
                        │ 两种方式互补
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ↓                               ↓
┌──────────────────┐          ┌──────────────────┐
│   主流程使用     │          │   GetRec使用     │
│  Dataverse表     │          │   RAG索引        │
│                  │          │                  │
│ - 查询代码定义   │          │ - 检索相关文档   │
│ - 构建查询字符串 │          │ - 生成推荐       │
│ - 关联数据       │          │ - 引用来源       │
└──────────────────┘          └──────────────────┘
```

---

## 2. CARC/RARC代码的双重存储

### 📊 存储方式对比

#### 方式1: Dataverse表（结构化存储）

**表名：** `rhail_codedefinitions`

**数据来源：** `data/rhail_codedefinitions.csv`

**存储内容：**
```csv
rhail_code,rhail_codetype,rhail_definition
"CO-16",984780000,"Claim/service lacks information or has submission/billing error(s)."
"N255",984780001,"Missing/incomplete/invalid billing provider taxonomy."
```

**用途：**
- ✅ 快速查询代码定义
- ✅ 数据关联（外键引用）
- ✅ 结构化数据操作
- ✅ 在Power Apps中显示

**在主流程中的使用：**
```json
{
  "Get_CARC_def": {
    "type": "OpenApiConnection",
    "inputs": {
      "parameters": {
        "entityName": "rhail_codedefinitions",
        "$filter": "rhail_code eq 'CO-16'"
      }
    }
  }
}
```

**获取结果：**
```json
{
  "rhail_code": "CO-16",
  "rhail_definition": "Claim/service lacks information or has submission/billing error(s).",
  "rhail_codedefinitionid": "guid-123"
}
```

---

#### 方式2: RAG索引（文档存储）

**索引名：** `filerecs-autocreate`

**存储内容：**
- PDF/DOCX文档包含CARC/RARC代码定义
- 代码的处理指南和最佳实践
- 代码相关的理赔处理步骤

**文档示例：**
```
文档：CARC_RARC_Definitions.pdf

内容：
"CO-16: Claim/service lacks information or has submission/billing error(s).
Usage: Do not use this code for claims attachment(s)/other documentation.
At least one Remark Code must be provided..."

"N255: Missing/incomplete/invalid billing provider taxonomy.
Action: Verify provider taxonomy code from enrollment documentation.
Contact the provider directly if needed..."
```

**用途：**
- ✅ 语义搜索（向量搜索）
- ✅ AI上下文增强
- ✅ 生成详细推荐
- ✅ 引用来源追踪

---

### 🔄 两种存储方式的协作

```
主流程处理理赔
   │
   ├─ 从835文件解析出CARC代码（如"CO-16"）
   │
   ├─ 查询Dataverse表获取代码定义
   │   └─ 得到："Claim/service lacks information..."
   │
   ├─ 构建查询字符串
   │   └─ "CO-16: Claim/service lacks information..."
   │
   └─ 调用GetRec，传入：
       ├─ CARC代码："CO-16"
       ├─ CARC定义："CO-16: Claim/service lacks information..."
       ├─ RARC代码："N255"
       └─ 保险公司："North Dakota Medicaid"
   ↓
GetRec使用RAG索引
   │
   ├─ 接收查询参数
   │
   ├─ 调用Azure OpenAI + RAG
   │   └─ 配置data_sources: filerecs-autocreate索引
   │
   ├─ Azure OpenAI自动检索
   │   ├─ 使用向量搜索查找相关文档
   │   ├─ 可能找到：
   │   │   ├─ CARC CO-16的详细定义文档
   │   │   ├─ RARC N255的处理指南
   │   │   ├─ North Dakota Medicaid政策文档
   │   │   └─ 用户反馈文档
   │   └─ 返回Top 5相关文档片段
   │
   └─ 基于检索到的文档生成推荐
```

---

## 3. RAG索引的构建过程

### 🏗️ 索引构建的完整流程

```
步骤1: 文档准备和上传
   │
   ├─ 用户准备文档：
   │   ├─ 理赔处理指南.pdf
   │   ├─ North_Dakota_Medicaid_Policy.docx
   │   ├─ CARC_RARC_Definitions.pdf
   │   └─ SubmittedUserFeedback.txt
   │
   └─ 上传到Azure Blob Storage的recs容器
       └─ 路径：recs/文档名.pdf
   ↓
步骤2: 索引器配置
   │
   └─ 索引器：indexerrec-autocreate
       ├─ 数据源：recs容器
       ├─ 目标索引：filerecs-autocreate
       ├─ 调度：每天运行一次（P1D）
       └─ 配置：
           ├─ dataToExtract: "contentAndMetadata"
           └─ parsingMode: "default"
   ↓
步骤3: 索引器自动运行
   │
   ├─ 检测recs容器中的新文档
   │
   ├─ 提取文档内容：
   │   ├─ PDF → 提取文本
   │   ├─ DOCX → 提取文本
   │   └─ TXT → 直接读取
   │
   ├─ 文档分块（Chunking）：
   │   └─ 将长文档分割成较小的片段
   │
   ├─ 生成向量嵌入（Vector Embedding）：
   │   ├─ 使用Azure OpenAI的嵌入模型
   │   ├─ 将文本转换为向量
   │   └─ 存储到contentVector字段
   │
   └─ 存储到索引：
       ├─ content: 文本内容
       ├─ metadata_storage_name: 文件名
       ├─ metadata_storage_path: 文件路径
       ├─ contentVector: 向量嵌入
       └─ 其他元数据字段
   ↓
步骤4: 索引可用于RAG检索
   │
   └─ GetRec可以查询这个索引
```

---

### 📐 索引结构详解

**索引配置（recsIndexProfile.json）：**

```json
{
  "name": "filerecs-autocreate",
  "fields": [
    {
      "name": "content",           // 文档文本内容（可搜索）
      "type": "Edm.String",
      "searchable": true,
      "analyzer": "standard.lucene"
    },
    {
      "name": "metadata_storage_name",  // 文件名（可搜索）
      "type": "Edm.String",
      "searchable": true
    },
    {
      "name": "metadata_storage_path",  // 文件路径（主键）
      "type": "Edm.String",
      "key": true
    },
    {
      "name": "contentVector",     // 向量嵌入（向量搜索）
      "type": "Collection(Edm.Single)",
      "vectorSearchProfile": "..."  // 向量搜索配置
    },
    {
      "name": "merged_content",    // 合并后的内容
      "type": "Edm.String",
      "searchable": true
    },
    {
      "name": "text",              // 文本集合
      "type": "Collection(Edm.String)",
      "searchable": true
    }
  ]
}
```

**关键字段说明：**

1. **content** - 文档的主要文本内容
   - 可搜索：支持关键词搜索
   - 用于RAG检索

2. **contentVector** - 向量嵌入
   - 用于语义搜索
   - 找到语义相似的内容，即使关键词不完全匹配

3. **metadata_storage_name** - 文件名
   - 可搜索：可以按文件名搜索
   - 用于识别文档来源

4. **metadata_storage_path** - 文件路径
   - 主键：唯一标识文档
   - 用于引用来源

---

## 4. GetRec如何使用RAG索引

### 🔍 GetRec的RAG检索配置

**从代码中可以看到：**

```json
{
  "data_sources": [
    {
      "type": "azure_search",
      "parameters": {
        "endpoint": "Azure Search URL",
        "index_name": "filerecs-autocreate",  // 使用Recs索引
        "authentication": {
          "type": "api_key",
          "key": "API Key"
        },
        "strictness": 5,              // 严格度：5（最高）
        "top_n_documents": 5,         // 返回Top 5文档
        "fields_mapping": {
          "content_fields_separator": "\n",
          "content_fields": [
            "content",                // 主要搜索字段
            "metadata_storage_name"    // 文件名也参与搜索
          ],
          "filepath_field": "metadata_storage_path",  // 用于引用
          "title_field": "metadata_storage_name",     // 标题字段
          "url_field": "url",
          "vector_fields": [
            "contentVector"           // 向量搜索字段
          ]
        }
      }
    }
  ]
}
```

### 🎯 检索过程详解

#### 步骤1: GetRec构建查询

**输入：**
- CARC代码：`"CO-16"`
- CARC定义：`"CO-16: Claim/service lacks information..."`
- RARC代码：`"N255"`
- 保险公司：`"North Dakota Medicaid"`

**构建的查询：**
```
"Provide recommendations on how to correct the claim for North Dakota Medicaid. 
In the steps, if applicable add steps on how to find items and documentation. 
Only include User Feedback (in the final response) from SubmittedUserFeedback.txt 
if it matches the CARC code and make sure to bold the text. 
CARC: CO-16. RARC: N255"
```

#### 步骤2: Azure OpenAI接收查询

**请求配置：**
```json
{
  "messages": [
    {
      "role": "system",
      "content": "You are an AI assistant that helps people fix medical billing files..."
    },
    {
      "role": "user",
      "content": "Provide recommendations... CARC: CO-16. RARC: N255"
    }
  ],
  "data_sources": [{
    "type": "azure_search",
    "parameters": {
      "index_name": "filerecs-autocreate"
    }
  }]
}
```

#### 步骤3: Azure OpenAI自动RAG检索

**内部处理：**

1. **查询向量化**
   ```
   用户查询文本
   → Azure OpenAI嵌入模型
   → 生成查询向量
   ```

2. **向量相似度搜索**
   ```
   查询向量
   → 在contentVector字段中搜索
   → 计算余弦相似度
   → 找到Top 5最相似的文档片段
   ```

3. **可能检索到的文档片段：**

   **片段1：** 来自CARC_RARC_Definitions.pdf
   ```
   "CO-16: Claim/service lacks information or has submission/billing error(s).
   Usage: Do not use this code for claims attachment(s)/other documentation.
   At least one Remark Code must be provided..."
   ```

   **片段2：** 来自North_Dakota_Medicaid_Policy.docx
   ```
   "North Dakota Medicaid requires complete billing provider taxonomy information.
   For claims with missing taxonomy, contact ND Medicaid Claim Customer Service 
   at 844-848-0844 or 701-328-2325..."
   ```

   **片段3：** 来自理赔处理指南.pdf
   ```
   "When encountering CARC CO-16 with RARC N255:
   1. Verify provider taxonomy code
   2. Check enrollment documentation
   3. Update claim with correct taxonomy
   4. Resubmit within 30 days..."
   ```

   **片段4：** 来自SubmittedUserFeedback.txt
   ```
   "For CARC CO-16 with N255, we found that updating the taxonomy code 
   and resubmitting within 30 days resolved the issue. 
   Contact information: NDMMISEDI@nd.gov"
   ```

   **片段5：** 来自RARC_N255_Guide.pdf
   ```
   "N255: Missing/incomplete/invalid billing provider taxonomy.
   Action: Verify provider taxonomy code from enrollment documentation.
   Ensure all required fields are accurately completed..."
   ```

4. **文档片段作为上下文**
   ```
   检索到的5个文档片段
   → 添加到AI的上下文窗口
   → 作为生成推荐的依据
   ```

#### 步骤4: AI生成推荐

**基于检索到的文档，AI生成：**

```html
<h2>CARC: CO-16</h2>
<p><strong>Claim/service lacks information or has submission/billing error(s).</strong></p>
<p>Usage: Do not use this code for claims attachment(s)/other documentation...</p>

<h3>RARC:</h3>
<p>N255: Missing/incomplete/invalid billing provider taxonomy.</p>

<h3>Recommendations and Steps:</h3>
<p><strong>Identify Missing Information:</strong></p>
<p>Review the claim to determine what specific information is missing, 
focusing on the billing provider taxonomy.</p>

<p><strong>Gather Required Information:</strong></p>
<p>Ensure you have the correct billing provider taxonomy. 
This can be found in the provider's enrollment documentation 
or by contacting the provider directly.</p>

<p><strong>Contact Customer Service if Needed:</strong></p>
<p>If you encounter issues, contact ND Medicaid Claim Customer Service 
at 844-848-0844 or 701-328-2325, or email NDMMISEDI@nd.gov.</p>

<h3>User Feedback:</h3>
<p><strong>For CARC CO-16 with N255, updating the taxonomy code and 
resubmitting within 30 days resolved the issue.</strong></p>
```

**同时返回引用来源：**
```json
{
  "citations": [
    {
      "content": "CO-16: Claim/service lacks information...",
      "title": "CARC_RARC_Definitions.pdf",
      "filepath": "recs/CARC_RARC_Definitions.pdf",
      "chunk_id": "chunk-123"
    },
    {
      "content": "North Dakota Medicaid requires complete...",
      "title": "North_Dakota_Medicaid_Policy.docx",
      "filepath": "recs/North_Dakota_Medicaid_Policy.docx",
      "chunk_id": "chunk-456"
    }
  ]
}
```

---

## 5. 完整数据流

### 🔄 从代码定义到推荐生成的完整流程

```
┌─────────────────────────────────────────────────────────────┐
│ 阶段1: 代码定义准备                                          │
└─────────────────────────────────────────────────────────────┘
         │
         ├─ 方式1: 导入CSV到Dataverse
         │   └─ data/rhail_codedefinitions.csv
         │       → rhail_codedefinitions表
         │
         └─ 方式2: 上传文档到Blob Storage
             └─ CARC_RARC_Definitions.pdf
                 → recs容器
                 → 索引器处理
                 → filerecs-autocreate索引
         ↓
┌─────────────────────────────────────────────────────────────┐
│ 阶段2: 理赔文件解析                                         │
└─────────────────────────────────────────────────────────────┘
         │
         ├─ 用户上传835文件到SharePoint
         │
         ├─ 主流程解析文件
         │   └─ 提取CARC代码："CO-16"
         │
         └─ 查询Dataverse表
             └─ 获取代码定义："Claim/service lacks information..."
         ↓
┌─────────────────────────────────────────────────────────────┐
│ 阶段3: 构建GetRec查询                                       │
└─────────────────────────────────────────────────────────────┘
         │
         ├─ 主流程构建查询字符串
         │   └─ "CO-16: Claim/service lacks information..."
         │
         └─ 调用GetRec子流程
             ├─ 输入：CARC代码 + 定义 + RARC + 保险公司
             └─ 准备RAG检索
         ↓
┌─────────────────────────────────────────────────────────────┐
│ 阶段4: RAG检索和推荐生成                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ├─ GetRec调用Azure OpenAI
         │   └─ 配置data_sources: filerecs-autocreate
         │
         ├─ Azure OpenAI自动RAG检索
         │   ├─ 向量搜索查询索引
         │   ├─ 找到Top 5相关文档片段：
         │   │   ├─ CARC CO-16定义文档
         │   │   ├─ RARC N255处理指南
         │   │   ├─ 保险公司政策文档
         │   │   ├─ 理赔处理指南
         │   │   └─ 用户反馈文档
         │   └─ 文档片段作为上下文
         │
         ├─ GPT-4o生成推荐
         │   ├─ 基于检索到的文档
         │   ├─ 结合代码定义
         │   ├─ 参考保险公司政策
         │   └─ 包含用户反馈
         │
         └─ 返回HTML推荐 + 引用来源
         ↓
┌─────────────────────────────────────────────────────────────┐
│ 阶段5: 存储和显示                                           │
└─────────────────────────────────────────────────────────────┘
         │
         ├─ 存储推荐到Dataverse
         │   └─ ClaimAdjustmentCodes表
         │
         └─ Power Apps显示给用户
             ├─ HTML格式推荐
             └─ 引用来源链接
```

---

## 6. 索引与RAG构建的关系

### 🏗️ RAG构建的关键组件

#### 6.1 索引结构（Index Structure）

**作用：** 定义如何存储和检索文档

**关键配置：**
- **字段定义**：哪些字段可搜索、哪些存储向量
- **分析器**：如何分词和索引文本
- **向量配置**：如何生成和搜索向量嵌入

**示例：**
```json
{
  "fields": [
    {
      "name": "content",
      "searchable": true,        // 支持关键词搜索
      "analyzer": "standard.lucene"
    },
    {
      "name": "contentVector",
      "vectorSearchProfile": "..."  // 支持向量搜索
    }
  ]
}
```

---

#### 6.2 索引器（Indexer）

**作用：** 从数据源提取文档并构建索引

**配置：**
```json
{
  "name": "indexerrec-autocreate",
  "dataSourceName": "recs容器",
  "targetIndexName": "filerecs-autocreate",
  "schedule": {
    "interval": "P1D"  // 每天运行一次
  },
  "parameters": {
    "dataToExtract": "contentAndMetadata",
    "parsingMode": "default"
  }
}
```

**工作流程：**
1. 扫描Blob Storage的recs容器
2. 提取文档文本内容
3. 文档分块（Chunking）
4. 生成向量嵌入
5. 存储到索引

---

#### 6.3 向量嵌入（Vector Embedding）

**作用：** 将文本转换为数值向量，支持语义搜索

**生成过程：**
```
文档文本
   ↓
Azure OpenAI嵌入模型（text-embedding-ada-002或类似）
   ↓
向量数组 [0.123, -0.456, 0.789, ...]
   ↓
存储到contentVector字段
```

**为什么需要向量：**
- **语义理解**：找到语义相似的内容，即使关键词不完全匹配
- **上下文关联**：理解"CARC CO-16"和"理赔信息缺失"的关联
- **多语言支持**：即使表达方式不同，也能找到相关内容

---

#### 6.4 RAG检索配置

**在GetRec中的配置：**

```json
{
  "data_sources": [{
    "type": "azure_search",
    "parameters": {
      "index_name": "filerecs-autocreate",
      "strictness": 5,              // 严格度：越高越精确
      "top_n_documents": 5,         // 返回文档数量
      "fields_mapping": {
        "content_fields": ["content", "metadata_storage_name"],
        "vector_fields": ["contentVector"]  // 使用向量搜索
      }
    }
  }]
}
```

**strictness参数说明：**
- **1-3**：宽松匹配，可能返回不太相关的结果
- **4-5**：严格匹配，只返回高度相关的结果
- **本项目使用5**：确保推荐基于最相关的文档

---

### 🔗 索引构建与RAG的关系

```
文档上传
   ↓
索引器运行
   ├─ 提取文本 → content字段
   ├─ 生成向量 → contentVector字段
   └─ 存储元数据 → metadata_*字段
   ↓
索引构建完成
   ↓
GetRec调用时
   ├─ 查询文本 → 转换为查询向量
   ├─ 向量搜索 → 在contentVector中查找相似向量
   ├─ 关键词搜索 → 在content字段中查找关键词
   ├─ 综合评分 → 结合向量相似度和关键词匹配
   └─ 返回Top 5 → 最相关的文档片段
   ↓
AI使用检索结果
   ├─ 文档片段作为上下文
   ├─ 结合用户查询
   └─ 生成个性化推荐
```

---

## 🎯 关键关系总结

### 1. CARC/RARC代码的双重角色

**在Dataverse表中：**
- 结构化数据
- 快速查询
- 数据关联

**在RAG索引中：**
- 文档形式
- 详细说明
- 处理指南
- 用于AI检索

### 2. 保险公司政策文档的作用

**存储位置：** RAG索引（filerecs-autocreate）

**作用：**
- 提供保险公司特定的政策信息
- 包含联系方式和处理流程
- 支持个性化推荐

**检索方式：**
- 向量搜索：基于语义相似度
- 关键词搜索：基于保险公司名称

### 3. GetRec与RAG索引的交互

**GetRec的作用：**
- 接收CARC/RARC代码和保险公司信息
- 构建查询
- 调用Azure OpenAI + RAG

**RAG索引的作用：**
- 存储所有相关文档
- 支持向量和关键词搜索
- 返回最相关的文档片段

**Azure OpenAI的作用：**
- 自动检索RAG索引
- 使用检索结果作为上下文
- 生成个性化推荐

### 4. 索引构建的重要性

**索引结构决定：**
- 哪些字段可以搜索
- 如何生成向量
- 检索的准确性

**索引器决定：**
- 如何提取文档内容
- 如何分块文档
- 如何生成向量嵌入

**文档质量决定：**
- 检索结果的准确性
- AI生成推荐的质量
- 推荐的实用性

---

## 💡 最佳实践

### 1. 文档组织建议

**推荐的文档结构：**
```
recs容器/
├─ CARC_RARC_Definitions.pdf      # 代码定义
├─ [保险公司名]_Policy.docx       # 各保险公司政策
├─ Claim_Processing_Guide.pdf     # 理赔处理指南
├─ SubmittedUserFeedback.txt      # 用户反馈
└─ [其他相关文档]
```

### 2. 文档内容建议

**每个文档应包含：**
- 清晰的标题和结构
- 相关的CARC/RARC代码
- 具体的处理步骤
- 联系信息和资源链接

### 3. 索引更新策略

**定期更新：**
- 每天自动运行索引器
- 上传新文档后手动触发
- 政策更新时及时更新文档

### 4. 查询优化

**GetRec查询优化：**
- 包含CARC代码和定义
- 包含RARC代码
- 明确指定保险公司
- 使用strictness=5确保精确匹配

---

## 📊 实际示例：完整流程

### 场景：处理CARC CO-16和RARC N255的理赔

```
1. 文档准备阶段
   ├─ 上传CARC_RARC_Definitions.pdf到recs容器
   │   └─ 包含CO-16和N255的详细说明
   ├─ 上传North_Dakota_Medicaid_Policy.docx
   │   └─ 包含ND Medicaid的理赔政策
   └─ 索引器运行，构建索引

2. 理赔解析阶段
   ├─ 用户上传835文件
   ├─ 主流程解析出CARC="CO-16", RARC="N255"
   └─ 查询Dataverse获取代码定义

3. GetRec调用阶段
   ├─ 主流程调用GetRec
   │   └─ 传入：CARC="CO-16", RARC="N255", 保险公司="ND Medicaid"
   │
   ├─ GetRec构建查询
   │   └─ "Provide recommendations... CARC: CO-16. RARC: N255"
   │
   └─ 调用Azure OpenAI + RAG

4. RAG检索阶段
   ├─ Azure OpenAI自动检索filerecs-autocreate索引
   │
   ├─ 向量搜索找到相关文档：
   │   ├─ CARC_RARC_Definitions.pdf中的CO-16定义
   │   ├─ CARC_RARC_Definitions.pdf中的N255说明
   │   ├─ North_Dakota_Medicaid_Policy.docx中的政策
   │   ├─ Claim_Processing_Guide.pdf中的处理步骤
   │   └─ SubmittedUserFeedback.txt中的用户反馈
   │
   └─ 返回Top 5文档片段

5. 推荐生成阶段
   ├─ GPT-4o基于检索结果生成推荐
   │   ├─ 结合CO-16的定义
   │   ├─ 结合N255的说明
   │   ├─ 参考ND Medicaid的政策
   │   ├─ 包含处理步骤
   │   └─ 包含用户反馈（如果有）
   │
   └─ 返回HTML推荐 + 引用来源

6. 结果存储和显示
   ├─ 存储到ClaimAdjustmentCodes表
   └─ Power Apps显示给用户
```

---

## ✅ 总结

### 核心关系

1. **CARC/RARC代码定义**
   - Dataverse表：结构化数据，快速查询
   - RAG索引：文档形式，详细说明，AI检索

2. **保险公司政策文档**
   - 存储在RAG索引中
   - 通过向量搜索检索
   - 用于生成个性化推荐

3. **GetRec子流程**
   - 接收代码和保险公司信息
   - 调用Azure OpenAI + RAG
   - 生成基于文档的推荐

4. **RAG索引构建**
   - 索引结构定义存储方式
   - 索引器提取和索引文档
   - 向量嵌入支持语义搜索
   - RAG配置控制检索行为

### 关键优势

✅ **双重存储**：结构化数据 + 文档检索  
✅ **语义搜索**：向量嵌入找到语义相关内容  
✅ **个性化推荐**：基于保险公司特定政策  
✅ **可追溯性**：每个推荐都有引用来源  
✅ **可更新性**：更新文档即可更新知识库  

---

**这就是GetRec、保险公司政策文档、CARC/RARC代码和RAG索引之间的完整关系！**
