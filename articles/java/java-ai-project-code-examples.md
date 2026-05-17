# 智能客服系统 - 完整代码示例

本文档提供《Java + AI 项目实战》文章中所有核心功能的完整代码实现。

---

## 📦 项目结构

```
intelligent-customer-service/
├── pom.xml
├── src/main/java/com/example/customer/
│   ├── CustomerServiceApplication.java
│   ├── config/
│   │   ├── LlmConfig.java
│   │   ├── MilvusConfig.java
│   │   ├── RedisConfig.java
│   │   └── AsyncConfig.java
│   ├── controller/
│   │   ├── ChatController.java
│   │   ├── DocumentController.java
│   │   └── AdminController.java
│   ├── service/
│   │   ├── IntelligentCustomerService.java
│   │   ├── IntentClassifier.java
│   │   ├── RagEngine.java
│   │   ├── ConversationManager.java
│   │   ├── CacheService.java
│   │   ├── DocumentService.java
│   │   └── QaLogService.java
│   ├── model/
│   │   ├── ChatResponse.java
│   │   ├── Message.java
│   │   ├── ConversationContext.java
│   │   ├── Intent.java
│   │   └── QaLog.java
│   ├── repository/
│   │   ├── ConversationRepository.java
│   │   ├── MessageRepository.java
│   │   └── QaLogRepository.java
│   └── util/
│       ├── PdfParser.java
│       └── TextSplitter.java
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       └── V1__init_schema.sql
└── docker-compose.yml
```

---

## 🔧 配置文件

### application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: intelligent-customer-service
  
  datasource:
    url: jdbc:mysql://localhost:3306/customer_service?useSSL=false&serverTimezone=UTC
    username: root
    password: root
  
  redis:
    host: localhost
    port: 6379
    timeout: 3000ms
  
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB

# LangChain4j 配置
langchain4j:
  dashscope:
    api-key: ${DASHSCOPE_API_KEY}
    chat-model: qwen-turbo
    embedding-model: text-embedding-v1
    temperature: 0.7

# Milvus 配置
milvus:
  host: localhost
  port: 19530
  collection-name: customer_service_kb

# 异步线程池配置
async:
  thread-pool:
    core-size: 10
    max-size: 20
    queue-capacity: 100

# 监控配置
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 💾 数据库初始化

### V1__init_schema.sql

```sql
-- 用户表
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    password VARCHAR(100) NOT NULL,
    role ENUM('customer', 'agent', 'admin') DEFAULT 'customer',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 会话表
CREATE TABLE conversations (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    status ENUM('active', 'closed', 'transferred') DEFAULT 'active',
    agent_id BIGINT NULL,
    started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    ended_at DATETIME NULL,
    FOREIGN KEY (user_id) REFERENCES users(id),
    INDEX idx_user_id (user_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 消息表
CREATE TABLE messages (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    conversation_id BIGINT NOT NULL,
    sender_type ENUM('user', 'bot', 'agent') NOT NULL,
    content TEXT NOT NULL,
    metadata JSON NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (conversation_id) REFERENCES conversations(id),
    INDEX idx_conversation_id (conversation_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 知识库表
CREATE TABLE knowledge_bases (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    category VARCHAR(50),
    created_by BIGINT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(id),
    INDEX idx_category (category)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 文档表
CREATE TABLE documents (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    kb_id BIGINT NOT NULL,
    filename VARCHAR(200) NOT NULL,
    file_path VARCHAR(500) NOT NULL,
    file_size INT,
    status ENUM('pending', 'processing', 'completed', 'failed') DEFAULT 'pending',
    segment_count INT DEFAULT 0,
    uploaded_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    processed_at DATETIME NULL,
    FOREIGN KEY (kb_id) REFERENCES knowledge_bases(id),
    INDEX idx_kb_id (kb_id),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 问答日志表
CREATE TABLE qa_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    question TEXT NOT NULL,
    answer TEXT,
    similarity_score DOUBLE,
    token_usage INT,
    duration_ms INT,
    from_cache BOOLEAN DEFAULT FALSE,
    feedback ENUM('good', 'bad') NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 🎯 核心代码实现

### 1. 智能客服服务

```java
package com.example.customer.service;

import com.example.customer.model.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
@Slf4j
public class IntelligentCustomerService {
    
    private final IntentClassifier intentClassifier;
    private final RagEngine ragEngine;
    private final CacheService cacheService;
    private final ConversationManager conversationManager;
    private final QaLogService qaLogService;
    
    public ChatResponse handleMessage(Long userId, String message, Long conversationId) {
        long startTime = System.currentTimeMillis();
        
        try {
            // 1. 检查缓存
            String cacheKey = generateCacheKey(message);
            ChatResponse cached = cacheService.getCachedAnswer(cacheKey);
            if (cached != null) {
                log.info("缓存命中: {}", message);
                cached.setFromCache(true);
                return cached;
            }
            
            // 2. 意图识别
            Intent intent = intentClassifier.classify(message);
            log.info("意图识别结果: {}", intent);
            
            // 3. 获取对话上下文
            ConversationContext context = conversationManager.getContext(conversationId);
            
            // 4. RAG 问答
            ChatResponse response = ragEngine.query(message, intent, context);
            
            // 5. 更新对话上下文
            conversationManager.updateContext(conversationId, message, response);
            
            // 6. 记录日志
            long duration = System.currentTimeMillis() - startTime;
            qaLogService.logQuery(userId, message, response, duration);
            
            // 7. 写入缓存
            cacheService.cacheAnswer(cacheKey, response);
            
            return response;
            
        } catch (Exception e) {
            log.error("处理消息失败", e);
            return ChatResponse.builder()
                .answer("抱歉，系统出现错误，请稍后重试。")
                .build();
        }
    }
    
    private String generateCacheKey(String message) {
        return "faq:" + md5(message.trim().toLowerCase());
    }
    
    private String md5(String input) {
        try {
            java.security.MessageDigest md = java.security.MessageDigest.getInstance("MD5");
            byte[] digest = md.digest(input.getBytes());
            return new java.math.BigInteger(1, digest).toString(16);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

### 2. 意图识别器

```java
package com.example.customer.service;

import com.example.customer.model.Intent;
import org.springframework.stereotype.Component;

import java.util.Arrays;
import java.util.List;
import java.util.Map;

@Component
public class IntentClassifier {
    
    private static final Map<String, List<String>> INTENT_KEYWORDS = Map.of(
        "LOGISTICS", Arrays.asList("物流", "快递", "配送", "发货", "运输", "签收"),
        "RETURN", Arrays.asList("退货", "换货", "退款", "退换", "返现"),
        "PRODUCT", Arrays.asList("产品", "商品", "规格", "价格", "优惠", "折扣")),
        "ACCOUNT", Arrays.asList("账号", "密码", "登录", "注册", "修改", "绑定"),
        "PROMOTION", Arrays.asList("活动", "促销", "优惠券", "积分", "会员")
    );
    
    public Intent classify(String message) {
        // 1. 关键词匹配
        for (Map.Entry<String, List<String>> entry : INTENT_KEYWORDS.entrySet()) {
            for (String keyword : entry.getValue()) {
                if (message.contains(keyword)) {
                    return new Intent(entry.getKey(), 0.8);
                }
            }
        }
        
        // 2. 默认意图
        return new Intent("GENERAL", 0.5);
    }
}
```

---

### 3. RAG 引擎

```java
package com.example.customer.service;

import com.example.customer.model.*;
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.dashscope.DashScopeChatModel;
import dev.langchain4j.model.dashscope.DashScopeEmbeddingModel;
import dev.langchain4j.store.embedding.EmbeddingMatch;
import dev.langchain4j.store.embedding.milvus.MilvusEmbeddingStore;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class RagEngine {
    
    private static final int TOP_K = 5;
    private static final double SIMILARITY_THRESHOLD = 0.7;
    
    private final MilvusEmbeddingStore embeddingStore;
    private final DashScopeEmbeddingModel embeddingModel;
    private final DashScopeChatModel chatModel;
    
    public ChatResponse query(String question, Intent intent, ConversationContext context) {
        try {
            // 1. 问题向量化
            Embedding questionEmbedding = embeddingModel.embed(question).content();
            
            // 2. 向量检索
            List<EmbeddingMatch<TextSegment>> matches = embeddingStore.findRelevant(
                questionEmbedding,
                TOP_K,
                SIMILARITY_THRESHOLD
            );
            
            log.info("检索到 {} 个相关文档", matches.size());
            
            if (matches.isEmpty()) {
                return ChatResponse.builder()
                    .answer("抱歉，没有找到相关的信息。您可以换个方式提问，或者转人工客服。")
                    .sources(List.of())
                    .build();
            }
            
            // 3. 组装上下文
            String context_text = buildContext(matches);
            
            // 4. 构建 Prompt（包含对话历史）
            String prompt = buildPrompt(question, context_text, context);
            
            // 5. 调用大模型
            String answer = chatModel.generate(prompt);
            
            // 6. 提取引用来源
            List<Source> sources = extractSources(matches);
            
            return ChatResponse.builder()
                .answer(answer)
                .sources(sources)
                .avgSimilarityScore(calculateAvgScore(matches))
                .build();
            
        } catch (Exception e) {
            log.error("RAG 查询失败", e);
            throw new RuntimeException("RAG 查询失败", e);
        }
    }
    
    private String buildContext(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> match.embedded().text())
            .collect(Collectors.joining("\n\n---\n\n"));
    }
    
    private String buildPrompt(String question, String context, ConversationContext history) {
        StringBuilder prompt = new StringBuilder();
        
        prompt.append("你是一个专业的电商客服助手，请根据以下文档内容回答问题。\n\n");
        
        // 添加对话历史
        if (history != null && !history.getHistory().isEmpty()) {
            prompt.append("对话历史：\n");
            prompt.append(history.getRecentHistory(3));
            prompt.append("\n\n");
        }
        
        prompt.append("文档内容：\n");
        prompt.append(context);
        prompt.append("\n\n");
        
        prompt.append("问题：").append(question).append("\n\n");
        
        prompt.append("要求：\n");
        prompt.append("1. 答案必须基于文档内容\n");
        prompt.append("2. 如果文档中没有相关信息，请明确告知\n");
        prompt.append("3. 回答要简洁明了，语气友好\n");
        prompt.append("4. 列出引用来源\n");
        
        return prompt.toString();
    }
    
    private List<Source> extractSources(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> Source.builder()
                .filename(match.embedded().metadata().get("filename"))
                .similarityScore(match.score())
                .text(match.embedded().text().substring(0, Math.min(100, match.embedded().text().length())))
                .build())
            .collect(Collectors.toList());
    }
    
    private double calculateAvgScore(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .mapToDouble(EmbeddingMatch::score)
            .average()
            .orElse(0.0);
    }
}
```

---

### 4. 多级缓存服务

```java
package com.example.customer.service;

import com.example.customer.model.ChatResponse;
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class CacheService {
    
    // L1: 本地缓存（Caffeine）
    private final Cache<String, ChatResponse> localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .recordStats()
        .build();
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final long REDIS_TTL = 2; // 2小时
    
    /**
     * 获取缓存的答案
     */
    public ChatResponse getCachedAnswer(String cacheKey) {
        // L1: 本地缓存
        ChatResponse local = localCache.getIfPresent(cacheKey);
        if (local != null) {
            log.debug("L1 缓存命中");
            return local;
        }
        
        // L2: Redis 缓存
        ChatResponse remote = (ChatResponse) redisTemplate.opsForValue().get(cacheKey);
        if (remote != null) {
            log.debug("L2 缓存命中");
            localCache.put(cacheKey, remote);  // 回写 L1
            return remote;
        }
        
        return null;
    }
    
    /**
     * 缓存答案
     */
    public void cacheAnswer(String cacheKey, ChatResponse response) {
        // 写入 L1
        localCache.put(cacheKey, response);
        
        // 写入 L2
        try {
            redisTemplate.opsForValue().set(cacheKey, response, REDIS_TTL, TimeUnit.HOURS);
        } catch (Exception e) {
            log.error("Redis 缓存写入失败", e);
        }
    }
    
    /**
     * 获取缓存统计
     */
    public String getCacheStats() {
        return String.format("L1命中率: %.2f%%, 大小: %d",
            localCache.stats().hitRate() * 100,
            localCache.estimatedSize());
    }
}
```

---

### 5. 对话管理器

```java
package com.example.customer.service;

import com.example.customer.model.ConversationContext;
import com.example.customer.model.Message;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class ConversationManager {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    private static final long CONTEXT_TTL = 30; // 30分钟
    private static final int MAX_HISTORY_SIZE = 20; // 最多保存10轮对话
    
    /**
     * 获取对话上下文
     */
    public ConversationContext getContext(Long conversationId) {
        String key = "conversation:" + conversationId;
        
        ConversationContext context = 
            (ConversationContext) redisTemplate.opsForValue().get(key);
        
        if (context == null) {
            context = new ConversationContext();
            context.setConversationId(conversationId);
            context.setHistory(new ArrayList<>());
        }
        
        return context;
    }
    
    /**
     * 更新对话上下文
     */
    public void updateContext(Long conversationId, String userMessage, com.example.customer.model.ChatResponse response) {
        String key = "conversation:" + conversationId;
        
        ConversationContext context = getContext(conversationId);
        
        // 添加用户消息
        context.addMessage(Message.builder()
            .senderType("user")
            .content(userMessage)
            .build());
        
        // 添加机器人回复
        context.addMessage(Message.builder()
            .senderType("bot")
            .content(response.getAnswer())
            .build());
        
        // 保持最近 N 条消息
        if (context.getHistory().size() > MAX_HISTORY_SIZE) {
            context.setHistory(context.getHistory().subList(
                context.getHistory().size() - MAX_HISTORY_SIZE,
                context.getHistory().size()
            ));
        }
        
        // 保存到 Redis
        redisTemplate.opsForValue().set(key, context, CONTEXT_TTL, TimeUnit.MINUTES);
    }
}
```

---

## 📊 监控指标

### Prometheus Metrics

```java
package com.example.customer.config;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.util.concurrent.TimeUnit;

@Component
@Slf4j
public class CustomerServiceMetrics {
    
    private final MeterRegistry meterRegistry;
    
    // 计数器
    private Counter totalRequests;
    private Counter cacheHits;
    private Counter errors;
    
    // 计时器
    private Timer requestDuration;
    
    public CustomerServiceMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @PostConstruct
    public void init() {
        totalRequests = Counter.builder("customer.request.total")
            .description("总请求数")
            .register(meterRegistry);
        
        cacheHits = Counter.builder("customer.cache.hits")
            .description("缓存命中数")
            .register(meterRegistry);
        
        errors = Counter.builder("customer.request.errors")
            .description("错误请求数")
            .register(meterRegistry);
        
        requestDuration = Timer.builder("customer.request.duration")
            .description("请求耗时")
            .register(meterRegistry);
    }
    
    public void recordRequest(long durationMs, boolean cacheHit, boolean error) {
        totalRequests.increment();
        
        if (cacheHit) {
            cacheHits.increment();
        }
        
        if (error) {
            errors.increment();
        }
        
        requestDuration.record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

---

## 🚀 Docker Compose 部署

```yaml
version: '3.8'

services:
  # 应用服务
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DASHSCOPE_API_KEY=${DASHSCOPE_API_KEY}
      - MYSQL_HOST=mysql
      - REDIS_HOST=redis
      - MILVUS_HOST=milvus
    depends_on:
      - mysql
      - redis
      - milvus
    restart: unless-stopped
  
  # MySQL
  mysql:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=customer_service
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - "3306:3306"
    restart: unless-stopped
  
  # Redis
  redis:
    image: redis:7.0
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped
  
  # Milvus
  milvus:
    image: milvusdb/milvus:v2.3.0
    environment:
      - ETCD_ENDPOINTS=etcd:2379
      - MINIO_ADDRESS=minio:9000
    volumes:
      - milvus-data:/var/lib/milvus
    ports:
      - "19530:19530"
    depends_on:
      - etcd
      - minio
    restart: unless-stopped
  
  etcd:
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
    volumes:
      - etcd-data:/etcd
  
  minio:
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    volumes:
      - minio-data:/minio_data
    command: minio server /minio_data
  
  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped
  
  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

volumes:
  mysql-data:
  redis-data:
  milvus-data:
  etcd-data:
  minio-data:
  grafana-data:
```

---

*最后更新: 2026-05-14*
