# Pantheooon Blog

欢迎访问本项目！这是 Pantheooon 的个人博客网站仓库，基于 [GitHub Pages](https://pages.github.com/) 搭建，实现静态网页托管与个人内容展示。

## 项目简介

本项目旨在分享我的学习笔记、技术文章、生活随笔等内容。网站采用静态页面生成方式，便于部署和维护。

## 功能特性

- 个人博客主页
- 技术文章与学习资料分享
- 响应式网页设计，适配多终端
- 维护简单，支持 Markdown 编辑

## 快速开始

1. **克隆仓库**

   ```bash
   git clone https://github.com/Pantheooon/pantheooon.github.io.git
   ```
   
2. **本地预览（可选）**

   建议安装 [Jekyll](https://jekyllrb.com/) 或其他静态网站生成器进行本地预览：

   ```bash
   gem install bundler jekyll
   cd pantheooon.github.io
   bundle install
   bundle exec jekyll serve
   ```
   然后访问 `http://localhost:4000` 预览效果。

3. **发布内容**

   直接通过提交 `.md` 文件或网页源代码，推送到 main 分支，GitHub Pages 会自动部署。

## 目录结构

```
.
├── _posts/            # 文章与博客内容（Markdown 格式）
├── _layouts/          # 页面模板
├── assets/            # 图片、CSS、JS 等静态资源
├── index.html         # 博客首页
└── ...                # 其他辅助文件
```

## 自定义与扩展

- 可根据需要自定义主题、样式与页面结构
- 推荐使用 Jekyll、Hexo 等主流静态博客框架

## 联系方式

欢迎关注与交流！  
- GitHub: [Pantheooon](https://github.com/Pantheooon)

## License

本项目内容遵循 MIT License 开源协议，欢迎 fork 和贡献代码。
