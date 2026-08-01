# Git Force Push 数据恢复指南

> 记录 `blog-img` 仓库被 `git push --force` 覆盖后，通过 jsdelivr CDN 缓存恢复全部数据的完整过程。

## 事故背景

- **仓库**: `zephyr110/blog-img`
- **时间**: 2026-07-25 14:52 UTC
- **原因**: SSH clone 超时导致空仓库 commit，随后 `git push --force` 覆盖远程
- **损失**: `main` 分支原有 121 个文件（2021-2025 年的图片 + README）被替换为 2 个新截图

```
+ fd01c89...085b16f main -> main (forced update)
```

- `fd01c89` — 原始 commit（121 个文件）
- `085b16f` — force push 后的 root-commit（仅 2 个文件）

## 恢复原理

### 为什么可以恢复

1. **GitHub 保留了孤儿对象**：force push 只改了 `refs/heads/main` 的指向，`fd01c89` 这个 commit 对象仍存在 GitHub 的 object store 中（仓库 size 9974 KB，当前文件仅 2.4 MB，剩余 ~7.6 MB 是旧数据）

2. **jsdelivr 按 commit hash 缓存，不按分支**：

   ```
   https://cdn.jsdelivr.net/gh/{user}/{repo}@{commit}/
   ```

   jsdelivr 在首次访问时就抓取了 `fd01c89` 的文件树并缓存。后续 GitHub 上的 force push 只影响 `main` 分支引用，不影响 CDN 已缓存的 hash 内容。

3. **GitHub API 404 ≠ 数据已删除**：

   | 查询方式 | 结果 | 原因 |
   |----------|------|------|
   | `GET /repos/{user}/{repo}/git/commits/fd01c89` | 404 | 孤儿 commit，不可通过 API 访问 |
   | `cdn.jsdelivr.net/gh/...@{commit}/` | 200 | 按 hash 缓存，不依赖 ref |
   | 仓库 `size` 字段 | 9974 KB | 对象仍存储在 GitHub |

### 为什么 GitHub API 返回 404 但 CDN 有数据

```
refs/heads/main → 085b16f → a6830e4   （当前可达链）
fd01c89                               （孤儿，无 ref 指向它）
```

GitHub API 沿 ref 可达链查找 commit，孤儿 commit 不在任何 ref 的可达路径上，因此返回 404。但 commit 对象本身仍在 object store 中，jsdelivr 可直接按 hash 寻址。

## 恢复步骤

### 1. 确认旧 commit hash

从 force push 输出或 GitHub Events API 获取：

```bash
# force push 的输出中直接显示
+ fd01c89...085b16f main -> main (forced update)
```

### 2. 检查 jsdelivr 是否有缓存

```bash
OLD_COMMIT="fd01c89"
REPO="zephyr110/blog-img"

# 访问 CDN 目录页
curl -s "https://cdn.jsdelivr.net/gh/${REPO}@${OLD_COMMIT}/"
# 返回 HTTP 200 + HTML 目录列表 → 缓存存在
# 返回 404 → 缓存不存在，尝试其他 CDN（见下文）
```

### 3. 提取文件列表

```bash
curl -s "https://cdn.jsdelivr.net/gh/${REPO}@${OLD_COMMIT}/" \
  | grep -oP 'href="/gh/'"${REPO}"'@'"${OLD_COMMIT}"'/[^"]*"' \
  | sed "s|href=\"/gh/${REPO}@${OLD_COMMIT}/||" \
  | sed 's/"$//' \
  | grep -v '^\.github/' \
  | sort -u > file-list.txt
```

### 4. 批量下载

```bash
RESTORE_DIR="/tmp/blog-img-restore"
mkdir -p "${RESTORE_DIR}" && cd "${RESTORE_DIR}"

while read file; do
  [ -z "$file" ] && continue
  dir=$(dirname "$file")
  [ "$dir" != "." ] && mkdir -p "$dir"
  echo "Downloading: $file"
  curl -sL "https://cdn.jsdelivr.net/gh/${REPO}@${OLD_COMMIT}/${file}" -o "$file"
done < file-list.txt
```

### 5. 推回仓库

```bash
cd "${RESTORE_DIR}"
git init
git remote add origin "https://github.com/${REPO}.git"
git pull origin main          # 拉取当前状态
git add -A
git commit -m "restore: recover files from force-pushed commit via CDN"
git push origin main
```

## 其他 CDN 缓存源

如果 jsdelivr 无缓存，按优先级尝试：

| CDN | URL 格式 | 备注 |
|-----|----------|------|
| **jsdelivr** | `cdn.jsdelivr.net/gh/{user}/{repo}@{hash}/` | ✅ 本次使用 |
| **Statically** | `cdn.statically.io/gh/{user}/{repo}/{hash}/` | jsdelivr 备选 |
| **GitHub Archive** | `github.com/{user}/{repo}/archive/{hash}.tar.gz` | 仅当 commit 可通过 web 访问 |
| **raw.githubusercontent.com** | `raw.githubusercontent.com/{user}/{repo}/{hash}/` | 同上 |

## 预防措施

### 即时措施

在 GitHub 仓库 Settings → Branches → Add branch protection rule：
- Branch name pattern: `main`
- ✅ **Lock branch** — 禁止任何人 force push

### 长期措施

```bash
# 本地禁用 force push 到 main（需在服务端配置）
git config --global push.default simple

# 使用 --force-with-lease 代替 --force（至少检查远程是否有未知的新 commit）
git push --force-with-lease origin main
# 比 --force 更安全：如果远程有你不知道的 commit，push 会被拒绝
```

### 检测 clone 是否成功

```bash
# clone 后立即验证
git clone <url> /tmp/repo && git -C /tmp/repo log --oneline -1
# 如果输出 "fatal: your current branch 'main' does not have any commits yet"
# 说明 clone 失败，不要继续操作
```

## 关键经验

1. **永远不要对 main 分支使用 `git push --force`**，除非你 100% 确定自己在做什么
2. **clone 之后检查 `.git/objects` 是否非空**，超时的 clone 可能只创建了 `.git` 目录但没有 fetch 到对象
3. **jsdelivr CDN 是 GitHub force push 的免费保险**——只要有人曾经通过 CDN 访问过那个 commit，数据就能恢复
4. **Git 的不可变性是最后一道防线**——commit 对象一旦创建就永远存在，直到 GitHub GC 清理（通常 90 天）
