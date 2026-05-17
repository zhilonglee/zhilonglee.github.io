# Ollama 本地部署开源大模型完全指南：Java 后端工程师实战

> **写在前面：** 为什么需要本地部署大模型？数据隐私、成本控制、离线可用、定制化需求……本文详细记录如何在 Linux 服务器上使用 Ollama 部署 Qwen2、Llama3 等开源大模型，并封装为 OpenAI 兼容接口，让 Java 应用无缝调用。从零开始，手把手教你搭建企业级私有大模型服务！

---

## 📋 目录

- [一、为什么需要本地部署大模型](#一为什么需要本地部署大模型)
- [二、主流开源大模型对比选型](#二主流开源大模型对比选型)
- [三、Ollama 安装与配置](#三ollama-安装与配置)
- [四、下载与运行大模型](#四下载与运行大模型)
- [五、量化压缩技术详解](#五量化压缩技术详解)
- [六、OpenAI 格式适配层开发](#六openai-格式适配层开发)
- [七、Java 应用集成实战](#七java-应用集成实战)
- [八、性能优化与监控](#八性能优化与监控)
- [九、生产环境部署方案](#九生产环境部署方案)
- [十、常见问题与解决方案](#十常见问题与解决方案)

---

## 一、为什么需要本地部署大模型

### 1.1 云端 API vs 本地部署对比

| 维度 | 云端 API | 本地部署 |
|------|---------|---------|
| **数据隐私** | ❌ 数据传到第三方 | ✅ 数据完全可控 |
| **成本** | ❌ 按 Token 计费，长期使用成本高 | ✅ 一次性硬件投入 |
| **稳定性** | ❌ 依赖网络和服务商 | ✅ 内网稳定可用 |
| **延迟** | ❌ 网络延迟 100-500ms | ✅ 局域网延迟 < 10ms |
| **定制化** | ❌ 无法微调模型 | ✅ 支持微调和定制 |
| **离线可用** | ❌ 必须联网 | ✅ 完全离线运行 |
| **并发限制** | ❌ 有 QPS 限制 | ✅ 取决于硬件 |
| **运维复杂度** | ✅ 无需运维 | ❌ 需要自己维护 |

---

### 1.2 适用场景分析

**✅ 适合本地部署的场景：**

1. **数据敏感行业**
   - 金融：客户信息、交易数据
   - 医疗：病历、诊断报告
   - 政府：机密文档

2. **高频调用场景**
   - 智能客服（日均 10 万+ 次调用）
   - 代码助手（开发者频繁使用）
   - 实时翻译

3. **成本敏感场景**
   - 初创公司（预算有限）
   - 内部工具（无直接收益）
   - 测试环境（大量测试调用）

4. **离线环境**
   - 内网隔离系统
   - 边缘计算设备
   - 移动部署

**❌ 不适合本地部署的场景：**

1. 偶尔使用（云端 API 更经济）
2. 需要最强模型能力（本地硬件受限）
3. 无运维团队（云端省心）

---

### 1.3 我的成本对比案例

**场景：** 企业内部知识库问答系统

**云端 API 方案（通义千问）：**
- 日均调用：5000 次
- 平均 Token：2000/次
- 月度 Token：3 亿
- 费用：0.008 元/千 Token = **2400 元/月**
- 年度费用：**28,800 元**

**本地部署方案（Qwen2-7B）：**
- 服务器：RTX 4090（24GB 显存）= 15,000 元
- 电费：500W × 24h × 365天 × 0.6元/kWh = **2,628 元/年**
- 首年总成本：15,000 + 2,628 = **17,628 元**
- 次年成本：**2,628 元/年**

**结论：** 第 1 年节省 11,172 元，第 2 年起每年节省 26,172 元！💰

---

## 二、主流开源大模型对比选型

### 2.1 热门开源模型对比表

| 模型 | 参数量 | 显存需求 | 中文能力 | 推理速度 | 许可证 | 适用场景 |
|------|-------|---------|---------|---------|--------|---------|
| **Qwen2-7B** | 7B | 14GB | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Apache 2.0 | **通用场景首选** ✅ |
| **Qwen2-72B** | 72B | 140GB | ⭐⭐⭐⭐⭐ | ⭐⭐ | Apache 2.0 | 高质量要求 |
| **Llama3-8B** | 8B | 16GB | ⭐⭐ | ⭐⭐⭐⭐ | Llama 3 | 英文场景 |
| **Llama3-70B** | 70B | 140GB | ⭐⭐ | ⭐⭐ | Llama 3 | 英文高质量 |
| **ChatGLM3-6B** | 6B | 12GB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | GLM | 中文对话 |
| **Yi-6B** | 6B | 12GB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Apache 2.0 | 中英双语 |
| **Baichuan2-7B** | 7B | 14GB | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 商用需授权 | 中文场景 |

---

### 2.2 选型建议

**🎯 推荐方案：**

**场景 1：个人学习/小规模应用**
- 模型：Qwen2-7B 或 ChatGLM3-6B
- 显存：12-16GB（RTX 3060/4060）
- 理由：资源需求低，中文能力强

**场景 2：企业级应用**
- 模型：Qwen2-72B（多卡）或 Qwen2-7B（单卡）
- 显存：24GB+（RTX 4090/A6000）
- 理由：性能好，Apache 2.0 协议商用友好

**场景 3：英文为主**
- 模型：Llama3-8B 或 Llama3-70B
- 显存：16GB+
- 理由：英文能力最强

**场景 4：资源受限**
- 模型：Qwen2-1.5B 或 Phi-3-mini
- 显存：4-8GB
- 理由：轻量级，可 CPU 运行

---

### 2.3 我的选择：Qwen2-7B

**选择理由：**

1. **中文能力强**：阿里通义千问系列，中文训练数据充足
2. **性能平衡**：7B 参数，质量和速度兼顾
3. **显存友好**：量化后 14GB 显存即可运行
4. **协议友好**：Apache 2.0，可商用
5. **生态完善**：Ollama、LangChain4j 完美支持
6. **持续更新**：阿里团队活跃维护

---

## 三、Ollama 安装与配置

### 3.1 什么是 Ollama？

**Ollama** 是一个开源的大模型运行框架，特点：

- ✅ 一键安装，开箱即用
- ✅ 支持多种开源模型
- ✅ 自动量化压缩
- ✅ 提供 REST API
- ✅ 跨平台（Linux/macOS/Windows）
- ✅ 活跃的社区支持

**官网：** [https://ollama.com](https://ollama.com)

---

### 3.2 Linux 服务器安装步骤

#### 步骤 1：检查系统要求

```bash
# 检查操作系统
cat /etc/os-release

# 检查 GPU（NVIDIA）
nvidia-smi

# 检查 CUDA 版本
nvcc --version

# 检查内存
free -h

# 检查磁盘空间
df -h
```

**最低要求：**
- CPU：4 核以上
- 内存：8GB+
- 磁盘：20GB 可用空间
- GPU（可选）：NVIDIA GTX 1060 6GB+

**推荐配置：**
- CPU：8 核以上
- 内存：32GB+
- 磁盘：100GB SSD
- GPU：RTX 3060 12GB+

---

#### 步骤 2：安装 Ollama

```bash
# 方法 1：官方脚本（推荐）
curl -fsSL https://ollama.com/install.sh | sh

# 方法 2：手动安装
# 下载二进制文件
wget https://ollama.com/download/ollama-linux-amd64.tgz

# 解压
tar -xzf ollama-linux-amd64.tgz

# 移动到系统目录
sudo mv ollama /usr/local/bin/

# 验证安装
ollama --version
```

**预期输出：**
```
ollama version is 0.1.38
```

---

#### 步骤 3：启动 Ollama 服务

```bash
# 前台启动（测试用）
ollama serve

# 后台启动（生产用）
nohup ollama serve > ollama.log 2>&1 &

# 验证服务
curl http://localhost:11434/api/tags
```

**预期响应：**
```json
{"models":[]}
```

---

#### 步骤 4：配置开机自启

创建 systemd 服务文件：

```bash
sudo vim /etc/systemd/system/ollama.service
```

内容如下：

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="OLLAMA_HOST=0.0.0.0:11434"

[Install]
WantedBy=default.target
```

启动服务：

```bash
# 重新加载 systemd
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start ollama

# 设置开机自启
sudo systemctl enable ollama

# 查看状态
sudo systemctl status ollama
```

---

### 3.3 配置环境变量

编辑 `~/.bashrc`：

```bash
# Ollama 配置
export OLLAMA_HOST=0.0.0.0:11434  # 监听所有网卡
export OLLAMA_ORIGINS=*            # 允许跨域
export OLLAMA_MODELS=/data/ollama/models  # 模型存储路径

# 使配置生效
source ~/.bashrc
```

**重要配置说明：**

| 变量 | 说明 | 默认值 |
|------|------|--------|
| OLLAMA_HOST | 监听地址 | 127.0.0.1:11434 |
| OLLAMA_ORIGINS | 允许的源 | * |
| OLLAMA_MODELS | 模型存储路径 | ~/.ollama/models |
| OLLAMA_KEEP_ALIVE | 模型保留时间 | 5m |
| OLLAMA_NUM_PARALLEL | 并行请求数 | 1 |

---

## 四、下载与运行大模型

### 4.1 下载 Qwen2-7B 模型

```bash
# 下载模型（首次约 4GB）
ollama pull qwen2:7b

# 查看已下载的模型
ollama list
```

**预期输出：**
```
NAME              ID              SIZE      MODIFIED
qwen2:7b          a1b2c3d4e5f6    4.7 GB    2 minutes ago
```

---

### 4.2 测试模型运行

```bash
# 交互式对话
ollama run qwen2:7b

# 输入问题
>>> 你好，请介绍一下你自己

# 退出
>>> /bye
```

---

### 4.3 API 调用测试

```bash
# 非流式调用
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2:7b",
  "prompt": "什么是 Spring Boot？",
  "stream": false
}'

# 流式调用
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2:7b",
  "prompt": "什么是 Spring Boot？",
  "stream": true
}'
```

---

### 4.4 常用模型管理命令

```bash
# 列出所有模型
ollama list

# 删除模型
ollama rm qwen2:7b

# 复制模型
ollama cp qwen2:7b my-qwen2

# 查看模型信息
ollama show qwen2:7b

# 导出模型
ollama cp qwen2:7b ./my-model.gguf
```

---

## 五、量化压缩技术详解

### 5.1 什么是量化？

**量化（Quantization）** 是将模型权重从高精度（FP16）转换为低精度（INT4/INT8），从而：

- ✅ 减少显存占用（降低 50%-75%）
- ✅ 提升推理速度（提升 2-4 倍）
- ⚠️ 轻微精度损失（通常 < 5%）

---

### 5.2 量化等级对比

| 量化等级 | 精度 | 显存占用 | 速度 | 质量损失 | 推荐场景 |
|---------|------|---------|------|---------|---------|
| **FP16** | 16-bit | 14GB | 1x | 0% | 高质量要求 |
| **Q8_0** | 8-bit | 7.5GB | 1.5x | < 1% | 平衡方案 |
| **Q4_K_M** | 4-bit | 4.7GB | 2.5x | 2-3% | **推荐** ✅ |
| **Q4_0** | 4-bit | 4.2GB | 3x | 3-5% | 资源受限 |
| **Q2_K** | 2-bit | 2.8GB | 4x | 5-10% | 极端场景 |

---

### 5.3 Ollama 自动量化

Ollama 默认使用 Q4_K_M 量化，无需手动操作：

```bash
# 下载的模型已经是量化版本
ollama pull qwen2:7b  # 自动使用 Q4_K_M
```

---

### 5.4 自定义量化配置

创建 Modelfile：

```bash
vim Modelfile
```

内容：

```dockerfile
FROM qwen2:7b

# 设置量化等级
PARAMETER num_ctx 4096
PARAMETER num_gpu_layers 35
PARAMETER temperature 0.7
```

构建自定义模型：

```bash
ollama create my-qwen2 -f Modelfile
```

---

## 六、OpenAI 格式适配层开发

### 6.1 为什么需要 OpenAI 格式？

**优势：**
1. **生态兼容**：LangChain、LlamaIndex 等库原生支持
2. **代码复用**：Java 应用无需修改即可切换模型
3. **标准化**：统一的请求/响应格式
4. **灵活性**：轻松切换本地模型和云端 API

---

### 6.2 Ollama 原生 API vs OpenAI API 对比

**Ollama 原生格式：**

```json
POST /api/generate
{
  "model": "qwen2:7b",
  "prompt": "你好",
  "stream": false
}
```

**OpenAI 格式：**

```json
POST /v1/chat/completions
{
  "model": "qwen2:7b",
  "messages": [
    {"role": "user", "content": "你好"}
  ],
  "stream": false
}
```

---

### 6.3 使用 Ollama 内置 OpenAI 兼容接口

**Ollama 0.1.30+ 版本已内置 OpenAI 兼容接口！**

```bash
# 直接调用 OpenAI 格式接口
curl http://localhost:11434/v1/chat/completions -d '{
  "model": "qwen2:7b",
  "messages": [
    {"role": "user", "content": "你好"}
  ],
  "stream": false
}'
```

**响应格式：**

```json
{
  "id": "chatcmpl-123",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "qwen2:7b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "你好！我是通义千问..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 10,
    "completion_tokens": 50,
    "total_tokens": 60
  }
}
```

---

### 6.4 自定义适配层（高级）

如果需要更多定制功能，可以开发适配层：

**SpringBoot 适配器：**

```java
package com.example.ollama.adapter;

import lombok.Data;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Flux;

import java.util.List;
import java.util.Map;

/**
 * OpenAI 格式适配器
 */
@RestController
@RequestMapping("/v1")
@RequiredArgsConstructor
public class OpenAIAdapterController {

    private final WebClient webClient;

    /**
     * 聊天完成接口
     */
    @PostMapping("/chat/completions")
    public ResponseEntity<ChatCompletionResponse> chatCompletions(
            @RequestBody ChatCompletionRequest request) {
        
        // 转换为 Ollama 格式
        OllamaRequest ollamaRequest = convertToOllamaRequest(request);
        
        // 调用 Ollama API
        OllamaResponse ollamaResponse = webClient.post()
            .uri("http://localhost:11434/api/chat")
            .bodyValue(ollamaRequest)
            .retrieve()
            .bodyToMono(OllamaResponse.class)
            .block();
        
        // 转换为 OpenAI 格式
        ChatCompletionResponse response = convertToOpenAIResponse(ollamaResponse);
        
        return ResponseEntity.ok(response);
    }

    /**
     * 流式输出接口
     */
    @PostMapping(value = "/chat/completions", produces = "text/event-stream")
    public Flux<String> chatCompletionsStream(
            @RequestBody ChatCompletionRequest request) {
        
        // 实现流式转换逻辑
        // ...
        
        return Flux.empty();
    }

    private OllamaRequest convertToOllamaRequest(ChatCompletionRequest request) {
        // 转换逻辑
        return new OllamaRequest();
    }

    private ChatCompletionResponse convertToOpenAIResponse(OllamaResponse response) {
        // 转换逻辑
        return new ChatCompletionResponse();
    }
}

@Data
class ChatCompletionRequest {
    private String model;
    private List<Message> messages;
    private boolean stream;
    private double temperature;
}

@Data
class Message {
    private String role;
    private String content;
}

@Data
class ChatCompletionResponse {
    private String id;
    private String object;
    private long created;
    private String model;
    private List<Choice> choices;
    private Usage usage;
}

@Data
class Choice {
    private int index;
    private Message message;
    private String finishReason;
}

@Data
class Usage {
    private int promptTokens;
    private int completionTokens;
    private int totalTokens;
}
```

---

## 七、Java 应用集成实战

### 7.1 LangChain4j 集成 Ollama

**Maven 依赖：**

```xml
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-ollama</artifactId>
    <version>0.26.0</version>
</dependency>
```

---

**配置类：**

```java
package com.example.config;

import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.ollama.OllamaChatModel;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OllamaConfig {

    @Value("${ollama.base-url:http://localhost:11434}")
    private String baseUrl;

    @Value("${ollama.model:qwen2:7b}")
    private String model;

    @Bean
    public ChatLanguageModel chatModel() {
        return OllamaChatModel.builder()
            .baseUrl(baseUrl)
            .modelName(model)
            .temperature(0.7)
            .timeout(java.time.Duration.ofSeconds(60))
            .build();
    }
}
```

---

**Service 实现：**

```java
package com.example.service;

import dev.langchain4j.model.chat.ChatLanguageModel;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service
@RequiredArgsConstructor
public class AiChatService {

    private final ChatLanguageModel chatModel;

    /**
     * 简单对话
     */
    public String chat(String userMessage) {
        return chatModel.generate(userMessage);
    }

    /**
     * RAG 问答
     */
    public String ragChat(String context, String question) {
        String prompt = String.format("""
            基于以下上下文回答问题：
            
            上下文：
            %s
            
            问题：%s
            
            回答：
            """, context, question);
        
        return chatModel.generate(prompt);
    }
}
```

---

### 7.2 多模型路由策略

```java
package com.example.service;

import dev.langchain4j.model.chat.ChatLanguageModel;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import java.util.Map;

@Service
@RequiredArgsConstructor
public class ModelRouterService {

    private final Map<String, ChatLanguageModel> models;

    /**
     * 根据场景选择模型
     */
    public String chat(String scene, String message) {
        ChatLanguageModel model = selectModel(scene);
        return model.generate(message);
    }

    private ChatLanguageModel selectModel(String scene) {
        return switch (scene) {
            case "simple" -> models.get("qwen2:1.5b");  // 轻量模型
            case "normal" -> models.get("qwen2:7b");    // 标准模型
            case "complex" -> models.get("qwen2:72b");  // 高质量模型
            default -> models.get("qwen2:7b");
        };
    }
}
```

---

## 八、性能优化与监控

### 8.1 并发优化

**调整 Ollama 配置：**

```bash
# 设置并行请求数
export OLLAMA_NUM_PARALLEL=4

# 设置模型保留时间（避免频繁加载）
export OLLAMA_KEEP_ALIVE=24h
```

---

### 8.2 缓存策略

**Redis 缓存高频问答：**

```java
@Service
@RequiredArgsConstructor
public class CachedAiService {

    private final ChatLanguageModel chatModel;
    private final RedisTemplate<String, String> redisTemplate;

    public String cachedChat(String question) {
        String cacheKey = "ai:cache:" + md5(question);
        
        // 查缓存
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // 调用模型
        String answer = chatModel.generate(question);
        
        // 写缓存（2小时过期）
        redisTemplate.opsForValue().set(cacheKey, answer, 2, TimeUnit.HOURS);
        
        return answer;
    }
}
```

---

### 8.3 监控指标

**Prometheus 监控：**

```java
@Component
public class OllamaMetrics {

    private final MeterRegistry meterRegistry;
    private final Counter requestCounter;
    private final Timer responseTimer;

    public OllamaMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.requestCounter = Counter.builder("ollama.requests.total")
            .register(meterRegistry);
        this.responseTimer = Timer.builder("ollama.response.time")
            .register(meterRegistry);
    }

    public void recordRequest(long durationMs) {
        requestCounter.increment();
        responseTimer.record(durationMs, TimeUnit.MILLISECONDS);
    }
}
```

---

## 九、生产环境部署方案

### 9.1 Docker Compose 部署

创建 `docker-compose.yml`：

```yaml
version: '3.8'

services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ./models:/root/.ollama/models
      - ./data:/root/.ollama/data
    environment:
      - OLLAMA_HOST=0.0.0.0:11434
      - OLLAMA_ORIGINS=*
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: always

  redis:
    image: redis:7-alpine
    container_name: ollama-redis
    ports:
      - "6379:6379"
    volumes:
      - ./redis-data:/data
    restart: always

  prometheus:
    image: prom/prometheus:latest
    container_name: ollama-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: always

  grafana:
    image: grafana/grafana:latest
    container_name: ollama-grafana
    ports:
      - "3000:3000"
    volumes:
      - ./grafana-data:/var/lib/grafana
    restart: always
```

启动：

```bash
docker-compose up -d
```

---

### 9.2 Nginx 反向代理

```nginx
server {
    listen 80;
    server_name ai.yourdomain.com;

    location / {
        proxy_pass http://localhost:11434;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # 流式输出支持
        proxy_buffering off;
        proxy_cache off;
    }

    # SSL 配置（生产环境必须）
    # listen 443 ssl;
    # ssl_certificate /path/to/cert.pem;
    # ssl_certificate_key /path/to/key.pem;
}
```

---

### 9.3 负载均衡（多机部署）

```nginx
upstream ollama_cluster {
    server 192.168.1.10:11434 weight=3;
    server 192.168.1.11:11434 weight=2;
    server 192.168.1.12:11434 weight=1;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://ollama_cluster;
        # ...
    }
}
```

---

## 十、常见问题与解决方案

### 问题 1：显存不足

**错误信息：**
```
CUDA out of memory
```

**解决方案：**

1. 使用更低量化等级
```bash
ollama pull qwen2:7b-q4_0  # 更小
```

2. 减少上下文长度
```bash
export OLLAMA_NUM_CTX=2048
```

3. 关闭其他 GPU 应用

---

### 问题 2：推理速度慢

**优化方案：**

1. 使用 GPU 而非 CPU
2. 升级显卡（RTX 3060 → RTX 4090）
3. 使用更小模型（7B → 1.5B）
4. 启用 TensorRT 加速

---

### 问题 3：模型下载失败

**解决方案：**

```bash
# 设置国内镜像
export OLLAMA_REGISTRY_URL=https://registry.ollama.ai

# 手动下载模型文件
wget https://example.com/qwen2-7b.gguf
ollama create qwen2:7b -f Modelfile
```

---

### 问题 4：服务不稳定

**解决方案：**

1. 使用 systemd 管理服务
2. 添加健康检查
3. 设置自动重启
4. 监控日志

```bash
# 查看日志
journalctl -u ollama -f

# 重启服务
sudo systemctl restart ollama
```

---

## 💡 总结

### 核心要点回顾

1. **本地部署优势明显**：数据隐私、成本控制、稳定性
2. **Ollama 是最佳选择**：简单易用、生态完善
3. **Qwen2-7B 性价比高**：中文能力强、资源需求适中
4. **OpenAI 格式兼容**：便于集成和切换
5. **生产环境需优化**：缓存、监控、负载均衡

### 下一步行动

1. 在测试环境部署 Ollama
2. 迁移现有 AI 应用到本地模型
3. 建立监控告警体系
4. 探索模型微调方案

---

*最后更新: 2026-05-17*

*作者：洛苡苑香 | Java 工程师转型 AI 应用开发中*
