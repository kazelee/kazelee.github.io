# Kazelee - BreezeLee's Blog

基于 [Astro](https://astro.build) & [Fuwari](https://github.com/saicaca/fuwari) 的个人静态博客网站。

主题配置参考 [Water Space - Fall In The Water](https://www.waterwater.moe/) 和 [时歌的博客 - 理解以真实为本，但真实本身并不会自动呈现](https://www.lapis.cafe/)

<!-- 吐槽：只有 Water Space 的博客网站可以实现移动端主题不丢失，Fuwari 官方和时歌的博客都有主题丢失的问题。但鉴于 Water Space 自定义页面取消了 TOC 和分类，尝试修改时出现很多问题难以修复，故最终还是选择放弃，不考虑网站对移动端的优化了。 -->

## ⚙️ 文章 Frontmatter

```yaml
---
title: My First Blog Post
published: 2023-09-09
description: This is the first post of my new Astro blog.
image: ./cover.jpg
tags: [Foo, Bar]
category: Front-end
draft: false
lang: jp      # 仅当文章语言与 `config.ts` 中的网站语言不同时需要设置
---
```

## 🧞 指令

下列指令均需要在项目根目录执行：

| Command                           | Action                            |
|:----------------------------------|:----------------------------------|
| `pnpm install` 并 `pnpm add sharp` | 安装依赖                              |
| `pnpm dev`                        | 在 `localhost:4321` 启动本地开发服务器      |
| `pnpm build`                      | 构建网站至 `./dist/`                   |
| `pnpm preview`                    | 本地预览已构建的网站                        |
| `pnpm new-post <filename>`        | 创建新文章                             |
| `pnpm astro ...`                  | 执行 `astro add`, `astro check` 等指令 |
| `pnpm astro --help`               | 显示 Astro CLI 帮助                   |
