# 小下同学的博客

基于 Hexo 6.3.0 和 Butterfly 主题构建的个人博客站点。

## 博客信息

- 站点名称: 小下同学
- 副标题: 找不到北,就多看东西
- 作者: 安小下同学
- 博客地址: https://blog.linganmin.cn

## 技术栈

- **静态站点生成器**: Hexo 6.3.0
- **主题**: Butterfly 4.5.1
- **部署方式**: GitHub Pages (linganmin.github.io)
- **插件**:
  - hexo-abbrlink: 永久链接生成
  - hexo-wordcount: 字数统计
  - hexo-deployer-git: Git 部署
  - hexo-butterfly-extjs: Butterfly 主题扩展

## 快速开始

### 安装依赖

```bash
npm install
```

### 本地预览

```bash
npm run server
```

访问 http://localhost:4000 查看博客

### 新建文章

```bash
hexo new "文章标题"
```

### 生成静态文件

```bash
npm run build
```

### 清理缓存

```bash
npm run clean
```

### 部署到 GitHub Pages

```bash
npm run deploy
```

## 项目结构

```
.
├── _config.yml              # Hexo 主配置文件
├── _config.butterfly.yml    # Butterfly 主题配置
├── source/                  # 源文件目录
│   ├── _posts/             # 文章目录
│   ├── _data/              # 数据文件
│   ├── categories/         # 分类页面
│   └── tags/               # 标签页面
├── themes/                  # 主题目录
├── scaffolds/              # 文章模板
└── public/                 # 生成的静态文件(git忽略)
```

## 配置说明

- 文章永久链接格式: `posts/:abbrlink/` (使用 crc32 算法生成)
- 时区: Asia/Shanghai
- 语言: 简体中文
- 每页显示: 10 篇文章

## 许可

Private
