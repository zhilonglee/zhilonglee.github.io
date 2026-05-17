# 图片资源清单

本文档列出了 `java-ai-first-rag-demo.md` 文章中需要的所有截图，请按照以下步骤补充。

---

## 📸 需要的图片列表

### 1. DashScope 相关图片

#### 1.1 dashscope-register.png
- **位置**: `articles/images/dashscope-register.png`
- **说明**: DashScope 注册页面截图
- **获取方式**: 
  1. 访问 https://dashscope.console.aliyun.com/
  2. 点击"免费注册"
  3. 截图注册页面

---

#### 1.2 dashscope-activate.png
- **位置**: `articles/images/dashscope-activate.png`
- **说明**: DashScope 服务开通页面截图
- **获取方式**:
  1. 登录 DashScope 控制台
  2. 首次使用会弹出开通提示
  3. 截图开通页面

---

#### 1.3 dashscope-create-key.png
- **位置**: `articles/images/dashscope-create-key.png`
- **说明**: 创建 API Key 对话框截图
- **获取方式**:
  1. 进入"API-KEY 管理"
  2. 点击"创建新的 API-KEY"
  3. 截图弹出的对话框

---

#### 1.4 postman-test.png
- **位置**: `articles/images/postman-test.png`
- **说明**: Postman 测试大模型 API 的截图
- **获取方式**:
  1. 打开 Postman
  2. 配置请求（URL、Headers、Body）
  3. 发送请求并收到响应
  4. 截图整个 Postman 窗口

---

### 2. Milvus 相关图片

#### 2.1 attu-ui.png
- **位置**: `articles/images/attu-ui.png`
- **说明**: Attu Milvus 管理工具界面截图
- **获取方式**:
  1. 启动 Attu: `docker run -p 8000:3000 -e MILVUS_URL=host.docker.internal:19530 zilliz/attu:v2.3`
  2. 访问 http://localhost:8000
  3. 连接到 Milvus
  4. 截图主界面

---

### 3. SpringBoot 项目相关图片

#### 3.1 spring-initializr.png
- **位置**: `articles/images/spring-initializr.png`
- **说明**: Spring Initializr 项目配置页面截图
- **获取方式**:
  1. 访问 https://start.spring.io/
  2. 配置项目信息（Group、Artifact、Dependencies 等）
  3. 截图配置页面

---

### 4. RAG Demo 测试相关图片

#### 4.1 upload-document.png
- **位置**: `articles/images/upload-document.png`
- **说明**: Postman 上传文档接口测试截图
- **获取方式**:
  1. 启动 RAG Demo 应用
  2. 打开 Postman
  3. 配置 POST 请求到 `/api/documents/upload`
  4. 选择 form-data，添加 file 字段
  5. 选择一个 PDF 文件
  6. 发送请求并收到成功响应
  7. 截图整个 Postman 窗口

---

## 🎨 图片要求

### 尺寸建议
- **宽度**: 1200-1600px
- **高度**: 根据内容自适应
- **格式**: PNG（支持透明背景）

### 质量要求
- **清晰度**: 文字清晰可读
- **完整性**: 包含完整的窗口或界面
- **隐私保护**: 
  - ⚠️ **隐藏 API Key**（用马赛克或替换为 `sk-xxxxxxxx`）
  - ⚠️ **隐藏个人敏感信息**（手机号、邮箱等）

### 命名规范
- 使用小写字母
- 单词之间用连字符 `-`
- 例如: `dashscope-register.png`

---

## 📝 临时替代方案

在正式截图完成前，可以使用以下临时方案：

### 方案 1: 使用占位图

创建简单的占位图片，标注图片说明：

```bash
# 使用在线工具生成占位图
# https://placeholder.com/
# 例如: https://via.placeholder.com/1200x600?text=DashScope+注册页面
```

---

### 方案 2: 使用文字说明

在文章中暂时移除图片引用，用详细的文字说明替代：

**原内容:**
```markdown
![注册页面](../images/dashscope-register.png)
```

**临时改为:**
```markdown
> 📷 **截图位置**: DashScope 注册页面
> 
> **说明**: 显示手机号输入框、验证码按钮、密码设置等元素
```

---

### 方案 3: 使用代码块展示关键信息

对于 API 响应等内容，直接用代码块展示：

**原内容:**
```markdown
![Postman 测试](../images/postman-test.png)
```

**临时改为:**
```markdown
**Postman 配置:**

URL: `https://dashscope.aliyuncs.com/api/v1/services/aigc/text-generation/generation`

Headers:
- Authorization: Bearer sk-your-api-key
- Content-Type: application/json

Body (JSON):
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

响应示例:
```json
{
  "output": {
    "text": "Spring Boot 是一个用于简化 Spring 应用开发的框架..."
  },
  "usage": {
    "total_tokens": 156
  }
}
```
```

---

## ✅ 图片补充检查清单

完成后请勾选：

- [ ] dashscope-register.png
- [ ] dashscope-activate.png
- [ ] dashscope-create-key.png
- [ ] postman-test.png
- [ ] attu-ui.png
- [ ] spring-initializr.png
- [ ] upload-document.png

---

## 🛠️ 截图工具推荐

### Windows
- **Snipping Tool**（系统自带）
- **ShareX**（免费开源，功能强大）
- **Snipaste**（免费，支持贴图）

### macOS
- **Screenshot**（Command + Shift + 4）
- **CleanShot X**（付费，功能丰富）

### Linux
- **Flameshot**（免费开源）
- **Shutter**（免费开源）

### 跨平台
- **OBS Studio**（录屏+截图）
- **Greenshot**（免费开源）

---

## 📤 图片存放位置

所有图片应存放在：

```
D:\project\zhilonglee.github.io\articles\images\
```

确保目录结构：

```
articles/
├── images/
│   ├── dashscope-register.png
│   ├── dashscope-activate.png
│   ├── dashscope-create-key.png
│   ├── postman-test.png
│   ├── attu-ui.png
│   ├── spring-initializr.png
│   └── upload-document.png
├── java/
│   └── java-ai-first-rag-demo.md
└── ...
```

---

## 💡 截图技巧

### 1. 突出关键区域
- 使用红框或箭头标注重点
- 放大关键按钮或输入框

### 2. 保持一致性
- 所有截图使用相同的缩放比例
- 保持相同的主题（浅色或深色）

### 3. 清理干扰元素
- 关闭不必要的浏览器标签
- 隐藏书签栏
- 清理桌面图标

### 4. 添加说明文字
- 使用工具添加标注
- 说明每个步骤的操作

---

## 🎯 优先级建议

如果时间有限，按以下优先级补充截图：

**高优先级（必须）:**
1. ✅ postman-test.png（展示 API 调用）
2. ✅ upload-document.png（展示文档上传）

**中优先级（建议）:**
3. dashscope-create-key.png（展示 API Key 创建）
4. spring-initializr.png（展示项目创建）

**低优先级（可选）:**
5. dashscope-register.png（可用文字说明）
6. dashscope-activate.png（可用文字说明）
7. attu-ui.png（可选功能）

---

*最后更新: 2026-05-14*
