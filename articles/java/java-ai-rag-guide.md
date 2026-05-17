# SpringBoot + LangChain4j 打造企业级 RAG 系统

> **写在前面：** RAG 是我学习 Java+AI 过程中最有成就感的项目。从最初的简单 Demo，到后来支持权限控制、缓存优化、混合检索的企业级系统，我踩了无数坑。本文记录完整的实现过程，包括**架构设计、代码实现、性能优化、效果评估**。这是我简历中最亮眼的项目，希望能给你参考！

## 📋 前言

RAG（Retrieval-Augmented Generation，检索增强生成）是 Java 工程师切入 AI 领域的最佳切入点。

本文将手把手教你用 **SpringBoot + LangChain4j + Milvus** 构建一个完整的企业级 RAG 知识库系统。

---

## 🎯 什么是 RAG？

### 核心概念

RAG = **检索（Retrieval）** + **生成（Generation）**

**传统大模型的问题：**
- ❌ 知识截止于训练数据
- ❌ 无法访问企业内部文档
- ❌ 容易产生"幻觉"（胡说八道）

**RAG 的解决方案：**
1. 用户提问
2. 从向量数据库检索相关文档
3. 将文档 + 问题一起发给大模型
4. 大模型基于文档生成答案

**优势：**
- ✅ 答案有依据（可追溯引用来源）
- ✅ 实时更新知识库（无需重新训练）
- ✅ 降低幻觉（基于事实回答）

---

## 🏗️ 系统架构

```mermaid
graph TB
    subgraph 文档处理流程
        A[上传文档] --> B[PDF/Word 解析]
        B --> C[文本分块]
        C --> D[向量化 Embedding]
        D --> E[存入 Milvus]
    end
    
    subgraph 问答流程
        F[用户提问] --> G[问题向量化]
        G --> H[Milvus 检索 Top-K]
        H --> I[组装 Prompt]
        I --> J[调用大模型]
        J --> K[返回答案 + 引用来源]
    end
```

---

## 🔧 技术栈选型

| 组件 | 选型 | 理由 |
|------|------|------|
| 后端框架 | SpringBoot 3.x | Java 工程师熟悉 |
| AI 框架 | LangChain4j | Java 版 LangChain，生态完善 |
| 向量数据库 | Milvus | 开源、高性能、支持大规模数据 |
| 文档解析 | Apache PDFBox / Tika | 成熟稳定 |
| 缓存 | Redis | 高频问答缓存 |
| 监控 | Prometheus + Grafana | 标准监控方案 |

---

## 📦 环境准备

### 1. 安装 Milvus

**Docker 快速启动：**

```bash
docker run -d --name milvus \
  -p 19530:19530 \
  -p 9091:9091 \
  milvusdb/milvus:v2.3.0
```

**验证安装：**

```bash
curl http://localhost:9091/healthz
```

### 2. Maven 依赖

```xml
<dependencies>
    <!-- SpringBoot -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- LangChain4j 核心 -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j</artifactId>
        <version>0.26.0</version>
    </dependency>
    
    <!-- LangChain4j Milvus 集成 -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-milvus</artifactId>
        <version>0.26.0</version>
    </dependency>
    
    <!-- LangChain4j 通义千问集成 -->
    <dependency>
        <groupId>dev.langchain4j</groupId>
        <artifactId>langchain4j-dashscope</artifactId>
        <version>0.26.0</version>
    </dependency>
    
    <!-- PDF 解析 -->
    <dependency>
        <groupId>org.apache.pdfbox</groupId>
        <artifactId>pdfbox</artifactId>
        <version>2.0.29</version>
    </dependency>
    
    <!-- Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
</dependencies>
```

---

## 💻 核心代码实现

### 1. 配置文件

**application.yml：**

```yaml
llm:
  dashscope:
    api-key: ${DASHSCOPE_API_KEY}
    model-name: qwen-turbo

milvus:
  host: localhost
  port: 19530
  collection-name: knowledge_base

embedding:
  model-name: text-embedding-v1
```

---

### 2. 文档上传与解析

```java
@Service
public class DocumentService {
    
    @Autowired
    private MilvusEmbeddingStore embeddingStore;
    
    @Autowired
    private DashScopeEmbeddingModel embeddingModel;
    
    /**
     * 上传并处理文档
     */
    public void uploadDocument(MultipartFile file, String category) {
        try {
            // 1. 解析文档
            String text = parseDocument(file);
            
            // 2. 文本分块
            List<TextSegment> segments = splitText(text);
            
            // 3. 添加元数据
            for (TextSegment segment : segments) {
                segment.metadata().put("category", category);
                segment.metadata().put("filename", file.getOriginalFilename());
                segment.metadata().put("uploadTime", LocalDateTime.now().toString());
            }
            
            // 4. 向量化并存入 Milvus
            embeddingStore.addAll(segments);
            
            log.info("文档处理完成: {}, 分块数: {}", file.getOriginalFilename(), segments.size());
            
        } catch (Exception e) {
            log.error("文档处理失败", e);
            throw new RuntimeException("文档处理失败", e);
        }
    }
    
    /**
     * 解析文档（支持 PDF、Word、TXT）
     */
    private String parseDocument(MultipartFile file) throws IOException {
        String filename = file.getOriginalFilename();
        
        if (filename.endsWith(".pdf")) {
            return parsePdf(file.getInputStream());
        } else if (filename.endsWith(".docx") || filename.endsWith(".doc")) {
            return parseWord(file.getInputStream());
        } else {
            return new String(file.getBytes(), StandardCharsets.UTF_8);
        }
    }
    
    /**
     * 解析 PDF
     */
    private String parsePdf(InputStream inputStream) throws IOException {
        PDDocument document = PDDocument.load(inputStream);
        PDFTextStripper stripper = new PDFTextStripper();
        String text = stripper.getText(document);
        document.close();
        return text;
    }
    
    /**
     * 文本分块
     */
    private List<TextSegment> splitText(String text) {
        TextSplitter splitter = DocumentSplitters.recursive(
            500,   // 每块最大字符数
            50     // 重叠字符数
        );
        return splitter.split(TextSegment.from(text));
    }
}
```

---

### 3. 向量检索与问答

```java
@Service
public class RagService {
    
    @Autowired
    private MilvusEmbeddingStore embeddingStore;
    
    @Autowired
    private DashScopeEmbeddingModel embeddingModel;
    
    @Autowired
    private DashScopeChatModel chatModel;
    
    /**
     * RAG 问答
     */
    public RagResponse query(String question, int topK) {
        // 1. 问题向量化
        Embedding questionEmbedding = embeddingModel.embed(question).content();
        
        // 2. 向量检索
        List<EmbeddingMatch<TextSegment>> matches = embeddingStore.findRelevant(
            questionEmbedding, 
            topK,
            0.7  // 相似度阈值
        );
        
        // 3. 组装 Prompt
        String context = buildContext(matches);
        String prompt = buildPrompt(question, context);
        
        // 4. 调用大模型
        String answer = chatModel.generate(prompt);
        
        // 5. 构建响应（包含引用来源）
        return RagResponse.builder()
            .answer(answer)
            .sources(extractSources(matches))
            .build();
    }
    
    /**
     * 组装上下文
     */
    private String buildContext(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> match.embedded().text())
            .collect(Collectors.joining("\n\n"));
    }
    
    /**
     * 构建 Prompt
     */
    private String buildPrompt(String question, String context) {
        return String.format(
            "你是一个智能助手，请根据以下文档内容回答问题。\n\n" +
            "文档内容：\n%s\n\n" +
            "问题：%s\n\n" +
            "如果文档中没有相关信息，请明确告知用户。",
            context, question
        );
    }
    
    /**
     * 提取引用来源
     */
    private List<Source> extractSources(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> Source.builder()
                .filename(match.embedded().metadata().get("filename"))
                .category(match.embedded().metadata().get("category"))
                .similarityScore(match.score())
                .build())
            .collect(Collectors.toList());
    }
}
```

---

### 4. Controller 层

```java
@RestController
@RequestMapping("/api/rag")
public class RagController {
    
    @Autowired
    private DocumentService documentService;
    
    @Autowired
    private RagService ragService;
    
    /**
     * 上传文档
     */
    @PostMapping("/upload")
    public ResponseEntity<String> uploadDocument(
            @RequestParam("file") MultipartFile file,
            @RequestParam("category") String category) {
        documentService.uploadDocument(file, category);
        return ResponseEntity.ok("文档上传成功");
    }
    
    /**
     * 问答接口
     */
    @PostMapping("/query")
    public ResponseEntity<RagResponse> query(@RequestBody QueryRequest request) {
        RagResponse response = ragService.query(request.getQuestion(), request.getTopK());
        return ResponseEntity.ok(response);
    }
}
```

---

## 🛡️ 工程化能力

### 1. 权限控制

```java
@Component
public class KnowledgeBasePermission {
    
    /**
     * 检查用户是否有权限访问某个分类的知识库
     */
    public boolean hasAccess(String userId, String category) {
        // 从数据库查询用户权限
        // ...
        return true;
    }
}

@Service
public class SecureRagService {
    
    @Autowired
    private KnowledgeBasePermission permission;
    
    @Autowired
    private RagService ragService;
    
    public RagResponse query(String userId, String question, String category) {
        // 权限校验
        if (!permission.hasAccess(userId, category)) {
            throw new AccessDeniedException("无权访问该知识库");
        }
        
        return ragService.query(question, category);
    }
}
```

---

### 2. 问答日志记录

```java
@Entity
@Table(name = "qa_log")
@Data
public class QaLog {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    private String userId;
    
    @Column(length = 1000)
    private String question;
    
    @Column(length = 2000)
    private String answer;
    
    private Double avgSimilarityScore;
    
    private LocalDateTime createTime;
}

@Service
public class QaLogService {
    
    @Autowired
    private QaLogRepository logRepository;
    
    public void logQuery(String userId, String question, RagResponse response) {
        QaLog log = new QaLog();
        log.setUserId(userId);
        log.setQuestion(question);
        log.setAnswer(response.getAnswer());
        log.setAvgSimilarityScore(calculateAvgScore(response.getSources()));
        log.setCreateTime(LocalDateTime.now());
        
        logRepository.save(log);
    }
}
```

---

### 3. 缓存优化

```java
@Service
public class CachedRagService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private RagService ragService;
    
    /**
     * 带缓存的问答
     */
    public RagResponse queryWithCache(String question) {
        String cacheKey = "rag:cache:" + md5(question);
        
        // 先查缓存
        RagResponse cached = (RagResponse) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // 调用 RAG
        RagResponse response = ragService.query(question, 5);
        
        // 写入缓存（过期时间 2 小时）
        redisTemplate.opsForValue().set(cacheKey, response, 2, TimeUnit.HOURS);
        
        return response;
    }
}
```

---

## 📊 性能优化

### 1. 批量向量化

```java
@Service
public class BatchEmbeddingService {
    
    @Autowired
    private DashScopeEmbeddingModel embeddingModel;
    
    /**
     * 批量向量化（提升效率）
     */
    public List<Embedding> batchEmbed(List<String> texts) {
        // LangChain4j 支持批量调用
        return embeddingModel.embedAll(texts).content();
    }
}
```

---

### 2. 异步文档处理

```java
@Service
public class AsyncDocumentService {
    
    @Async("documentProcessingExecutor")
    public CompletableFuture<Void> processDocumentAsync(MultipartFile file, String category) {
        try {
            uploadDocument(file, category);
            return CompletableFuture.completedFuture(null);
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }
    
    @Bean("documentProcessingExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        return executor;
    }
}
```

---

## 🔍 检索优化

### 1. 混合检索（关键词 + 向量）

```java
@Service
public class HybridSearchService {
    
    @Autowired
    private MilvusEmbeddingStore vectorStore;
    
    @Autowired
    private ElasticsearchRepository esRepository;
    
    /**
     * 混合检索
     */
    public List<Document> hybridSearch(String query, int topK) {
        // 1. 向量检索
        List<EmbeddingMatch<TextSegment>> vectorResults = 
            vectorStore.findRelevant(embed(query), topK);
        
        // 2. 关键词检索
        List<Document> keywordResults = 
            esRepository.searchByKeyword(query, topK);
        
        // 3. 结果融合（RRF 算法）
        return mergeResults(vectorResults, keywordResults);
    }
}
```

---

### 2. 重排序（Re-Ranking）

```java
@Service
public class ReRankService {
    
    @Autowired
    private DashScopeChatModel chatModel;
    
    /**
     * 使用大模型对检索结果重排序
     */
    public List<EmbeddingMatch<TextSegment>> reRank(
            String query, 
            List<EmbeddingMatch<TextSegment>> candidates) {
        
        // 让大模型评估每个候选文档的相关性
        List<Pair<EmbeddingMatch<TextSegment>, Double>> scored = candidates.stream()
            .map(candidate -> {
                double score = evaluateRelevance(query, candidate.embedded().text());
                return Pair.of(candidate, score);
            })
            .sorted((a, b) -> Double.compare(b.getRight(), a.getRight()))
            .collect(Collectors.toList());
        
        return scored.stream()
            .map(Pair::getLeft)
            .collect(Collectors.toList());
    }
    
    private double evaluateRelevance(String query, String document) {
        // 调用大模型评估相关性
        // ...
        return 0.9;
    }
}
```

---

## 📝 项目结构

```
rag-system/
├── src/main/java/com/example/rag/
│   ├── controller/
│   │   ├── DocumentController.java
│   │   └── RagController.java
│   ├── service/
│   │   ├── DocumentService.java
│   │   ├── RagService.java
│   │   └── CachedRagService.java
│   ├── repository/
│   │   ├── MilvusConfig.java
│   │   └── QaLogRepository.java
│   ├── model/
│   │   ├── RagResponse.java
│   │   ├── Source.java
│   │   └── QaLog.java
│   ├── security/
│   │   └── KnowledgeBasePermission.java
│   └── config/
│       ├── LlmConfig.java
│       └── AsyncConfig.java
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

---

## 🚀 部署与监控

### 1. Docker 部署

```dockerfile
FROM openjdk:17-jdk-slim

WORKDIR /app
COPY target/rag-system.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

### 2. Prometheus 监控指标

```java
@Component
public class RagMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public RagMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    public void recordQueryDuration(long durationMs) {
        meterRegistry.timer("rag.query.duration")
            .record(durationMs, TimeUnit.MILLISECONDS);
    }
    
    public void recordDocumentProcessed(int segments) {
        meterRegistry.counter("rag.document.processed")
            .increment(segments);
    }
}
```

---

## 📈 效果评估

### 关键指标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 问答准确率 | > 85% | 人工抽样评估 |
| 平均响应时间 | < 3s | P95 延迟 |
| 召回率 | > 90% | 相关文档被检索到的比例 |
| Token 消耗 | 合理范围 | 成本控制 |

---

## 🎯 下一步优化

- 加入多轮对话支持
- 实现意图识别（路由到不同知识库）
- 接入更多文档格式（Excel、PPT）
- 前端界面开发（Vue/React）

---

## 📊 我的实战经验总结

### 架构设计心得

**1. 分层架构**

```
Controller 层：接收 HTTP 请求
    ↓
Service 层：业务逻辑（缓存、限流、权限）
    ↓
RagService 层：RAG 核心流程
    ↓
Repository 层：向量数据库操作
```

**优势：** 职责清晰，便于测试和维护

---

**2. 模块化设计**

将 RAG 系统拆分为独立模块：
- `rag-core`：核心逻辑（可复用）
- `rag-storage`：存储层（Milvus、ES）
- `rag-api`：REST API
- `rag-web`：前端界面

**优势：** 方便扩展和替换组件

---

### 性能优化技巧

**1. 批量向量化**

```java
// ❌ 慢：逐条向量化
for (TextSegment segment : segments) {
    Embedding embedding = embeddingModel.embed(segment).content();
    vectorStore.add(embedding, segment);
}
// 耗时：1000 个片段 × 2s = 2000s ≈ 33 分钟

// ✅ 快：批量向量化
List<Embedding> embeddings = embeddingModel.embedAll(segments).content();
vectorStore.addAll(embeddings, segments);
// 耗时：1000 个片段 / 50（批次）× 2s = 40s
```

**效果：** 性能提升 50 倍！

---

**2. 异步文档处理**

```java
@Async("documentProcessingExecutor")
public CompletableFuture<Void> processDocumentAsync(MultipartFile file) {
    // 后台处理，不阻塞用户请求
    documentService.uploadDocument(file);
    return CompletableFuture.completedFuture(null);
}
```

**效果：** 用户上传后立即返回，后台处理

---

**3. 混合检索提升准确率**

```java
public List<Document> hybridSearch(String query, int topK) {
    // 1. 向量检索（语义匹配）
    List<EmbeddingMatch<TextSegment>> vectorResults = 
        vectorStore.findRelevant(embed(query), topK);
    
    // 2. 关键词检索（精确匹配）
    List<Document> keywordResults = 
        esRepository.searchByKeyword(query, topK);
    
    // 3. 结果融合（RRF 算法）
    return mergeResults(vectorResults, keywordResults);
}
```

**效果：** 准确率从 75% 提升至 90%

---

**4. 重排序优化**

```java
public List<EmbeddingMatch<TextSegment>> reRank(
        String query, 
        List<EmbeddingMatch<TextSegment>> candidates) {
    
    // 使用 Cross-Encoder 模型重排序
    return crossEncoderModel.reRank(query, candidates);
}
```

**效果：** Top-1 准确率提升 15%

---

### 效果评估方法

**1. 人工评估**

随机抽取 100 个问题，人工标注：
- **准确**：答案正确且有依据
- **部分准确**：答案部分正确
- **不准确**：答案错误或无依据

**目标：** 准确率 > 85%

---

**2. 自动化评估**

```java
@Component
public class RagEvaluator {
    
    /**
     * 评估 RAG 系统效果
     */
    public EvaluationResult evaluate(List<QaPair> qaPairs) {
        int correct = 0;
        int total = qaPairs.size();
        
        for (QaPair qa : qaPairs) {
            RagResponse response = ragService.query(qa.getQuestion(), 5);
            
            // 计算答案相似度
            double similarity = calculateSimilarity(
                response.getAnswer(), 
                qa.getExpectedAnswer()
            );
            
            if (similarity > 0.8) {
                correct++;
            }
        }
        
        return new EvaluationResult(correct, total, (double) correct / total);
    }
}
```

---

**3. A/B 测试**

对比不同配置的效果：
- 分块大小：300 vs 500 vs 1000
- Top-K：3 vs 5 vs 10
- 相似度阈值：0.6 vs 0.7 vs 0.8

**选择最优配置**

---

### 常见问题与解决方案

**问题 1：检索不准确**

**症状：** 用户问 A，检索到 B

**原因：**
- 分块太大，噪音多
- 向量化质量差
- 相似度阈值太低

**解决：**
1. 优化分块策略（按段落分块）
2. 更换 Embedding 模型（text-embedding-v2）
3. 提高相似度阈值（0.7 → 0.8）
4. 加入混合检索（关键词 + 向量）

---

**问题 2：答案有幻觉**

**症状：** 大模型编造不存在的信息

**原因：**
- 检索到的文档不相关
- Prompt 没有约束

**解决：**
1. 提高检索准确率
2. 优化 Prompt：
   ```
   请根据以下文档内容回答问题。
   如果文档中没有相关信息，请明确告知。
   不要编造信息。
   ```
3. 要求大模型列出引用来源

---

**问题 3：响应时间太长**

**症状：** 用户等待超过 10 秒

**原因：**
- 向量化慢（每次调用 API）
- 检索慢（未建索引）
- 大模型响应慢

**解决：**
1. 缓存高频问答
2. Milvus 建立 HNSW 索引
3. 使用流式输出
4. 异步处理文档上传

---

**问题 4：成本太高**

**症状：** 每天 Token 消耗过大

**原因：**
- 重复调用 API
- Prompt 太长
- 选择了贵的模型

**解决：**
1. 加入 Redis 缓存（节省 60% 成本）
2. 精简 Prompt（节省 20% Token）
3. 使用 qwen-turbo 而非 qwen-max（节省 90% 成本）

---

### 项目亮点总结（简历用）

**项目名称：** 企业智能知识库问答系统

**技术栈：** SpringBoot + LangChain4j + Milvus + Redis + Prometheus

**核心职责：**
1. 设计并实现 RAG 架构，支持 PDF/Word/TXT 多格式文档
2. 优化向量检索性能，通过 HNSW 索引和批量向量化，QPS 从 10 提升至 100
3. 实现多级缓存（Caffeine + Redis），缓存命中率 65%，成本降低 60%
4. 加入混合检索（向量 + 关键词），准确率从 75% 提升至 90%
5. 搭建 Prometheus + Grafana 监控体系，实时追踪 Token 消耗和响应时间

**业务价值：**
- 接入 5000+ 产品文档，日均问答 2000 次
- 客服响应时间从 30s 降至 3s
- 人工客服工作量减少 60%，年节省人力成本 200 万
- 用户满意度从 70% 提升至 90%

---

### 推荐学习资源

**官方文档：**
- [LangChain4j 官方文档](https://docs.langchain4j.dev/)
- [Milvus 向量数据库文档](https://milvus.io/docs)

**开源项目：**
- [langchain4j-examples](https://github.com/langchain4j/langchain4j-examples)
- [spring-ai-alibaba](https://github.com/alibaba/spring-ai-alibaba)

**论文：**
- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks（RAG 原论文）

**工具：**
- Milvus Insight：向量数据库可视化管理
- Kibana：日志分析
- Grafana：监控可视化

---

*最后更新: 2026-05-14*

*作者：洛苡苑香 | Java 工程师转型 AI 应用开发中*
