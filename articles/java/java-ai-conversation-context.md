# 大模型对话上下文管理实战：多轮对话、记忆隔离与性能优化

> **写在前面：** 单轮对话只是聊天，多轮对话才是真正的智能助手。如何实现用户说"帮我查一下订单"，接着说"那个昨天的"，AI 能理解指的是"昨天的订单"？这需要完善的上下文管理机制。本文详细讲解如何设计会话ID、存储对话历史、管理上下文窗口、实现多用户隔离，打造生产级多轮对话系统！

---

## 📋 目录

- [一、为什么需要上下文管理](#一为什么需要上下文管理)
- [二、上下文管理核心挑战](#二上下文管理核心挑战)
- [三、会话ID设计与数据模型](#三会话id设计与数据模型)
- [四、对话历史存储方案对比](#四对话历史存储方案对比)
- [五、上下文拼接与窗口管理](#五上下文拼接与窗口管理)
- [六、多用户会话隔离实现](#六多用户会话隔离实现)
- [七、会话生命周期管理](#七会话生命周期管理)
- [八、性能优化：缓存与冷热分离](#八性能优化缓存与冷热分离)
- [九、完整实战案例](#九完整实战案例)
- [十、常见问题与解决方案](#十常见问题与解决方案)

---

## 一、为什么需要上下文管理

### 1.1 单轮对话 vs 多轮对话

**❌ 单轮对话（无上下文）：**

```
用户：北京天气怎么样？
AI：北京今天晴天，气温 25°C

用户：那上海呢？
AI：[无法理解"那上海呢"指的是什么]
   请明确您的问题
```

**✅ 多轮对话（有上下文）：**

```
用户：北京天气怎么样？
AI：北京今天晴天，气温 25°C

用户：那上海呢？
AI：[理解用户想查询上海天气]
   上海今天多云，气温 28°C

用户：哪个更热？
AI：[基于前两轮对话]
   上海（28°C）比北京（25°C）更热
```

---

### 1.2 上下文管理的价值

| 维度 | 无上下文 | 有上下文 |
|------|---------|---------|
| **用户体验** | ❌ 需重复说明 | ✅ 自然流畅 |
| **意图理解** | ❌ 每轮独立 | ✅ 关联推理 |
| **业务场景** | 有限 | 丰富（客服、顾问等） |
| **技术复杂度** | 低 | 中高 |

---

### 1.3 典型应用场景

**场景 1：智能客服**
```
用户：我的订单还没收到
AI：请问订单号是多少？
用户：#12345
AI：[查询订单] 您的订单已发货...
用户：什么时候能到？
AI：[基于订单信息] 预计明天送达
```

**场景 2：技术咨询**
```
用户：Spring Boot 怎么连接数据库？
AI：使用 JDBC 或 JPA...
用户：那事务怎么处理？
AI：[理解在讨论 Spring Boot] 使用 @Transactional...
用户：分布式事务呢？
AI：[延续话题] 可以使用 Seata...
```

**场景 3：代码助手**
```
用户：帮我写个用户登录接口
AI：[生成代码]
用户：加上 JWT 认证
AI：[在之前代码基础上添加 JWT]
用户：再加个限流
AI：[继续增强代码]
```

---

## 二、上下文管理核心挑战

### 2.1 四大核心挑战

#### 挑战 1：上下文窗口限制

**问题：** 大模型有最大 Token 限制（如 4096、8192、32768）

```
假设：
- 模型上下文窗口：4096 tokens
- 平均每轮对话：200 tokens
- 最多支持轮数：4096 / 200 ≈ 20 轮

超过 20 轮后，必须截断或总结前面的对话！
```

---

#### 挑战 2：多用户隔离

**问题：** 如何确保用户 A 看不到用户 B 的对话历史？

```
错误设计：
- 所有对话存在一个列表
- 没有用户标识
- 用户 A 可能看到用户 B 的历史

正确设计：
- 每个用户独立的会话
- 严格的权限隔离
- 会话 ID + 用户 ID 双重校验
```

---

#### 挑战 3：性能问题

**问题：** 频繁读写数据库导致响应慢

```
场景：
- 每轮对话需要：
  1. 读取历史（数据库查询）
  2. 拼接上下文
  3. 调用大模型
  4. 保存新对话（数据库写入）

如果每次都要查数据库，响应时间会增加 100-200ms
```

---

#### 挑战 4：数据清理

**问题：** 对话历史无限增长，占用存储空间

```
假设：
- 日活用户：10,000
- 平均每用户每天：20 轮对话
- 每轮对话：500 bytes
- 每日新增数据：10,000 × 20 × 500 = 100 MB
-  yearly 数据量：36.5 GB

需要定期清理过期数据！
```

---

## 三、会话ID设计与数据模型

### 3.1 会话 ID 生成策略

#### 方案 1：UUID（推荐）

```java
String sessionId = UUID.randomUUID().toString();
// 示例：550e8400-e29b-41d4-a716-446655440000
```

**优点：**
- ✅ 全局唯一
- ✅ 无需协调
- ✅ 安全性高（不可预测）

**缺点：**
- ❌ 长度较长（36 字符）

---

#### 方案 2：雪花算法

```java
long sessionId = snowflakeIdGenerator.nextId();
// 示例：1234567890123456789
```

**优点：**
- ✅ 长度短（Long 类型）
- ✅ 有序（便于索引）
- ✅ 高性能

**缺点：**
- ❌ 需要部署 ID 生成器
- ❌ 时钟回拨问题

---

#### 方案 3：业务规则生成

```java
String sessionId = "SESSION_" + userId + "_" + timestamp;
// 示例：SESSION_1001_1716000000000
```

**优点：**
- ✅ 可读性强
- ✅ 包含业务信息

**缺点：**
- ❌ 可能冲突
- ❌ 暴露业务信息

---

**💡 推荐方案：** UUID（简单可靠）

---

### 3.2 数据表设计

#### 方案 1：单表设计（简单场景）

```sql
CREATE TABLE `conversation_messages` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  `session_id` VARCHAR(64) NOT NULL COMMENT '会话ID',
  `user_id` VARCHAR(64) NOT NULL COMMENT '用户ID',
  `role` VARCHAR(10) NOT NULL COMMENT '角色：user/assistant/system',
  `content` TEXT NOT NULL COMMENT '消息内容',
  `tokens` INT DEFAULT NULL COMMENT 'Token 数量',
  `metadata` JSON DEFAULT NULL COMMENT '元数据（扩展字段）',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  
  PRIMARY KEY (`id`),
  KEY `idx_session_id` (`session_id`) COMMENT '会话ID索引',
  KEY `idx_user_id` (`user_id`) COMMENT '用户ID索引',
  KEY `idx_create_time` (`create_time`) COMMENT '时间索引'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对话消息表';
```

**优点：**
- ✅ 结构简单
- ✅ 易于查询

**缺点：**
- ❌ 单表数据量大（千万级后性能下降）

---

#### 方案 2：会话表 + 消息表（推荐）

```sql
-- 会话表
CREATE TABLE `conversations` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  `session_id` VARCHAR(64) NOT NULL COMMENT '会话ID',
  `user_id` VARCHAR(64) NOT NULL COMMENT '用户ID',
  `title` VARCHAR(255) DEFAULT NULL COMMENT '会话标题',
  `message_count` INT DEFAULT 0 COMMENT '消息数量',
  `last_message_time` DATETIME DEFAULT NULL COMMENT '最后消息时间',
  `status` TINYINT DEFAULT 1 COMMENT '状态：1-活跃，2-关闭',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_session_id` (`session_id`) COMMENT '会话ID唯一索引',
  KEY `idx_user_id` (`user_id`) COMMENT '用户ID索引',
  KEY `idx_last_message_time` (`last_message_time`) COMMENT '最后消息时间索引'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='会话表';

-- 消息表
CREATE TABLE `conversation_messages` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '主键',
  `session_id` VARCHAR(64) NOT NULL COMMENT '会话ID',
  `role` VARCHAR(10) NOT NULL COMMENT '角色：user/assistant/system',
  `content` TEXT NOT NULL COMMENT '消息内容',
  `tokens` INT DEFAULT NULL COMMENT 'Token 数量',
  `metadata` JSON DEFAULT NULL COMMENT '元数据',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  
  PRIMARY KEY (`id`),
  KEY `idx_session_id` (`session_id`) COMMENT '会话ID索引',
  KEY `idx_create_time` (`create_time`) COMMENT '时间索引'
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='对话消息表';
```

**优点：**
- ✅ 会话级别操作高效（查询会话列表、删除会话）
- ✅ 消息表可按会话分片
- ✅ 便于统计（会话数、消息数）

---

### 3.3 Entity 实体类

```java
package com.example.conversation.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;

import java.time.LocalDateTime;

/**
 * 会话实体
 */
@Data
@TableName("conversations")
public class Conversation {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String sessionId;

    private String userId;

    private String title;

    private Integer messageCount;

    private LocalDateTime lastMessageTime;

    private Integer status;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}
```

```java
package com.example.conversation.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;

import java.time.LocalDateTime;

/**
 * 对话消息实体
 */
@Data
@TableName("conversation_messages")
public class ConversationMessage {

    @TableId(type = IdType.AUTO)
    private Long id;

    private String sessionId;

    private String role; // user / assistant / system

    private String content;

    private Integer tokens;

    @TableField(typeHandler = JacksonTypeHandler.class)
    private Map<String, Object> metadata;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
}
```

---

## 四、对话历史存储方案对比

### 4.1 方案对比表

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|---------|
| **MySQL** | 成熟稳定、易维护 | 性能一般、成本高 | 中小规模（< 1000 万） |
| **Redis** | 速度快、支持 TTL | 内存成本高、持久化弱 | 高频访问、短期存储 |
| **MongoDB** | 灵活 schema、水平扩展 | 运维复杂 | 大规模、非结构化数据 |
| **混合方案** | 兼顾性能和成本 | 架构复杂 | **生产环境推荐** ✅ |

---

### 4.2 混合存储方案（推荐）

**架构设计：**

```
┌─────────────┐
│  用户请求    │
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌──────────────┐
│  Redis 缓存  │◄────►│  MySQL 存储   │
│  (热数据)    │      │  (冷数据)     │
│  TTL: 24h   │      │  永久存储     │
└─────────────┘      └──────────────┘
```

**工作流程：**

1. **写入：**
   - 同时写入 Redis 和 MySQL
   - Redis 设置 TTL（24 小时）

2. **读取：**
   - 先查 Redis（命中率高）
   - Redis 未命中再查 MySQL
   - 从 MySQL 读取后回填 Redis

3. **清理：**
   - Redis 自动过期
   - MySQL 定时清理（保留 30 天）

---

### 4.3 Redis 缓存实现

```java
package com.example.conversation.cache;

import com.example.conversation.entity.ConversationMessage;
import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Component
@RequiredArgsConstructor
public class ConversationCache {

    private final RedisTemplate<String, Object> redisTemplate;

    private static final String KEY_PREFIX = "conversation:";
    private static final long TTL_HOURS = 24;

    /**
     * 缓存会话消息列表
     */
    public void cacheMessages(String sessionId, List<ConversationMessage> messages) {
        String key = KEY_PREFIX + sessionId;
        redisTemplate.opsForValue().set(key, messages, TTL_HOURS, TimeUnit.HOURS);
    }

    /**
     * 获取缓存的会话消息
     */
    @SuppressWarnings("unchecked")
    public List<ConversationMessage> getCachedMessages(String sessionId) {
        String key = KEY_PREFIX + sessionId;
        return (List<ConversationMessage>) redisTemplate.opsForValue().get(key);
    }

    /**
     * 追加单条消息到缓存
     */
    public void appendMessage(String sessionId, ConversationMessage message) {
        String key = KEY_PREFIX + sessionId;
        
        List<ConversationMessage> messages = getCachedMessages(sessionId);
        if (messages != null) {
            messages.add(message);
            redisTemplate.opsForValue().set(key, messages, TTL_HOURS, TimeUnit.HOURS);
        }
    }

    /**
     * 删除会话缓存
     */
    public void evictCache(String sessionId) {
        String key = KEY_PREFIX + sessionId;
        redisTemplate.delete(key);
    }
}
```

---

## 五、上下文拼接与窗口管理

### 5.1 上下文拼接策略

#### 策略 1：全量拼接（简单场景）

```java
public String buildFullContext(List<ConversationMessage> messages) {
    StringBuilder context = new StringBuilder();
    
    for (ConversationMessage msg : messages) {
        String role = msg.getRole().equals("user") ? "用户" : "助手";
        context.append(role).append("：").append(msg.getContent()).append("\n");
    }
    
    return context.toString();
}
```

**优点：** 简单直接  
**缺点：** 容易超出 Token 限制

---

#### 策略 2：滑动窗口（推荐）

```java
public List<ConversationMessage> buildSlidingWindow(
        List<ConversationMessage> messages, 
        int maxTokens) {
    
    List<ConversationMessage> window = new ArrayList<>();
    int currentTokens = 0;
    
    // 从最新消息向前遍历
    for (int i = messages.size() - 1; i >= 0; i--) {
        ConversationMessage msg = messages.get(i);
        int msgTokens = estimateTokens(msg.getContent());
        
        if (currentTokens + msgTokens > maxTokens) {
            break; // 超出限制，停止
        }
        
        window.add(0, msg); // 插入到列表头部
        currentTokens += msgTokens;
    }
    
    return window;
}

private int estimateTokens(String text) {
    // 粗略估算：中文 1.5 字符/token，英文 4 字符/token
    return text.length() / 3;
}
```

**优点：** 保证不超限  
**缺点：** 可能丢失早期重要信息

---

#### 策略 3：关键信息提取 + 滑动窗口（高级）

```java
public String buildSmartContext(List<ConversationMessage> messages, int maxTokens) {
    // 1. 提取关键信息（首条系统消息 + 最近 N 轮）
    List<ConversationMessage> recentMessages = buildSlidingWindow(messages, maxTokens);
    
    // 2. 如果有更早的消息，生成摘要
    if (messages.size() > recentMessages.size()) {
        String summary = generateSummary(messages.subList(0, messages.size() - recentMessages.size()));
        recentMessages.add(0, new ConversationMessage("system", summary));
    }
    
    // 3. 拼接上下文
    return concatenateMessages(recentMessages);
}

private String generateSummary(List<ConversationMessage> oldMessages) {
    // 调用大模型生成摘要
    String oldContext = concatenateMessages(oldMessages);
    String prompt = "请总结以下对话的关键信息（100 字以内）：\n" + oldContext;
    
    return chatModel.generate(prompt);
}
```

**优点：** 保留关键信息，充分利用上下文  
**缺点：** 需要额外调用大模型

---

### 5.2 Token 计算工具

```java
package com.example.conversation.util;

import com.knuddels.jtokkit.Encodings;
import com.knuddels.jtokkit.api.Encoding;
import com.knuddels.jtokkit.api.EncodingRegistry;
import com.knuddels.jtokkit.api.ModelType;

/**
 * Token 计算工具
 */
public class TokenCounter {

    private static final EncodingRegistry registry = Encodings.newDefaultEncodingRegistry();
    private static final Encoding encoding = registry.getEncodingForModel(ModelType.GPT_3_5_TURBO);

    /**
     * 计算文本的 Token 数量
     */
    public static int countTokens(String text) {
        return encoding.countTokens(text);
    }

    /**
     * 计算消息列表的总 Token 数
     */
    public static int countTotalTokens(List<ConversationMessage> messages) {
        return messages.stream()
            .mapToInt(msg -> countTokens(msg.getContent()))
            .sum();
    }
}
```

**Maven 依赖：**

```xml
<dependency>
    <groupId>com.knuddels</groupId>
    <artifactId>jtokkit</artifactId>
    <version>0.6.1</version>
</dependency>
```

---

## 六、多用户会话隔离实现

### 6.1 会话服务实现

```java
package com.example.conversation.service;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.example.conversation.cache.ConversationCache;
import com.example.conversation.entity.Conversation;
import com.example.conversation.entity.ConversationMessage;
import com.example.conversation.mapper.ConversationMapper;
import com.example.conversation.mapper.ConversationMessageMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ConversationService {

    private final ConversationMapper conversationMapper;
    private final ConversationMessageMapper messageMapper;
    private final ConversationCache conversationCache;

    /**
     * 创建新会话
     */
    @Transactional
    public String createSession(String userId) {
        String sessionId = UUID.randomUUID().toString();
        
        Conversation conversation = new Conversation();
        conversation.setSessionId(sessionId);
        conversation.setUserId(userId);
        conversation.setTitle("新会话");
        conversation.setMessageCount(0);
        conversation.setStatus(1);
        conversation.setLastMessageTime(LocalDateTime.now());
        
        conversationMapper.insert(conversation);
        
        log.info("创建会话成功: sessionId={}, userId={}", sessionId, userId);
        
        return sessionId;
    }

    /**
     * 添加消息
     */
    @Transactional
    public void addMessage(String sessionId, String userId, String role, String content) {
        // 1. 权限校验：确保用户只能操作自己的会话
        validateSessionOwnership(sessionId, userId);
        
        // 2. 保存消息到数据库
        ConversationMessage message = new ConversationMessage();
        message.setSessionId(sessionId);
        message.setRole(role);
        message.setContent(content);
        message.setTokens(TokenCounter.countTokens(content));
        message.setCreateTime(LocalDateTime.now());
        
        messageMapper.insert(message);
        
        // 3. 更新会话信息
        updateSessionInfo(sessionId);
        
        // 4. 更新缓存
        conversationCache.evictCache(sessionId); // 清除旧缓存
        
        log.debug("添加消息成功: sessionId={}, role={}", sessionId, role);
    }

    /**
     * 获取会话历史
     */
    public List<ConversationMessage> getSessionHistory(String sessionId, String userId) {
        // 1. 权限校验
        validateSessionOwnership(sessionId, userId);
        
        // 2. 先查缓存
        List<ConversationMessage> cached = conversationCache.getCachedMessages(sessionId);
        if (cached != null) {
            log.debug("缓存命中: sessionId={}", sessionId);
            return cached;
        }
        
        // 3. 查数据库
        List<ConversationMessage> messages = messageMapper.selectList(
            new LambdaQueryWrapper<ConversationMessage>()
                .eq(ConversationMessage::getSessionId, sessionId)
                .orderByAsc(ConversationMessage::getCreateTime)
        );
        
        // 4. 写入缓存
        if (!messages.isEmpty()) {
            conversationCache.cacheMessages(sessionId, messages);
        }
        
        return messages;
    }

    /**
     * 获取用户的会话列表
     */
    public List<Conversation> getUserSessions(String userId, int pageNum, int pageSize) {
        return conversationMapper.selectPage(
            new Page<>(pageNum, pageSize),
            new LambdaQueryWrapper<Conversation>()
                .eq(Conversation::getUserId, userId)
                .eq(Conversation::getStatus, 1)
                .orderByDesc(Conversation::getLastMessageTime)
        ).getRecords();
    }

    /**
     * 删除会话
     */
    @Transactional
    public void deleteSession(String sessionId, String userId) {
        // 1. 权限校验
        validateSessionOwnership(sessionId, userId);
        
        // 2. 删除消息
        messageMapper.delete(
            new LambdaQueryWrapper<ConversationMessage>()
                .eq(ConversationMessage::getSessionId, sessionId)
        );
        
        // 3. 删除会话
        conversationMapper.delete(
            new LambdaQueryWrapper<Conversation>()
                .eq(Conversation::getSessionId, sessionId)
        );
        
        // 4. 清除缓存
        conversationCache.evictCache(sessionId);
        
        log.info("删除会话成功: sessionId={}", sessionId);
    }

    /**
     * 校验会话所有权
     */
    private void validateSessionOwnership(String sessionId, String userId) {
        Conversation conversation = conversationMapper.selectOne(
            new LambdaQueryWrapper<Conversation>()
                .eq(Conversation::getSessionId, sessionId)
        );
        
        if (conversation == null) {
            throw new BusinessException("会话不存在");
        }
        
        if (!conversation.getUserId().equals(userId)) {
            throw new BusinessException("无权访问此会话");
        }
    }

    /**
     * 更新会话信息
     */
    private void updateSessionInfo(String sessionId) {
        Conversation conversation = conversationMapper.selectOne(
            new LambdaQueryWrapper<Conversation>()
                .eq(Conversation::getSessionId, sessionId)
        );
        
        if (conversation != null) {
            conversation.setMessageCount(conversation.getMessageCount() + 1);
            conversation.setLastMessageTime(LocalDateTime.now());
            conversationMapper.updateById(conversation);
        }
    }
}
```

---

### 6.2 Controller 接口

```java
package com.example.conversation.controller;

import com.example.common.result.Result;
import com.example.conversation.dto.CreateSessionRequest;
import com.example.conversation.dto.SendMessageRequest;
import com.example.conversation.entity.Conversation;
import com.example.conversation.entity.ConversationMessage;
import com.example.conversation.service.ConversationService;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/conversations")
@RequiredArgsConstructor
public class ConversationController {

    private final ConversationService conversationService;

    /**
     * 创建会话
     */
    @PostMapping
    public Result<String> createSession(@RequestBody CreateSessionRequest request) {
        String sessionId = conversationService.createSession(request.getUserId());
        return Result.success(sessionId);
    }

    /**
     * 发送消息
     */
    @PostMapping("/{sessionId}/messages")
    public Result<Void> sendMessage(
            @PathVariable String sessionId,
            @RequestBody SendMessageRequest request) {
        
        conversationService.addMessage(
            sessionId,
            request.getUserId(),
            request.getRole(),
            request.getContent()
        );
        
        return Result.success(null);
    }

    /**
     * 获取会话历史
     */
    @GetMapping("/{sessionId}/messages")
    public Result<List<ConversationMessage>> getSessionHistory(
            @PathVariable String sessionId,
            @RequestParam String userId) {
        
        List<ConversationMessage> messages = conversationService.getSessionHistory(sessionId, userId);
        return Result.success(messages);
    }

    /**
     * 获取用户的会话列表
     */
    @GetMapping
    public Result<List<Conversation>> getUserSessions(
            @RequestParam String userId,
            @RequestParam(defaultValue = "1") int pageNum,
            @RequestParam(defaultValue = "20") int pageSize) {
        
        List<Conversation> sessions = conversationService.getUserSessions(userId, pageNum, pageSize);
        return Result.success(sessions);
    }

    /**
     * 删除会话
     */
    @DeleteMapping("/{sessionId}")
    public Result<Void> deleteSession(
            @PathVariable String sessionId,
            @RequestParam String userId) {
        
        conversationService.deleteSession(sessionId, userId);
        return Result.success(null);
    }
}
```

---

## 七、会话生命周期管理

### 7.1 会话状态机

```
新建 → 活跃 → 闲置 → 关闭
         ↑      |
         └──────┘ (用户继续对话)
```

**状态定义：**

| 状态 | 说明 | 触发条件 |
|------|------|---------|
| **新建** | 刚创建的会话 | 用户首次发消息 |
| **活跃** | 正在使用的会话 | 最近 1 小时内有消息 |
| **闲置** | 长时间未使用 | 超过 1 小时无消息 |
| **关闭** | 已关闭的会话 | 用户手动关闭或超时 |

---

### 7.2 自动清理定时任务

```java
package com.example.conversation.task;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.example.conversation.entity.Conversation;
import com.example.conversation.mapper.ConversationMapper;
import com.example.conversation.mapper.ConversationMessageMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;

@Slf4j
@Component
@RequiredArgsConstructor
public class ConversationCleanupTask {

    private final ConversationMapper conversationMapper;
    private final ConversationMessageMapper messageMapper;

    /**
     * 每天凌晨 2 点清理过期会话
     */
    @Scheduled(cron = "0 0 2 * * ?")
    public void cleanupExpiredSessions() {
        log.info("开始清理过期会话...");
        
        // 清理 30 天前的会话
        LocalDateTime cutoffTime = LocalDateTime.now().minusDays(30);
        
        // 1. 查询过期会话
        List<Conversation> expiredSessions = conversationMapper.selectList(
            new LambdaQueryWrapper<Conversation>()
                .lt(Conversation::getLastMessageTime, cutoffTime)
        );
        
        log.info("找到 {} 个过期会话", expiredSessions.size());
        
        // 2. 删除消息
        for (Conversation session : expiredSessions) {
            messageMapper.delete(
                new LambdaQueryWrapper<ConversationMessage>()
                    .eq(ConversationMessage::getSessionId, session.getSessionId())
            );
        }
        
        // 3. 删除会话
        conversationMapper.delete(
            new LambdaQueryWrapper<Conversation>()
                .lt(Conversation::getLastMessageTime, cutoffTime)
        );
        
        log.info("清理完成，删除 {} 个会话", expiredSessions.size());
    }

    /**
     * 每小时标记闲置会话
     */
    @Scheduled(cron = "0 0 * * * ?")
    public void markIdleSessions() {
        log.info("开始标记闲置会话...");
        
        LocalDateTime idleThreshold = LocalDateTime.now().minusHours(1);
        
        // 更新最后消息时间在 1-24 小时之间的会话为闲置状态
        // （实际项目中可以添加 status 字段）
        
        log.info("闲置会话标记完成");
    }
}
```

**启用定时任务：**

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## 八、性能优化：缓存与冷热分离

### 8.1 多级缓存架构

```
用户请求
   │
   ▼
┌─────────────┐
│ L1: 本地缓存 │ (Caffeine, 10 分钟)
└──────┬──────┘
       │ 未命中
       ▼
┌─────────────┐
│ L2: Redis    │ (24 小时)
└──────┬──────┘
       │ 未命中
       ▼
┌─────────────┐
│ L3: MySQL    │ (永久存储)
└─────────────┘
```

---

### 8.2 Caffeine 本地缓存

```java
package com.example.conversation.cache;

import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.concurrent.TimeUnit;

@Component
public class LocalConversationCache {

    private final Cache<String, List<ConversationMessage>> cache;

    public LocalConversationCache() {
        this.cache = Caffeine.newBuilder()
            .maximumSize(1000)  // 最多缓存 1000 个会话
            .expireAfterWrite(10, TimeUnit.MINUTES)  // 10 分钟过期
            .build();
    }

    public void put(String sessionId, List<ConversationMessage> messages) {
        cache.put(sessionId, messages);
    }

    public List<ConversationMessage> get(String sessionId) {
        return cache.getIfPresent(sessionId);
    }

    public void invalidate(String sessionId) {
        cache.invalidate(sessionId);
    }
}
```

---

### 8.3 冷热数据分离

**热数据：** 最近 24 小时的会话 → Redis  
**冷数据：** 24 小时以上的会话 → MySQL

```java
public List<ConversationMessage> getSessionHistoryWithColdHotSeparation(
        String sessionId, String userId) {
    
    // 1. 查本地缓存
    List<ConversationMessage> local = localCache.get(sessionId);
    if (local != null) {
        return local;
    }
    
    // 2. 查 Redis（热数据）
    List<ConversationMessage> redis = redisCache.getCachedMessages(sessionId);
    if (redis != null) {
        localCache.put(sessionId, redis);
        return redis;
    }
    
    // 3. 查 MySQL（冷数据）
    List<ConversationMessage> mysql = messageMapper.selectList(...);
    
    // 4. 判断是否为热数据（最近 24 小时）
    if (isHotData(mysql)) {
        redisCache.cacheMessages(sessionId, mysql);
    }
    
    localCache.put(sessionId, mysql);
    
    return mysql;
}

private boolean isHotData(List<ConversationMessage> messages) {
    if (messages.isEmpty()) {
        return false;
    }
    
    LocalDateTime lastMessageTime = messages.get(messages.size() - 1).getCreateTime();
    return lastMessageTime.isAfter(LocalDateTime.now().minusHours(24));
}
```

---

## 九、完整实战案例

### 9.1 智能客服多轮对话

**场景：** 用户咨询订单问题

```
第 1 轮：
用户：我的订单还没收到
AI：请问订单号是多少？

第 2 轮：
用户：#12345
AI：[调用 getOrder 函数]
   您的订单 #12345 已发货，物流单号 SF123

第 3 轮：
用户：什么时候能到？
AI：[基于订单信息]
   预计明天送达

第 4 轮：
用户：能改地址吗？
AI：[理解用户在问订单 #12345]
   订单已发货，无法修改地址。您可以联系快递员...
```

**实现要点：**

1. **保持会话 ID：** 整个对话过程使用同一个 sessionId
2. **上下文拼接：** 每轮对话都携带前几轮历史
3. **意图识别：** AI 理解"能改地址吗"指的是订单 #12345

---

### 9.2 代码助手连续优化

**场景：** 用户逐步完善代码

```
第 1 轮：
用户：帮我写个用户登录接口
AI：[生成基础代码]

第 2 轮：
用户：加上 JWT 认证
AI：[在之前代码基础上添加 JWT]

第 3 轮：
用户：再加个限流
AI：[继续增强代码]

第 4 轮：
用户：性能怎么样？
AI：[基于完整代码分析性能]
```

**实现要点：**

1. **代码累积：** 每轮都在之前的代码基础上修改
2. **上下文窗口管理：** 如果代码太长，使用摘要策略
3. **版本管理：** 保存每轮代码版本，支持回滚

---

## 十、常见问题与解决方案

### 问题 1：上下文超出 Token 限制

**症状：**
```
Error: Context length exceeded (4096 tokens)
```

**解决方案：**

1. **滑动窗口截断**
```java
List<ConversationMessage> window = buildSlidingWindow(messages, 3500);
```

2. **生成摘要**
```java
String summary = generateSummary(oldMessages);
```

3. **升级模型**
```
qwen-turbo (4096) → qwen-plus (32768) → qwen-max (200000)
```

---

### 问题 2：多用户数据泄露

**症状：**
```
用户 A 看到了用户 B 的对话历史
```

**解决方案：**

1. **严格的权限校验**
```java
validateSessionOwnership(sessionId, userId);
```

2. **数据库层面隔离**
```sql
WHERE session_id = ? AND user_id = ?
```

3. **Redis Key 包含用户 ID**
```java
String key = "conversation:" + userId + ":" + sessionId;
```

---

### 问题 3：响应速度慢

**症状：**
```
每轮对话耗时 > 2 秒
```

**解决方案：**

1. **增加缓存层级**
```
本地缓存 (Caffeine) → Redis → MySQL
```

2. **异步写入**
```java
@Async
public void saveMessageAsync(ConversationMessage message) {
    messageMapper.insert(message);
}
```

3. **批量查询优化**
```java
// 一次性查询会话信息和消息
SELECT c.*, m.* 
FROM conversations c
LEFT JOIN conversation_messages m ON c.session_id = m.session_id
WHERE c.session_id = ?
```

---

### 问题 4：存储空间爆炸

**症状：**
```
数据库每月增长 50GB
```

**解决方案：**

1. **定期清理**
```java
@Scheduled(cron = "0 0 2 * * ?")
public void cleanupExpiredSessions() {
    // 删除 30 天前的数据
}
```

2. **数据压缩**
```java
// 对消息内容进行压缩
String compressed = compress(message.getContent());
```

3. **归档冷数据**
```
热数据（7 天）→ MySQL
冷数据（7-90 天）→ OSS/冷存储
超冷数据（90 天+）→ 删除
```

---

## 💡 总结

### 核心要点回顾

1. **会话 ID 设计：** 推荐使用 UUID，简单可靠
2. **存储方案：** 混合方案（Redis + MySQL）最佳
3. **上下文管理：** 滑动窗口 + 摘要生成
4. **多用户隔离：** 严格的权限校验
5. **性能优化：** 多级缓存 + 冷热分离
6. **生命周期：** 定时清理 + 状态管理

### 下一步行动

1. 实现基础的会话管理功能
2. 添加 Redis 缓存提升性能
3. 实施定时清理任务
4. 监控和优化响应时间

---

*最后更新: 2026-05-17*

*作者：洛苡苑香 | Java 工程师转型 AI 应用开发中*
