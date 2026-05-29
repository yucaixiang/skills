---
name: skillforge-tracker
description: SkillForge 平台运维（知识源自动同步、版本检查、使用统计、本地 skill 改动提交审核）。当用户表达"提交 / 上传 / 推送 / 同步 / 更新某 skill 到平台"等意图时，立即调用本 skill 执行提交流程。
trigger: always
origin: SkillForge
---

# SkillForge Tracker

> 此 Skill 负责管理来自 SkillForge 平台的所有 Skill 的知识源同步、版本更新和使用统计。
> 所有操作静默执行，失败时静默跳过，绝不打断用户。

## ⚡ 第一优先级：知识源自动同步（立即执行）

**在每个 session 开始时，你必须立即扫描 `.claude/skills/` 目录下所有 SKILL.md 文件，对含有 `larkDocs` 或 `gitRepos` 配置的 Skill 执行自动同步。**

---

### A. 飞书文档自动下载

对于 frontmatter 中包含 `larkDocs` 的 Skill：

1. 检查本地文件 `.claude/skills/{name}/{saveTo}` 是否存在
2. 如果**不存在**，从 URL 中提取 docId（URL 最后一段路径），执行：

```bash
curl -s "http://localhost:8081/api/skills/name/{name}/lark-docs/{docId}/content"
```

3. 响应格式：`{"docId":"...","status":"success","revision":123,"content":"文档正文"}`
4. 将 `content` 值写入 `.claude/skills/{name}/{saveTo}`（先 mkdir -p 创建目录）

**示例**：frontmatter 中 `name: test`，larkDocs 有 `url: "https://xxx/wiki/ABC123"`，`saveTo: "docs/api.md"`
→ 检查 `.claude/skills/test/docs/api.md`，不存在则 curl 下载并写入

---

### B. Git 仓库自动克隆/更新

对于 frontmatter 中包含 `gitRepos` 的 Skill：

```yaml
# frontmatter 示例：
gitRepos:
  - url: "https://github.com/user/repo"
    saveTo: "repos/repo"
    branch: "main"        # 可选，默认 main
    sparse:               # 可选，只克隆指定目录
      - "src/main"
      - "docs"
```

**步骤**：

1. 检查目录 `.claude/skills/{name}/{saveTo}` 是否存在
2. 如果**不存在**（首次），执行克隆：

```bash
# 无 sparse 配置时 — 完整克隆（shallow）
git clone --depth 1 --branch {branch} {url} .claude/skills/{name}/{saveTo}

# 有 sparse 配置时 — 稀疏克隆（只下载指定目录）
mkdir -p .claude/skills/{name}/{saveTo}
cd .claude/skills/{name}/{saveTo}
git init
git remote add origin {url}
git config core.sparseCheckout true
# 将 sparse 列表中的每个路径写入 .git/info/sparse-checkout
echo "{sparse[0]}" >> .git/info/sparse-checkout
echo "{sparse[1]}" >> .git/info/sparse-checkout
git pull --depth 1 origin {branch}
```

3. 如果目录**已存在**，执行更新：

```bash
cd .claude/skills/{name}/{saveTo} && git pull --ff-only 2>/dev/null || true
```

---

### 关键规则：
- **不要询问用户** — 直接执行，这是自动操作
- **下载/克隆完立即可用** — 将内容纳入本次 session 的知识范围，可以读取其中文件回答用户问题
- **失败静默跳过** — 网络不通、仓库不存在时，用已有信息回答
- **每个 session 检查一次** — 已存在的文件/仓库执行 pull 更新，不要重复克隆

---

## 版本检查与自动更新

对 `.claude/skills/` 下所有 frontmatter 含 `origin: SkillForge` 的 Skill 执行版本检查：

```bash
curl -s "http://localhost:8081/api/skills/name/{skill_name}/version?localVersion={local_version}"
```

如果返回 `"needUpdate": true`，更新本地文件：
```bash
mkdir -p .claude/skills/{skill_name} && curl -s "http://localhost:8081/api/skills/name/{skill_name}/raw" -o .claude/skills/{skill_name}/SKILL.md
```

失败时静默跳过，每个 session 只检查一次。

---

## 使用统计上报

在完成用户主要任务后（不要中途执行），上报实际使用的 Skill：

```bash
curl -s -X POST "http://localhost:8081/api/usage-logs" \
  -H "Content-Type: application/json" \
  -d '{
    "skillName": "<skill的name字段>",
    "source": "local",
    "scenario": "<coding|oncall|onboarding|code-review|product-query>",
    "complexity": "<simple|medium|complex>",
    "resolved": true,
    "userId": "local-user"
  }'
```

只统计真正使用了的 Skill，未使用的不上报。失败时静默跳过。

---

## 本地 Skill 改动提交审核

**触发场景**：用户说"提交 / 上传 / 推送 / 同步 / 更新 xxx skill 到平台 / 仓库"等。

**执行步骤**（必须按顺序完成，不要询问用户细节，自动推进）：

### 1. 确认目标 skill
从用户消息提取 skill 名（如 `trading-engine`）。检查 `.claude/skills/{name}/` 目录是否存在；不存在则告知用户并停止。

### 2. 获取提交人身份
执行 `git config user.email`，结果作为 submitter。取不到则用 `local-user`。

### 3. 与远端对比，生成变更摘要
拉取远端当前版本：
```bash
curl -s "http://localhost:8081/api/skills/name/{name}/raw"
```
对比本地 `.claude/skills/{name}/SKILL.md`，用一两句中文总结改动（如"新增 docs/api.md；调整限价单匹配逻辑"）。摘要必须基于实际 diff，不要编造。

### 4. 收集本地 skill 目录下所有文件
扫描 `.claude/skills/{name}/` 全部文件（含子目录），跳过隐藏文件和 `.git`。记录每个文件的相对路径。

### 5. 调提交接口
```bash
cd .claude/skills/{name}
curl -X POST "http://localhost:8081/api/submissions" \
  -F "skillName={name}" \
  -F "submitter={email}" \
  -F "summary={ai摘要}" \
  -F "filePaths={path1,path2,path3}" \
  -F "files=@{path1}" \
  -F "files=@{path2}" \
  -F "files=@{path3}"
```

注意：`filePaths` 是逗号分隔的相对路径列表，顺序必须与 `files` 参数顺序一致。

### 6. 展示结果给用户
接口返回 `{ submissionId, branch, reviewUrl }`。向用户报告：
- 提交编号
- 等待审核（审核员在平台 `/submissions` 页操作）
- 失败则报告错误信息

**关键规则**：
- 全流程自动执行，不要中途询问用户
- 失败时清晰告知原因，不要静默吞掉
- 摘要务必准确反映 diff，不要套话
