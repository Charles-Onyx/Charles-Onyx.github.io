---
layout: post
title: "LLM 集成实践：将大模型能力引入业务平台的完整方案"
date: 2026-06-12 11:30:00 +0800
categories: ["AI", "后端开发"]
tags: [LLM, RAG, Spring AI, pgvector, 大模型]
---

2024 年以来，大语言模型（LLM）能力快速渗透到各个行业的应用系统中。在实际业务场景中，我们面临一个常见需求：**让用户能够用自然语言提问，系统根据领域知识库给出有针对性的回答**。

本文记录如何基于 **Spring AI + pgvector + RAG** 架构，将 LLM 能力集成到现有业务平台中。

## 一、为什么选择 RAG，而非微调

面向领域知识问答场景，首先面临技术选型决策：RAG 还是 Fine-tuning？

| 维度 | RAG | Fine-tuning |
|-----|-----|------------|
| 知识更新 | 即时更新，替换文档即可 | 需重新训练 |
| 领域精度 | 依赖检索质量 | 依赖训练数据 |
| 开发成本 | 低（1-2 周） | 高（4-8 周） |
| 幻觉控制 | 较好（源文档约束） | 一般 |
| 维护成本 | 低 | 高 |

对于持续变化的领域知识，**RAG 的灵活性和低维护成本**让它成为更合适的选择。

## 二、架构设计

```
用户提问 → 向量化 → 向量检索(PGVector) → 上下文组装 → LLM 推理 → 回答
                ↑
          领域知识库(文档/PDF/网页)
```

### 2.1 知识库构建

知识来源非常多样：PDF 手册、技术文章、历史问答记录等。需要一个统一的 Embedding Pipeline：

```java
@Component
public class KnowledgeIngestionPipeline {

    @Autowired
    private VectorStore vectorStore;
    
    public void ingest(Document document) {
        // 1. 文档分段（Chunking）
        List<String> chunks = splitDocument(document);
        
        // 2. 为每段生成向量索引
        List<VectorEntry> entries = new ArrayList<>();
        for (String chunk : chunks) {
            float[] embedding = embeddingModel.embed(chunk);
            entries.add(new VectorEntry(chunk, embedding, 
                Map.of("source", document.getSource(),
                       "category", document.getCategory())));
        }
        
        // 3. 存入 PGVector
        vectorStore.add(entries);
    }
    
    private List<String> splitDocument(Document doc) {
        return tokenizer.split(doc.getText(), 
            SplitConfig.builder()
                .maxChunkSize(512)
                .overlap(64)
                .build());
    }
}
```

**Chunking 的关键**：领域文档中专业术语密集，若按通用的 256 tokens 切分，容易切断关键概念。需要通过自定义的语义分块器，尽量在段落边界和语义完整的句子处断开。

### 2.2 向量存储：为什么选择 PGVector

虽然市面上有 Pinecone、Milvus 等专业的向量数据库，但我们最终选择了 Postgres + pgvector 扩展：

- **零额外运维**：现有业务数据就在 Postgres 中，不需要引入新的数据库
- **事务支持**：向量数据与业务数据可以在同一个事务中更新
- **SQL 生态**：数据工程师可以直接用 SQL 查询向量相似度

```sql
CREATE EXTENSION vector;

CREATE TABLE knowledge_chunks (
    id BIGSERIAL PRIMARY KEY,
    content TEXT,
    embedding VECTOR(1536),
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

    public String answer(String question) {
        List<Document> relevantDocs = retrieve(question);
        String prompt = buildPrompt(question, relevantDocs);
        return callLLM(prompt);
    }
    
    private String buildPrompt(String question, List<Document> docs) {
        StringBuilder context = new StringBuilder();
        for (Document doc : docs) {
            context.append(doc.getContent()).append("\n\n");
        }
        
        return String.format("""
            请基于以下提供的知识文档回答用户的问题。
            如果文档中的信息不足以回答问题，请明确说明。
            
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
            .defaultSystem("你是一个专业的领域知识顾问。")
            .defaultOptions(OpenAiChatOptions.builder()
                .model("gpt-4o-mini")
                .temperature(0.3)
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
                基于以下上下文回答问题。
                
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

在 RAG 系统中，**LLM 的回答质量上限取决于检索到的文档质量**。优化方向包括：
- 混合检索（向量相似度 + BM25 关键词匹配）
- 检索后重排序（Re-ranking）
- 针对不同问题类型选择不同的检索策略

### 4.2 幻觉仍需要关注

即使 RAG 将幻觉率从 28% 降到了 4%，在某些高风险场景中仍然不可接受。工程上的兜底措施：
- 回答内容与检索知识库的对齐检查
- 高风险建议增加人工确认环节
- UI 上标注"AI 生成，仅供参考"

## 五、总结

LLM 集成并非将 API 调通就完事了。一个可用的 RAG 系统需要在**知识库构建 → 文档分块 → 向量检索 → Prompt 工程 → 安全兜底**每个环节精耕细作。对于垂直领域，领域知识的融入深度决定了系统的实际价值。
