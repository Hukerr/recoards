---
title: '从零搭建个人博客：Hexo + Vercel + 桌面发布工具'
date: 2026-03-13 11:38:14
tags: [博客, Hexo, Vercel, Electron]
categories: [折腾记录]
---

今天花了几个小时，从零搭了一个完整的个人博客系统——不只是能跑起来，还顺带做了一个好看的桌面写作工具，写完一键发布。这篇文章把整个过程记下来，给同样想折腾的人参考。

<!-- more -->

## 整体方案

```
写文章（Electron 桌面工具）
  → 保存为 .md 文件
  → git push 到 GitHub
  → Vercel 自动检测更新
  → 部署上线
```

三个核心工具：

| 工具 | 作用 |
|------|------|
| **Hexo** | 静态博客框架，把 Markdown 渲染成网页 |
| **GitHub** | 代码托管，Vercel 从这里拉代码 |
| **Vercel** | 自动部署，push 之后几秒就更新 |

---

## 一、安装 Hexo

先确保电脑装了 [Node.js](https://nodejs.org)（建议 18+），然后：

```bash
npm install -g hexo-cli
```

初始化博客项目：

```bash
hexo init my-blog
cd my-blog
npm install
```

本地预览：

```bash
hexo server
```

打开 `http://localhost:4000` 就能看到默认博客页面。

### 目录结构

```
my-blog/
├── source/
│   └── _posts/     ← 文章放这里，.md 格式
├── themes/         ← 主题
├── _config.yml     ← 全局配置
└── package.json
```

---

## 二、推送到 GitHub

在 GitHub 新建一个仓库（比如 `my-blog`），然后：

```bash
cd my-blog
git init
git add .
git commit -m "init blog"
git remote add origin https://github.com/你的用户名/my-blog.git
git push -u origin main
```

---

## 三、部署到 Vercel

1. 打开 [vercel.com](https://vercel.com)，用 GitHub 账号登录
2. 点击 **Add New Project**，选择刚才的 `my-blog` 仓库
3. Framework Preset 选 **Other**（Hexo 不在列表里，但 Vercel 能自动识别）
4. Build Command 填：`npx hexo generate`
5. Output Directory 填：`public`
6. 点 **Deploy**

部署完成后 Vercel 会给你一个域名，比如 `your-blog.vercel.app`，博客就上线了。

之后每次 `git push`，Vercel 会自动触发重新部署，几十秒就更新完毕。

---

## 四、写文章的正确姿势

Hexo 文章是 Markdown 格式，放在 `source/_posts/` 目录下，头部需要写 front matter：

```markdown
---
title: '我的第一篇文章'
date: 2026-03-13 12:00:00
tags: [随笔, 生活]
categories: [日记]
---

正文内容从这里开始...
```

手动操作流程：

```bash
# 写完文章之后
git add .
git commit -m "新增文章: 标题"
git push
```

这样每次发文章都要打命令行，有点麻烦——所以我又做了一个桌面工具。

---

## 五、用 Electron 做一个桌面发布工具

纯命令行发文章不够直观，我用 Electron 做了一个桌面 App：

- **左侧**：文章列表，点击切换
- **右侧**：分屏编辑器，左写 Markdown，右边实时预览
- **顶部**：填写标题、标签、分类
- **底部**：一键发布按钮

点击「发布文章」之后，程序自动：

1. 把内容写入 `source/_posts/标题.md`
2. 执行 `git add .`
3. 执行 `git commit -m "新增文章: 标题"`
4. 执行 `git push`

然后 Vercel 自动部署，几十秒后文章上线。

### 核心代码片段（发布逻辑）

```javascript
// main.js - Electron 主进程
ipcMain.handle('publish-post', async (_, { filename, title, tags, categories, body }) => {
  // 生成 front matter
  const date = new Date().toISOString().slice(0, 19).replace('T', ' ');
  const content = `---\ntitle: '${title}'\ndate: ${date}\ntags: [${tags.join(', ')}]\n---\n\n${body}`;

  // 写文件
  fs.writeFileSync(path.join(POSTS_DIR, filename), content, 'utf-8');

  // git 操作
  await execAsync('git add .', { cwd: BLOG_DIR });
  await execAsync(`git commit -m "新增文章: ${title}"`, { cwd: BLOG_DIR });
  await execAsync('git push', { cwd: BLOG_DIR });

  return { ok: true };
});
```

界面用的暗黑风格，主色调是暗红 + 深蓝，发布成功后有粒子爆破动效。

打包成单个 `.exe`，放在桌面，双击就能用，不需要装任何环境。

---

## 六、踩过的坑

**1. Vercel 部署后页面空白**

检查 Output Directory 是否填的 `public`，Hexo 生成的静态文件在这个目录。

**2. git push 每次都要输密码**

配置 SSH key，或者用 Git Credential Manager：

```bash
git config --global credential.helper manager
```

**3. 文章中文文件名乱码**

文章文件名建议用英文或拼音，front matter 里的 `title` 可以是中文，不影响访问。

**4. Electron 打包体积大**

portable 模式打出来大概 150MB，因为 Electron 本身就包含了一个完整的 Chromium。接受不了可以用 Web 版替代，或者用更轻量的 Tauri（但需要 Rust 环境）。

---

## 总结

整套流程搭完之后，写博客的体验非常流畅：

1. 打开桌面的 `BlogPublisher.exe`
2. 点「新建」，写文章，实时看预览
3. 填好标题和标签，点「发布文章」
4. 等十几秒，博客更新完毕

从想法到发布，全程不需要碰命令行。

如果你也想搭一个，整个过程大概需要一两个小时，Hexo 和 Vercel 都有免费额度，完全零成本。