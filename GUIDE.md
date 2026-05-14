# Docsify 个人网站使用指南

## 📖 项目说明

这是一个基于 [Docsify](https://docsify.js.org/) 搭建的个人网站,通过编写 Markdown 文件来分享文章。

## 🚀 本地预览

在本地预览网站,可以使用以下方法:

### 方法一: 使用 VS Code Live Server 插件
1. 安装 Live Server 插件
2. 右键点击 `index.html`,选择 "Open with Live Server"

### 方法二: 使用 Python
```bash
# Python 3
python -m http.server 8080

# Python 2
python -m SimpleHTTPServer 8080
```

### 方法三: 使用 Node.js
```bash
npm install -g docsify-cli
docsify serve .
```

然后在浏览器中访问 `http://localhost:8080`

## 📝 如何添加新文章

### 1. 创建文章文件

在 `articles` 目录下创建新的 Markdown 文件,例如:
```
articles/
├── java/          # Java 相关
│   └── your-article.md
├── springboot/    # Spring Boot 相关
│   └── your-article.md
└── life/          # 生活随笔
    └── your-article.md
```

### 2. 更新侧边栏导航

编辑 `_sidebar.md` 文件,添加新文章的链接:

```markdown
- [你的文章标题](articles/分类/文件名.md)
```

### 3. 提交到 GitHub

```bash
git add .
git commit -m "添加新文章: 文章标题"
git push
```

GitHub Pages 会自动部署,几分钟后即可在线访问。

## 🎨 自定义配置

### 修改网站名称

编辑 `index.html`,修改 `name` 字段:
```javascript
window.$docsify = {
  name: '你的网站名称',
  // ...
}
```

### 修改主题

在 `index.html` 中更换 CSS 主题:
```html
<!-- Vue 主题 -->
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css">

<!-- Dark 主题 -->
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/dark.css">

<!-- Buble 主题 -->
<link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/buble.css">
```

### 添加更多代码高亮语言

在 `index.html` 中添加对应的 Prism 组件:
```html
<script src="//cdn.jsdelivr.net/npm/prismjs@1/components/prism-语言名.min.js"></script>
```

## 📌 Markdown 写作技巧

### 基本语法

- 使用 `#` 表示标题
- 使用 `-` 或 `*` 表示列表
- 使用 ``` 包裹代码块
- 使用 `**文字**` 加粗
- 使用 `*文字*` 斜体

### 插入图片

```markdown
![图片描述](图片路径)
```

### 插入链接

```markdown
[链接文本](URL)
```

### 代码块

```markdown
```java
public class Example {
    // 你的代码
}
```
```

## 🔗 相关链接

- [Docsify 官方文档](https://docsify.js.org/)
- [Markdown 语法指南](https://markdown.com.cn/)
- [GitHub Pages 文档](https://pages.github.com/)

---

祝写作愉快! ✍️
