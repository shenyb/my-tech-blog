---
title: '用 Cloudflare Pages 免费搭建个人技术博客'
description: '从零开始，使用 Astro + Cloudflare Pages 搭建一个免费、快速的个人技术博客站点'
pubDate: 2024-12-01
heroImage: ''
category: engineering
tags: ['Cloudflare', 'Astro', '部署']
---

## 为什么选择 Cloudflare Pages？

Cloudflare Pages 提供：

- **免费无限带宽** — 不像 Vercel/Netlify 有流量限制
- **全球 CDN** — 300+ 节点，国内访问速度也不错
- **自动部署** — Git push 自动构建发布
- **免费域名** — `yourname.pages.dev`
- **自定义域名** — 支持绑定自己的域名

## 技术选型

| 技术 | 作用 |
|------|------|
| Astro | 静态站点生成，零 JS 输出 |
| Cloudflare Pages | 托管 + CDN |
| GitHub | 代码仓库 + 触发部署 |

## 部署步骤

### 1. 初始化项目

```bash
npm create astro@latest my-blog -- --template blog
cd my-blog
npm install
```

### 2. 推送到 GitHub

```bash
git init
git add .
git commit -m "init: my tech blog"
git remote add origin https://github.com/yourname/my-blog.git
git push -u origin main
```

### 3. 连接 Cloudflare Pages

1. 登录 [dash.cloudflare.com](https://dash.cloudflare.com)
2. 进入 Workers & Pages → Create → Pages → Connect to Git
3. 选择你的 GitHub 仓库
4. 构建设置：
   - Framework: `Astro`
   - Build command: `npm run build`
   - Build output: `dist`
5. 点击 Save and Deploy

### 4. 等待部署完成

大约 1-2 分钟，你的站点就会上线在 `your-project.pages.dev`。

## 免费域名方案

除了自带的 `.pages.dev`，还可以申请免费域名：

- **us.kg** — 在 [register.us.kg](https://register.us.kg) 免费注册
- **eu.org** — 经典免费域名，审批较慢（1-2 周）
- **pp.ua** — 乌克兰免费域名

然后在 Cloudflare Pages 设置中添加自定义域名即可。

## 写新文章

在 `src/content/blog/` 下新建 `.md` 文件：

```markdown
---
title: '文章标题'
description: '文章描述'
pubDate: 2024-12-01
---

文章内容...
```

Push 到 GitHub，自动部署上线！🎉
