# 从零搭建第一个 RAG Demo：通义千问 + Milvus 实战指南

> **写在前面：** 这是我真正动手搭建第一个 RAG Demo 的完整记录。从注册通义千问账号开始，到 Docker 安装 Milvus，再到用 SpringBoot + LangChain4j 实现完整的文档问答系统，每一步我都详细记录了操作过程、遇到的问题和解决方案。如果你也是第一次接触 AI 应用开发，跟着这篇文章一步步操作，2 小时内你也能跑通自己的第一个 RAG Demo！

---

## 📋 目录

- [一、环境准备](#一环境准备)
- [二、注册通义千问账号并获取 API Key](#二注册通义千问账号并获取-api-key)
- [三、Docker 安装 Milvus 向量数据库](#三docker-安装-milvus-向量数据库)
- [四、创建 SpringBoot 项目](#四创建-springboot-项目)
- [五、实现文档上传与解析](#五实现文档上传与解析)
- [六、实现向量化与存储](#六实现向量化与存储)
- [七、实现向量检索与问答](#七实现向量检索与问答)
- [八、测试与验证](#八测试与验证)
- [九、常见问题与解决方案](#九常见问题与解决方案)
- [十、我的心得体会](#十我的心得体会)

---

## 一、环境准备

### 1.1 软硬件要求

**操作系统：**
- Windows 10/11（推荐 WSL2）
- macOS 10.15+
- Linux（Ubuntu 20.04+）

**必需软件：**

| 软件 | 版本 | 用途 | 下载地址 |
|------|------|------|---------|
| JDK | 17+ | Java 运行环境 | [Oracle JDK](https://www.oracle.com/java/technologies/downloads/) |
| Maven | 3.6+ | 项目构建 | [Maven](https://maven.apache.org/download.cgi) |
| Docker | 20.10+ | 运行 Milvus | [Docker Desktop](https://www.docker.com/products/docker-desktop/) |
| IDE | - | 代码编辑 | IntelliJ IDEA / VS Code |
| Postman | - | API 测试 | [Postman](https://www.postman.com/downloads/) |

---

### 1.2 验证环境

**检查 JDK：**

```bash
java -version
```

预期输出：
```
openjdk version "17.0.2" 2022-01-18
OpenJDK Runtime Environment (build 17.0.2+8)
```

---

**检查 Maven：**

```bash
mvn -version
```

预期输出：
```
Apache Maven 3.8.6
```

---

**检查 Docker：**

```bash
docker --version
docker-compose --version
```

预期输出：
```
Docker version 24.0.2
Docker Compose version v2.19.1
```

---

**💡 我的经验：**
> 我第一次安装 Docker 时遇到了问题，Windows 上需要启用 WSL2。如果遇到 "Docker Desktop requires WSL 2 support" 错误，按照以下步骤操作：
> 
> 1. 以管理员身份打开 PowerShell
> 2. 执行：`wsl --install`
> 3. 重启电脑
> 4. 重新安装 Docker Desktop

---

## 二、注册通义千问账号并获取 API Key

### 2.1 注册阿里云账号

**步骤 1：访问 DashScope 控制台**

打开浏览器，访问：[https://dashscope.console.aliyun.com/](https://dashscope.console.aliyun.com/)

---

**步骤 2：登录/注册**

- 如果有阿里云账号，直接登录
- 如果没有，点击"免费注册"，使用手机号注册

![注册页面](../images/dashscope-register.png)

---

**步骤 3：实名认证**

首次使用需要完成实名认证（个人认证即可）：
1. 点击右上角头像 → "账号管理"
2. 选择"实名认证"
3. 使用支付宝扫码或上传身份证

**⚠️ 注意：** 实名认证审核通常需要 1-2 小时，建议提前完成。

---

### 2.2 开通 DashScope 服务

**步骤 1：进入控制台**

登录后，自动进入 DashScope 控制台首页。

---

**步骤 2：开通服务**

如果是首次使用，会提示开通服务：
1. 点击"立即开通"
2. 阅读并同意服务协议
3. 确认开通

![开通服务](../images/dashscope-activate.png)

---

**步骤 3：查看免费额度**

新用户有免费额度：
- 通义千问 qwen-turbo：每月 100 万 Token
- 文本嵌入 text-embedding-v1：每月 100 万 Token

**💡 心得：** 免费额度足够学习和小规模测试使用，不用担心费用问题。

---

### 2.3 创建 API Key

**步骤 1：进入 API Key 管理**

在控制台左侧菜单，点击"API-KEY 管理"。

---

**步骤 2：创建新的 API Key**

1. 点击"创建新的 API-KEY"
2. 输入名称（例如：`rag-demo-key`）
3. 点击"确定"

![创建 API Key](../images/dashscope-create-key.png)

---

**步骤 3：复制并保存 API Key**

⚠️ **重要：** API Key 只会显示一次，务必立即复制保存！

```
sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

**安全建议：**
- ✅ 保存在密码管理器中
- ✅ 放在环境变量或配置文件中
- ❌ 不要硬编码在代码里
- ❌ 不要上传到 GitHub

---

### 2.4 测试 API 调用

**方法 1：使用 curl 快速测试**

```bash
curl -X POST https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation \
  -H "Authorization: Bearer sk-your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-turbo",
    "input": {
      "messages": [
        {
          "role": "user",
          "content": "你好，请介绍一下你自己"
        }
      ]
    }
  }'
```

**预期响应：**

```json
{
  "output": {
    "text": "你好！我是通义千问，由阿里巴巴通义实验室独立开发的大型语言模型..."
  },
  "usage": {
    "total_tokens": 50,
    "input_tokens": 20,
    "output_tokens": 30
  },
  "request_id": "xxx-xxx-xxx"
}
```

---

**方法 2：使用 Postman 测试**

1. 打开 Postman，创建新请求
2. 设置：
   - Method: POST
   - URL: `https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation`
   - Headers:
     - `Authorization: Bearer sk-your-api-key`
     - `Content-Type: application/json`
   - Body (raw JSON):
     ```json
     {
       "model": "qwen-turbo",
       "input": {
         "messages": [
           {
             "role": "user",
             "content": "什么是 Spring Boot？"
           }
         ]
       }
     }
     ```

3. 点击"Send"，查看响应

![Postman 测试](../images/postman-test.png)

---

**💡 我的第一次调用经历：**

> 我第一次调用时遇到了 401 错误，原因是 API Key 复制时多了一个空格。后来我把 API Key 放在环境变量中，问题就解决了。
> 
> **推荐做法：**
> ```bash
> # ~/.bashrc 或 ~/.zshrc
> export DASHSCOPE_API_KEY="sk-your-api-key"
> 
> # 使配置生效
> source ~/.bashrc
> 
> # 验证
> echo $DASHSCOPE_API_KEY
> ```

---

## 三、Docker 安装 Milvus 向量数据库

### 3.1 为什么选择 Milvus？

**对比其他向量数据库：**

| 数据库 | 类型 | 优点 | 缺点 | 适用场景 |
|--------|------|------|------|---------|
| **Milvus** | 开源 | 高性能、功能全、社区活跃 | 部署稍复杂 | **企业级应用** ✅ |
| Chroma | 开源 | 轻量、易用 | 性能一般、功能少 | 小规模项目 |
| Pinecone | SaaS | 无需运维 | 收费、数据出境 | 快速原型 |
| Elasticsearch | 开源 | 生态成熟 | 向量搜索非主业 | 已有 ES 集群 |

**我选择 Milvus 的原因：**
1. 开源免费，可本地部署
2. 性能优秀，支持大规模数据
3. Java SDK 完善
4. 国内社区活跃，遇到问题容易找到解答

---

### 3.2 使用 Docker Compose 安装

**步骤 1：创建 docker-compose.yml**

在你的项目目录下创建 `milvus-docker-compose.yml`：

```yaml
version: '3.5'

services:
  etcd:
    container_name: milvus-etcd
    image: quay.io/coreos/etcd:v3.5.5
    environment:
      - ETCD_AUTO_COMPACTION_MODE=revision
      - ETCD_AUTO_COMPACTION_RETENTION=1000
      - ETCD_QUOTA_BACKEND_BYTES=4294967296
      - ETCD_SNAPSHOT_COUNT=50000
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/etcd:/etcd
    command: etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    healthcheck:
      test: ["CMD", "etcdctl", "endpoint", "health"]
      interval: 30s
      timeout: 20s
      retries: 3

  minio:
    container_name: milvus-minio
    image: minio/minio:RELEASE.2023-03-20T20-16-18Z
    environment:
      MINIO_ACCESS_KEY: minioadmin
      MINIO_SECRET_KEY: minioadmin
    ports:
      - "9001:9001"
      - "9000:9000"
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/minio:/minio_data
    command: minio server /minio_data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  standalone:
    container_name: milvus-standalone
    image: milvusdb/milvus:v2.3.0
    command: ["milvus", "run", "standalone"]
    security_opt:
      - seccomp:unconfined
    environment:
      ETCD_ENDPOINTS: etcd:2379
      MINIO_ADDRESS: minio:9000
    volumes:
      - ${DOCKER_VOLUME_DIRECTORY:-.}/volumes/milvus:/var/lib/milvus
    ports:
      - "19530:19530"
      - "9091:9091"
    depends_on:
      - "etcd"
      - "minio"

networks:
  default:
    name: milvus
```

---

**步骤 2：启动 Milvus**

```bash
# 创建目录
mkdir -p milvus-demo
cd milvus-demo

# 创建 docker-compose 文件
# （将上面的内容保存到 milvus-docker-compose.yml）

# 启动 Milvus
docker-compose -f milvus-docker-compose.yml up -d
```

**预期输出：**

```
Creating milvus-etcd    ... done
Creating milvus-minio   ... done
Creating milvus-standalone ... done
```

---

**步骤 3：验证安装**

```bash
# 查看容器状态
docker ps | grep milvus
```

**预期输出：**

```
CONTAINER ID   IMAGE                      STATUS         PORTS
xxx            milvusdb/milvus:v2.3.0     Up 2 minutes   0.0.0.0:19530->19530/tcp, 0.0.0.0:9091->9091/tcp
xxx            quay.io/coreos/etcd:v3.5.5 Up 2 minutes   2379/tcp
xxx            minio/minio                Up 2 minutes   0.0.0.0:9000-9001->9000-9001/tcp
```

---

**步骤 4：健康检查**

```bash
# 检查 Milvus 健康状态
curl http://localhost:9091/healthz
```

**预期响应：**

```json
{"status":"ok"}
```

---

**💡 我遇到的问题和解决方案：**

**问题 1：端口被占用**

```
Error starting userland proxy: listen tcp4 0.0.0.0:19530: bind: address already in use
```

**解决：**

```bash
# 查找占用端口的进程
lsof -i :19530

# 杀死进程
kill -9 <PID>

# 或者修改 docker-compose.yml 中的端口映射
ports:
  - "19531:19530"  # 改为 19531
```

---

**问题 2：容器启动失败**

```
milvus-standalone exited with code 1
```

**解决：**

```bash
# 查看日志
docker logs milvus-standalone

# 常见原因：
# 1. 磁盘空间不足 → 清理磁盘
# 2. 内存不足 → 增加 Docker 内存限制
# 3. 配置文件错误 → 检查 docker-compose.yml
```

---

**问题 3：连接超时**

```
io.milvus.exception.MilvusException: Connect to Milvus failed
```

**解决：**

```bash
# 等待 30 秒，Milvus 启动需要时间
sleep 30

# 验证端口是否监听
netstat -an | grep 19530

# 检查防火墙
sudo ufw allow 19530/tcp
```

---

### 3.3 安装 Milvus 可视化工具（可选）

**Attu - Milvus 管理工具**

```bash
docker run -p 8000:3000 -e MILVUS_URL=host.docker.internal:19530 zilliz/attu:v2.3
```

访问：[http://localhost:8000](http://localhost:8000)

![Attu 界面](../images/attu-ui.png)

**功能：**
- 查看 Collection
- 浏览向量数据
- 执行向量搜索
- 监控性能指标

---

## 四、创建 SpringBoot 项目

### 4.1 使用 Spring Initializr 创建项目

**步骤 1：访问 Spring Initializr**

打开：[https://start.spring.io/](https://start.spring.io/)

**步骤 2：配置项目**

- Project: Maven
- Language: Java
- Spring Boot: 3.2.x
- Group: com.example
- Artifact: rag-demo
- Name: rag-demo
- Description: RAG Demo with LangChain4j
- Package name: com.example.ragdemo
- Packaging: Jar
- Java: 17

**步骤 3：添加依赖**

点击"Add Dependencies"，搜索并添加：
- Spring Web
- Spring Data Redis（可选，用于缓存）

**步骤 4：生成项目**

点击"GENERATE"，下载 zip 文件并解压。

---

### 4.2 添加 LangChain4j 依赖

编辑 `pom.xml`，添加以下依赖：

```xml
<dependencies>
    <!-- SpringBoot Web -->
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
    
    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    
    <!-- Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

### 4.3 配置 application.yml

创建 `src/main/resources/application.yml`：

```yaml
server:
  port: 8080

# 通义千问配置
dashscope:
  api:
    key: ${DASHSCOPE_API_KEY:your-api-key-here}
  embedding:
    model: text-embedding-v1
  chat:
    model: qwen-turbo

# Milvus 配置
milvus:
  host: localhost
  port: 19530
  collection-name: rag_demo_collection

# 文件上传配置
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 10MB
```

**⚠️ 注意：** 将 `your-api-key-here` 替换为你的真实 API Key，或者设置环境变量 `DASHSCOPE_API_KEY`。

---

### 4.4 项目结构

```
rag-demo/
├── src/main/java/com/example/ragdemo/
│   ├── RagDemoApplication.java
│   ├── config/
│   │   ├── LlmConfig.java
│   │   └── MilvusConfig.java
│   ├── controller/
│   │   ├── DocumentController.java
│   │   └── ChatController.java
│   ├── service/
│   │   ├── DocumentService.java
│   │   └── RagService.java
│   ├── model/
│   │   ├── RagResponse.java
│   │   └── Source.java
│   └── util/
│       └── PdfParser.java
├── src/main/resources/
│   └── application.yml
└── pom.xml
```

---

## 五、实现文档上传与解析

### 5.1 PDF 解析工具类

创建 `src/main/java/com/example/ragdemo/util/PdfParser.java`：

```java
package com.example.ragdemo.util;

import lombok.extern.slf4j.Slf4j;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;

import java.io.File;
import java.io.IOException;

/**
 * PDF 解析工具类
 */
@Slf4j
public class PdfParser {
    
    /**
     * 从 PDF 文件提取文本
     */
    public static String extractText(File pdfFile) {
        try (PDDocument document = PDDocument.load(pdfFile)) {
            PDFTextStripper stripper = new PDFTextStripper();
            
            // 设置起始页和结束页（null 表示全部）
            stripper.setStartPage(1);
            stripper.setEndPage(document.getNumberOfPages());
            
            String text = stripper.getText(document);
            
            log.info("PDF 解析成功: {}, 页数: {}, 文本长度: {}", 
                pdfFile.getName(), 
                document.getNumberOfPages(),
                text.length()
            );
            
            return text;
            
        } catch (IOException e) {
            log.error("PDF 解析失败: {}", pdfFile.getName(), e);
            throw new RuntimeException("PDF 解析失败", e);
        }
    }
    
    /**
     * 从字节数组提取文本
     */
    public static String extractText(byte[] pdfBytes) {
        try (PDDocument document = PDDocument.load(pdfBytes)) {
            PDFTextStripper stripper = new PDFTextStripper();
            return stripper.getText(document);
        } catch (IOException e) {
            throw new RuntimeException("PDF 解析失败", e);
        }
    }
}
```

---

### 5.2 文档服务

创建 `src/main/java/com/example/ragdemo/service/DocumentService.java`：

```java
package com.example.ragdemo.service;

import com.example.ragdemo.util.PdfParser;
import dev.langchain4j.data.document.Document;
import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.document.splitter.DocumentSplitters;
import dev.langchain4j.data.segment.TextSegment;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

/**
 * 文档处理服务
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class DocumentService {
    
    private static final int CHUNK_SIZE = 500;  // 每块字符数
    private static final int CHUNK_OVERLAP = 50; // 重叠字符数
    
    /**
     * 上传并处理文档
     */
    public List<TextSegment> processDocument(MultipartFile file) {
        try {
            // 1. 保存临时文件
            File tempFile = saveTempFile(file);
            
            // 2. 解析文档
            String text = parseDocument(tempFile, file.getOriginalFilename());
            
            // 3. 文本分块
            List<TextSegment> segments = splitText(text);
            
            // 4. 添加元数据
            addMetadata(segments, file.getOriginalFilename());
            
            log.info("文档处理完成: {}, 分块数: {}", file.getOriginalFilename(), segments.size());
            
            // 5. 删除临时文件
            Files.deleteIfExists(tempFile.toPath());
            
            return segments;
            
        } catch (Exception e) {
            log.error("文档处理失败: {}", file.getOriginalFilename(), e);
            throw new RuntimeException("文档处理失败", e);
        }
    }
    
    /**
     * 保存临时文件
     */
    private File saveTempFile(MultipartFile file) throws IOException {
        String tempDir = System.getProperty("java.io.tmpdir");
        String fileName = UUID.randomUUID().toString() + "_" + file.getOriginalFilename();
        Path tempPath = Path.of(tempDir, fileName);
        
        Files.copy(file.getInputStream(), tempPath, StandardCopyOption.REPLACE_EXISTING);
        
        return tempPath.toFile();
    }
    
    /**
     * 解析文档（支持 PDF、TXT）
     */
    private String parseDocument(File file, String originalFilename) {
        if (originalFilename.endsWith(".pdf")) {
            return PdfParser.extractText(file);
        } else if (originalFilename.endsWith(".txt")) {
            try {
                return Files.readString(file.toPath());
            } catch (IOException e) {
                throw new RuntimeException("TXT 解析失败", e);
            }
        } else {
            throw new IllegalArgumentException("不支持的文件格式: " + originalFilename);
        }
    }
    
    /**
     * 文本分块
     */
    private List<TextSegment> splitText(String text) {
        DocumentSplitter splitter = DocumentSplitters.recursive(CHUNK_SIZE, CHUNK_OVERLAP);
        Document document = Document.from(text);
        return splitter.split(document);
    }
    
    /**
     * 添加元数据
     */
    private void addMetadata(List<TextSegment> segments, String filename) {
        for (TextSegment segment : segments) {
            segment.metadata().put("filename", filename);
            segment.metadata().put("uploadTime", String.valueOf(System.currentTimeMillis()));
        }
    }
}
```

---

### 5.3 文档上传 Controller

创建 `src/main/java/com/example/ragdemo/controller/DocumentController.java`：

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.service.DocumentService;
import dev.langchain4j.data.segment.TextSegment;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.HashMap;
import java.util.List;
import java.util.Map;

/**
 * 文档管理接口
 */
@RestController
@RequestMapping("/api/documents")
@RequiredArgsConstructor
@Slf4j
public class DocumentController {
    
    private final DocumentService documentService;
    
    /**
     * 上传文档
     */
    @PostMapping("/upload")
    public ResponseEntity<Map<String, Object>> uploadDocument(@RequestParam("file") MultipartFile file) {
        try {
            log.info("收到文档上传请求: {}", file.getOriginalFilename());
            
            // 处理文档
            List<TextSegment> segments = documentService.processDocument(file);
            
            // 返回结果
            Map<String, Object> result = new HashMap<>();
            result.put("success", true);
            result.put("message", "文档上传成功");
            result.put("filename", file.getOriginalFilename());
            result.put("segments", segments.size());
            
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            log.error("文档上传失败", e);
            
            Map<String, Object> result = new HashMap<>();
            result.put("success", false);
            result.put("message", "文档上传失败: " + e.getMessage());
            
            return ResponseEntity.internalServerError().body(result);
        }
    }
}
```

---

## 六、实现向量化与存储

### 6.1 配置 LangChain4j

创建 `src/main/java/com/example/ragdemo/config/LlmConfig.java`：

```java
package com.example.ragdemo.config;

import dev.langchain4j.model.dashscope.DashScopeChatModel;
import dev.langchain4j.model.dashscope.DashScopeEmbeddingModel;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * 大模型配置
 */
@Configuration
public class LlmConfig {
    
    @Value("${dashscope.api.key}")
    private String apiKey;
    
    @Value("${dashscope.embedding.model}")
    private String embeddingModel;
    
    @Value("${dashscope.chat.model}")
    private String chatModel;
    
    /**
     * 配置 Embedding 模型
     */
    @Bean
    public DashScopeEmbeddingModel embeddingModel() {
        return DashScopeEmbeddingModel.builder()
            .apiKey(apiKey)
            .modelName(embeddingModel)
            .build();
    }
    
    /**
     * 配置 Chat 模型
     */
    @Bean
    public DashScopeChatModel chatModel() {
        return DashScopeChatModel.builder()
            .apiKey(apiKey)
            .modelName(chatModel)
            .temperature(0.7)  // 温度参数，控制随机性
            .build();
    }
}
```

---

### 6.2 配置 Milvus

创建 `src/main/java/com/example/ragdemo/config/MilvusConfig.java`：

```java
package com.example.ragdemo.config;

import dev.langchain4j.store.embedding.milvus.MilvusEmbeddingStore;
import io.milvus.param.IndexType;
import io.milvus.param.MetricType;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Milvus 向量数据库配置
 */
@Configuration
public class MilvusConfig {
    
    @Value("${milvus.host}")
    private String host;
    
    @Value("${milvus.port}")
    private int port;
    
    @Value("${milvus.collection-name}")
    private String collectionName;
    
    /**
     * 配置 Milvus Embedding Store
     */
    @Bean
    public MilvusEmbeddingStore embeddingStore() {
        return MilvusEmbeddingStore.builder()
            .host(host)
            .port(port)
            .collectionName(collectionName)
            .dimension(1536)  // 向量维度（text-embedding-v1 的维度）
            .indexType(IndexType.HNSW)  // 索引类型
            .metricType(MetricType.COSINE)  // 相似度度量
            .build();
    }
}
```

---

### 6.3 向量化存储服务

创建 `src/main/java/com/example/ragdemo/service/VectorStoreService.java`：

```java
package com.example.ragdemo.service;

import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.dashscope.DashScopeEmbeddingModel;
import dev.langchain4j.store.embedding.milvus.MilvusEmbeddingStore;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;

/**
 * 向量存储服务
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class VectorStoreService {
    
    private final MilvusEmbeddingStore embeddingStore;
    private final DashScopeEmbeddingModel embeddingModel;
    
    /**
     * 批量存入文档片段
     */
    public void storeSegments(List<TextSegment> segments) {
        try {
            log.info("开始向量化并存储 {} 个文档片段", segments.size());
            
            // 批量向量化（LangChain4j 自动处理）
            embeddingStore.addAll(segments);
            
            log.info("文档片段存储成功");
            
        } catch (Exception e) {
            log.error("文档片段存储失败", e);
            throw new RuntimeException("文档存储失败", e);
        }
    }
    
    /**
     * 清空集合（用于测试）
     */
    public void clearCollection() {
        try {
            embeddingStore.removeAll();
            log.info("集合已清空");
        } catch (Exception e) {
            log.error("清空集合失败", e);
        }
    }
}
```

---

### 6.4 更新 DocumentController

在 `DocumentController` 中添加向量存储逻辑：

```java
@RestController
@RequestMapping("/api/documents")
@RequiredArgsConstructor
@Slf4j
public class DocumentController {
    
    private final DocumentService documentService;
    private final VectorStoreService vectorStoreService;  // 新增
    
    @PostMapping("/upload")
    public ResponseEntity<Map<String, Object>> uploadDocument(@RequestParam("file") MultipartFile file) {
        try {
            log.info("收到文档上传请求: {}", file.getOriginalFilename());
            
            // 1. 处理文档
            List<TextSegment> segments = documentService.processDocument(file);
            
            // 2. 向量化并存储（新增）
            vectorStoreService.storeSegments(segments);
            
            // 3. 返回结果
            Map<String, Object> result = new HashMap<>();
            result.put("success", true);
            result.put("message", "文档上传成功");
            result.put("filename", file.getOriginalFilename());
            result.put("segments", segments.size());
            
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            log.error("文档上传失败", e);
            
            Map<String, Object> result = new HashMap<>();
            result.put("success", false);
            result.put("message", "文档上传失败: " + e.getMessage());
            
            return ResponseEntity.internalServerError().body(result);
        }
    }
}
```

---

## 七、实现向量检索与问答

### 7.1 RAG 服务

创建 `src/main/java/com/example/ragdemo/service/RagService.java`：

```java
package com.example.ragdemo.service;

import com.example.ragdemo.model.RagResponse;
import com.example.ragdemo.model.Source;
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.message.UserMessage;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.chat.ChatLanguageModel;
import dev.langchain4j.model.dashscope.DashScopeChatModel;
import dev.langchain4j.model.dashscope.DashScopeEmbeddingModel;
import dev.langchain4j.model.output.Response;
import dev.langchain4j.store.embedding.EmbeddingMatch;
import dev.langchain4j.store.embedding.milvus.MilvusEmbeddingStore;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.stream.Collectors;

/**
 * RAG 问答服务
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class RagService {
    
    private static final int TOP_K = 5;  // 检索 Top-K 个文档
    private static final double SIMILARITY_THRESHOLD = 0.7;  // 相似度阈值
    
    private final MilvusEmbeddingStore embeddingStore;
    private final DashScopeEmbeddingModel embeddingModel;
    private final DashScopeChatModel chatModel;
    
    /**
     * RAG 问答
     */
    public RagResponse query(String question) {
        long startTime = System.currentTimeMillis();
        
        try {
            log.info("收到问答请求: {}", question);
            
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
                return RagResponse.builder()
                    .answer("抱歉，没有找到相关的文档内容。")
                    .sources(List.of())
                    .build();
            }
            
            // 3. 组装上下文
            String context = buildContext(matches);
            
            // 4. 构建 Prompt
            String prompt = buildPrompt(question, context);
            
            // 5. 调用大模型
            Response<String> response = chatModel.generate(prompt);
            String answer = response.content();
            
            // 6. 提取引用来源
            List<Source> sources = extractSources(matches);
            
            long duration = System.currentTimeMillis() - startTime;
            
            log.info("问答完成，耗时: {}ms", duration);
            
            return RagResponse.builder()
                .answer(answer)
                .sources(sources)
                .durationMs(duration)
                .build();
            
        } catch (Exception e) {
            log.error("问答失败", e);
            throw new RuntimeException("问答失败", e);
        }
    }
    
    /**
     * 组装上下文
     */
    private String buildContext(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> match.embedded().text())
            .collect(Collectors.joining("\n\n---\n\n"));
    }
    
    /**
     * 构建 Prompt
     */
    private String buildPrompt(String question, String context) {
        return String.format("""
            你是一个智能助手，请根据以下文档内容回答问题。
            
            文档内容：
            %s
            
            问题：%s
            
            要求：
            1. 答案必须基于文档内容
            2. 如果文档中没有相关信息，请明确告知
            3. 回答要简洁明了
            4. 列出引用来源
            """, context, question);
    }
    
    /**
     * 提取引用来源
     */
    private List<Source> extractSources(List<EmbeddingMatch<TextSegment>> matches) {
        return matches.stream()
            .map(match -> Source.builder()
                .filename(match.embedded().metadata().get("filename"))
                .similarityScore(match.score())
                .text(match.embedded().text().substring(0, Math.min(100, match.embedded().text().length())))
                .build())
            .collect(Collectors.toList());
    }
}
```

---

### 7.2 响应模型

创建 `src/main/java/com/example/ragdemo/model/RagResponse.java`：

```java
package com.example.ragdemo.model;

import lombok.Builder;
import lombok.Data;

import java.util.List;

/**
 * RAG 问答响应
 */
@Data
@Builder
public class RagResponse {
    
    private String answer;              // 答案
    private List<Source> sources;       // 引用来源
    private Long durationMs;            // 响应时间（毫秒）
}
```

---

创建 `src/main/java/com/example/ragdemo/model/Source.java`：

```java
package com.example.ragdemo.model;

import lombok.Builder;
import lombok.Data;

/**
 * 引用来源
 */
@Data
@Builder
public class Source {
    
    private String filename;            // 文件名
    private Double similarityScore;     // 相似度分数
    private String text;                // 文档片段（前 100 字）
}
```

---

### 7.3 问答 Controller

创建 `src/main/java/com/example/ragdemo/controller/ChatController.java`：

```java
package com.example.ragdemo.controller;

import com.example.ragdemo.model.RagResponse;
import com.example.ragdemo.service.RagService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

/**
 * 聊天接口
 */
@RestController
@RequestMapping("/api/chat")
@RequiredArgsConstructor
@Slf4j
public class ChatController {
    
    private final RagService ragService;
    
    /**
     * 问答接口
     */
    @PostMapping("/query")
    public ResponseEntity<?> query(@RequestBody Map<String, String> request) {
        try {
            String question = request.get("question");
            
            if (question == null || question.trim().isEmpty()) {
                return ResponseEntity.badRequest().body(Map.of(
                    "success", false,
                    "message", "问题不能为空"
                ));
            }
            
            RagResponse response = ragService.query(question);
            
            Map<String, Object> result = new HashMap<>();
            result.put("success", true);
            result.put("data", response);
            
            return ResponseEntity.ok(result);
            
        } catch (Exception e) {
            log.error("问答失败", e);
            
            return ResponseEntity.internalServerError().body(Map.of(
                "success", false,
                "message", "问答失败: " + e.getMessage()
            ));
        }
    }
}
```

---

## 八、测试与验证

### 8.1 启动应用

```bash
# 进入项目目录
cd rag-demo

# 编译项目
mvn clean package -DskipTests

# 运行应用
java -jar target/rag-demo-0.0.1-SNAPSHOT.jar
```

**预期输出：**

```
Started RagDemoApplication in 5.234 seconds
```

---

### 8.2 上传测试文档

**准备测试文档：**

创建一个简单的 PDF 文档，内容例如：

```
Spring Boot 入门教程

Spring Boot 是由 Pivotal 团队提供的全新框架，
其设计目的是用来简化新 Spring 应用的初始搭建以及开发过程。

核心特性：
1. 独立运行的 Spring 项目
2. 内嵌 Servlet 容器
3. 提供 starter 简化 Maven 配置
4. 自动配置 Spring
5. 无代码生成和 XML 配置
```

---

**使用 Postman 上传：**

1. 创建 POST 请求
2. URL: `http://localhost:8080/api/documents/upload`
3. Body → form-data:
   - Key: `file` (类型选择 File)
   - Value: 选择你的 PDF 文件

![上传文档](../images/upload-document.png)

**预期响应：**

```json
{
  "success": true,
  "message": "文档上传成功",
  "filename": "spring-boot-intro.pdf",
  "segments": 8
}
```

---

### 8.3 测试问答

**使用 Postman 提问：**

1. 创建 POST 请求
2. URL: `http://localhost:8080/api/chat/query`
3. Headers: `Content-Type: application/json`
4. Body (raw JSON):
   ```json
   {
     "question": "Spring Boot 的核心特性有哪些？"
   }
   ```

**预期响应：**

```json
{
  "success": true,
  "data": {
    "answer": "根据文档内容，Spring Boot 的核心特性包括：\n\n1. 独立运行的 Spring 项目\n2. 内嵌 Servlet 容器\n3. 提供 starter 简化 Maven 配置\n4. 自动配置 Spring\n5. 无代码生成和 XML 配置",
    "sources": [
      {
        "filename": "spring-boot-intro.pdf",
        "similarityScore": 0.89,
        "text": "核心特性：\n1. 独立运行的 Spring 项目\n2. 内嵌 Servlet 容器..."
      }
    ],
    "durationMs": 3245
  }
}
```

---

### 8.4 更多测试问题

**测试 1：简单问题**

```json
{
  "question": "什么是 Spring Boot？"
}
```

---

**测试 2：具体问题**

```json
{
  "question": "Spring Boot 如何简化 Maven 配置？"
}
```

---

**测试 3：无关问题（测试降级）**

```json
{
  "question": "今天天气怎么样？"
}
```

**预期响应：**

```json
{
  "success": true,
  "data": {
    "answer": "抱歉，没有找到相关的文档内容。",
    "sources": [],
    "durationMs": 1523
  }
}
```

---

### 8.5 使用 curl 测试

```bash
# 上传文档
curl -X POST http://localhost:8080/api/documents/upload \
  -F "file=@spring-boot-intro.pdf"

# 问答
curl -X POST http://localhost:8080/api/chat/query \
  -H "Content-Type: application/json" \
  -d '{"question": "Spring Boot 的核心特性有哪些？"}'
```

---

## 九、常见问题与解决方案

### 问题 1：API Key 无效

**错误信息：**

```
401 Unauthorized: Invalid API Key
```

**解决方案：**

1. 检查 API Key 是否正确复制（没有多余空格）
2. 确认已在 DashScope 控制台开通服务
3. 确认 API Key 未过期

```yaml
# 方式 1：直接在配置文件中
dashscope:
  api:
    key: sk-your-real-api-key

# 方式 2：使用环境变量（推荐）
export DASHSCOPE_API_KEY="sk-your-real-api-key"
```

---

### 问题 2：连接 Milvus 失败

**错误信息：**

```
io.milvus.exception.MilvusException: Connect to Milvus failed
```

**解决方案：**

1. 确认 Milvus 容器正在运行

```bash
docker ps | grep milvus
```

2. 等待 30 秒（Milvus 启动需要时间）

```bash
sleep 30
```

3. 检查端口是否监听

```bash
netstat -an | grep 19530
```

4. 测试连接

```bash
telnet localhost 19530
```

---

### 问题 3：向量维度不匹配

**错误信息：**

```
Dimension mismatch: expected 1536, got 768
```

**解决方案：**

确认 Embedding 模型和 Milvus 配置的维度一致：

```yaml
# text-embedding-v1 的维度是 1536
milvus:
  collection-name: rag_demo_collection
  
# 在 MilvusConfig 中
.dimension(1536)
```

如果使用其他模型，需要调整维度：
- text-embedding-v1: 1536
- text-embedding-v2: 1536

---

### 问题 4：PDF 解析乱码

**症状：** 解析出的文本是乱码

**解决方案：**

1. 确认 PDF 文件不是扫描件（需要 OCR）
2. 尝试其他 PDF 文件测试
3. 检查文件编码

```java
// 指定编码
String text = new String(files.readAllBytes(), StandardCharsets.UTF_8);
```

---

### 问题 5：响应时间太长

**症状：** 问答超过 10 秒

**优化方案：**

1. 减少 Top-K 值（从 5 改为 3）
2. 降低相似度阈值（从 0.7 改为 0.6）
3. 使用更快的模型（qwen-turbo 而非 qwen-max）
4. 加入缓存（后续优化）

```java
private static final int TOP_K = 3;  // 减少检索数量
private static final double SIMILARITY_THRESHOLD = 0.6;  // 降低阈值
```

---

## 十、我的心得体会

### 10.1 整个流程总结

**从零到一的完整流程：**

```mermaid
graph LR
    A[注册通义千问] --> B[获取 API Key]
    B --> C[Docker 安装 Milvus]
    C --> D[创建 SpringBoot 项目]
    D --> E[实现文档上传]
    E --> F[实现向量化存储]
    F --> G[实现向量检索]
    G --> H[实现问答接口]
    H --> I[测试验证]
```

**总耗时：** 我第一次花了约 3 小时（包括排查问题）

**如果跟着本文操作：** 预计 2 小时内完成

---

### 10.2 最大的收获

**1. 理解了 RAG 的完整流程**

以前只是理论了解，现在亲手实现了：
- 文档解析 → 分块 → 向量化 → 存储 → 检索 → 问答

**2. 掌握了 LangChain4j 的使用**

发现 Java 做 AI 应用也很方便，不需要转 Python！

**3. 积累了踩坑经验**

- API Key 管理
- Docker 网络配置
- 向量维度匹配
- 异常处理

---

### 10.3 下一步优化方向

**短期（本周）：**
1. 加入 Redis 缓存（高频问答）
2. 加入限流保护（防止超额调用）
3. 加入监控指标（Prometheus）

**中期（本月）：**
1. 支持更多文档格式（Word、Excel）
2. 实现权限控制（多用户隔离）
3. 加入混合检索（关键词 + 向量）

**长期（本季）：**
1. 前端界面开发（Vue/React）
2. 多轮对话支持
3. 意图识别（路由到不同知识库）

---

### 10.4 给初学者的建议

**1. 先跑通，再优化**

不要一开始就追求完美，先把 Demo 跑起来，建立信心！

**2. 善用官方文档**

- [LangChain4j 文档](https://docs.langchain4j.dev/)
- [通义千问文档](https://help.aliyun.com/zh/dashscope/)
- [Milvus 文档](https://milvus.io/docs)

**3. 记录踩坑经验**

遇到问题时，记录解决方案，下次遇到同样问题能快速解决。

**4. 加入社区**

- GitHub Issues
- Stack Overflow
- 知乎、掘金等技术社区

---

### 10.5 成本估算

**免费额度：**
- 通义千问 qwen-turbo：100 万 Token/月
- text-embedding-v1：100 万 Token/月

**实际使用：**
- 上传 10 个文档（每个 5000 字）：约 5 万 Token
- 每天 100 次问答（每次 500 Token）：约 150 万 Token/月

**结论：** 免费额度足够学习和小规模测试！

---

## 🎯 总结

恭喜你！到这里，你已经成功搭建了第一个 RAG Demo！

**你学会了：**
- ✅ 注册通义千问并获取 API Key
- ✅ Docker 安装 Milvus 向量数据库
- ✅ 使用 SpringBoot + LangChain4j 实现 RAG
- ✅ 文档上传、解析、分块、向量化、存储
- ✅ 向量检索和大模型问答
- ✅ 完整的测试验证流程

**接下来：**
1. 继续优化这个 Demo（加入缓存、限流、监控）
2. 学习更高级的 RAG 技巧（混合检索、重排序）
3. 准备简历项目描述
4. 开始投递面试

加油！你已经迈出了 Java+AI 转型的第一步！🚀

---

*最后更新: 2026-05-14*

*作者：洛苡苑香 | Java 工程师转型 AI 应用开发中*
