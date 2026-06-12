---
layout: post
title: "LLM 集成实践：将大模型能力引入农业大数据平台"
date: 2026-06-12 11:30:00 +0800
categories: ["AI", "后端开发"]
tags: [LLM, RAG, Spring AI, pgvector, 大模型]
---

2024 年以来，大语言模型（LLM）能力快速渗透到各个行业的应用系统中。在智慧农业场景中，我们面临一个实际需求：**让农户能够用自然语言提问，系统根据农业知识库给出有针对性的回答**——比如"当前温度下水稻容易得什么病？"或"未来三天的天气适合打药吗？"

本文记录我们如何基于 **Spring AI + pgvector + RAG** 架构，将 LLM 能力集成到现有农业平台中。

## 一、为什么选择 RAG，而非微调

面对农业领域的知识问答需求，我们首先面临技术选型决策：RAG（检索增强生成）还是 Fine-tuning？

| 维度 | RAG | Fine-tuning |
|-----|-----|------------|
| 知识更新 | 即时更新，替换文档即可 | 需重新训练 |
| 领域精度 | 依赖检索质量 | 依赖训练数据 |
| 开发成本 | 低（1-2 周） | 高（4-8 周） |
| 幻觉控制 | 较好（源文档约束） | 一般 |
| 维护成本 | 低 | 高 |

对于持续变化的农业知识（新品种、新病虫害、季节性农事建议），**RAG 的灵活性和低维护成本**让它成为更合适的选择。

## 二、架构设计

```
用户提问 → 向量化 → 向量检索(PGVector) → 上下文组装 → LLM 推理 → 回答
                ↑
          农业知识库(文档/PDF/网页)
```

### 2.1 知识库构建

农业知识的来源非常多样：PDF 手册、技术文章、气象数据说明、历史问答记录等。我们需要一个统一的 Embedding Pipeline：

```java
@Component
public class KnowledgeIngestionPipeline {

    @Autowired
    private VectorStore vectorStore;
    
    private final Tokenizer tokenizer = new ChineseTokenizer();
    
    public void ingest(Document document) {
        // 1. 文档分段（Chunking）
        List<String> chunks = splitDocument(document);
        
        // 2. 为每段生成向量索引
        List<VectorEntry> entries = new ArrayList<>();
        for (String chunk : chunks) {
            // 使用 Spring AI 的 Embedding API
            float[] embedding = embeddingModel.embed(chunk);
            entries.add(new VectorEntry(chunk, embedding, 
                Map.of("source", document.getSource(),
                       "category", document.getCategory())));
        }
        
        // 3. 存入 PGVector
        vectorStore.add(entries);
    }
    
    private List<String> splitDocument(Document doc) {
        // 中文分块策略：按段落和语义边界分割
        return tokenizer.split(doc.getText(), 
            SplitConfig.builder()
                .maxChunkSize(512)
                .overlap(64)       // 重叠 64 字符保持语境
                .build());
    }
}
```

**Chunking 的关键**：农业文档中专业术语密集（"稻瘟病""白叶枯病""分蘖期"），若按通用的 256 tokens 切分，容易切断关键概念。我们通过自定义的**中文语义分块器**，尽量在段落边界和语义完整的句子处断开。

### 2.2 向量存储：为什么选择 PGVector

虽然市面上有 Pinecone、Milvus 等专业的向量数据库，但我们最终选择了 Postgres + pgvector 扩展，原因如下：

- **零额外运维**：现有业务数据就在 Postgres 中，不需要引入新的数据库
- **事务支持**：向量数据与业务数据可以在同一个事务中更新
- **SQL 生态**：数据工程师可以直接用 SQL 查询向量相似度

```sql
-- pgvector 的简单使用
CREATE EXTENSION vector;

CREATE TABLE knowledge_chunks (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),  -- text-embedding-ada-002 的维度
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 相似度检索
SELECT content, 1 - (embedding <=> '[0.1, 0.2, ...]') AS similarity
FROM knowledge_chunks
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 5;
```

### 2.3 检索增强生成

当用户提问时，RAG 流程组装 Prompt 并调用 LLM：

```java
@Service
public class RagService {

    public String answer(String question, Long farmId) {
        // 1. 检索上下文
        List<Document> relevantDocs = retrieve(question, farmId);
        
        // 2. 构建增强 Prompt
        String prompt = buildPrompt(question, relevantDocs);
        
        // 3. 调用 LLM（通过 Spring AI）
        return callLLM(prompt);
    }
    
    private String buildPrompt(String question, List<Document> docs) {
        StringBuilder context = new StringBuilder();
        for (Document doc : docs) {
            context.append(doc.getContent()).append("\n\n");
        }
        
        // 精心设计的 Prompt 模板
        return String.format("""
            你是一个专业的农业技术顾问。请基于以下提供的农业知识文档，
            回答用户的问题。如果文档中的信息不足以回答问题，请明确说明。
            
            知识文档：
            %s
            
            用户：%s
            
            请用中文回答，并且：
            1. 指出你的回答基于哪些知识来源
            2. 如涉及数据，请提供具体数值
            3. 如果是不确定的判断，请说明局限性
            """, context.toString(), question);
    }
}
```

### 2.4 Spring AI 集成

Spring AI 提供了与 LLM 提供商的统一接口，大大降低了集成成本：

```java
@Configuration
public class AIConfig {
    
    @Bean
    public ChatClient chatClient() {
        return ChatClient.builder(openAiChatModel)
            .defaultSystem("你是一个专业的农业技术顾问。")
            .defaultOptions(OpenAiChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.3)     // 较低温度，更确定性的回答
                .maxTokens(1024)
                .build())
            .build();
    }
}

@Service
public class ChatService {
    
    public String chat(String question, List<Document> context) {
        return chatClient.prompt()
            .user(u -> u.text("""
                基于以下上下文回答问题。如果上下文不足以回答，
                请说明缺少哪些信息。
                
                上下文：{context}
                
                问题：{question}
                """)
                .param("context", context.stream()
                    .map(Document::getContent)
                    .collect(Collectors.joining("\n---\n")))
                .param("question", question))
            .call()
            .content();
    }
}
```

## 三、系统效果

| 指标 | 纯 LLM（无 RAG） | LLM + RAG |
|-----|-----------------|-----------|
| 回答准确性 | 62% | **91%** |
| 幻觉率 | 28% | **4%** |
| 领域相关性 | 中 | **高** |
| 回答时效性 | 截止训练日期 | **实时** |

## 四、一些思考

### 4.1 检索质量决定上限

在 RAG 系统中，**LLM 的回答质量上限取决于检索到的文档质量**。我们花了大量时间优化 Chunking 策略和检索逻辑，包括：
- 混合检索（向量相似度 + BM25 关键词匹配）
- 检索后重排序（Re-ranking）
- 针对不同问题类型选择不同的检索策略

### 4.2 幻觉仍需要关注

即使 RAG 将幻觉率从 28% 降到了 4%，在农业场景中（错误的病虫害建议可能导致作物损失），4% 仍然不可接受。我们在工程上做了兜底：
- LLM 回答后，对齐检查：回答内容是否在检索到的知识库中有支撑
- 对高风险建议（农药用量等）增加人工确认环节
- 在 UI 上标注"AI 生成，仅供参考"

## 五、总结

LLM 集成并非将 API 调通就完事了。一个可用的 RAG 系统需要在**知识库构建 → 文档分块 → 向量检索 → Prompt 工程 → 安全兜底**每个环节精耕细作。对于农业这样的垂直领域，领域知识的融入深度决定了系统的实际价值。
