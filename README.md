# blog-img

个人图床仓库，用于托管 [bitlog](https://github.com/zephyr110/bitlog) 博客文章中引用的媒体资源（截图、动图等）。

## 用途

- 为 [zephyr110/bitlog](https://github.com/zephyr110/bitlog) 项目提供图片托管
- 通过 GitHub CDN（raw.githubusercontent.com / jsdelivr）访问

## 使用方式

在 Markdown 中引用：

```markdown
# raw GitHub（直链）
![](https://raw.githubusercontent.com/zephyr110/blog-img/main/example.png)

# jsdelivr CDN（更快，推荐）
![](https://cdn.jsdelivr.net/gh/zephyr110/blog-img@main/example.png)
```

## 目录

| 类型 | 说明 |
|------|------|
| `*.png` | 截图、插图 |
| `*.gif` | 动图演示 |
| `*.webp` | WebP 格式图片 |
| `RECOVERY.md` | [Git force push 数据恢复指南](./RECOVERY.md) |
