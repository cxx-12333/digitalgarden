---
{"dg-publish": true, "permalink": "/Blog/git-pr-parallel-history-trap/", "title": "为什么 PR 里出现了别人早就合并过的提交——Git compare 的祖先真相", "tags": ["git", "工程实践"], "created": "2026-08-20", "updated": "2026-08-20", "dg-note-properties": {"title": "为什么 PR 里出现了别人早就合并过的提交——Git compare 的祖先真相", "permalink": "git-pr-parallel-history-trap", "created": "2026-08-20", "updated": "2026-08-20", "tags": ["git", "工程实践"], "type": "blog"}}
---



# 为什么 PR 里出现了别人早就合并过的提交

## 现象

团队里常见的一幕：dev 分支开发完两个提交，想合进 test。打开 PR 一看——**提交列表里躺着一长串早就进过 test 的历史提交**，review 页面脏得没法看，reviewer 一头雾水："这些不是都合过了吗？"

## 原理：PR 列的不是"内容差异"，是"祖先关系"

GitHub/GitLab 的 PR 提交列表，计算的是：

> **compare 分支上有、而 base 分支历史里没有的提交**——按 commit hash 的祖先关系判断，**不看内容**。

关键在"历史里没有"这四个字。看这个翻车模型：

```
          A---B---C---D   ← dev（A/B 是老提交，C/D 是你的新工作）
         /
----E---F---G---H----     ← test（G 里手工"复制"了 A/B 的改动重新提交）
```

如果 test 上的 G 不是通过 merge dev 得来，而是**把 A/B 的代码手工复制过来重新提交**（新 hash），那么在 Git 眼里：

- A、B 的**内容**在 test 里存在 ✓
- 但 A、B 这两个 **commit hash** 不在 test 的祖先链上 ✗

于是 `PR(test ← dev)` 的提交列表 = A、B、C、D **全部**——尽管 A/B 的功能 test 早就有了。**平行历史让 Git 的"是否已有"判断永久失明。**

这不是 bug，是 Git 的设计哲学：commit 是内容+历史的原子单元，内容相同但 hash 不同就是"不同的提交"。

## 解法：cherry-pick 精确摘取（已验证）

当场最快的洗法——把"想带的提交"摘到一条干净的新分支上再发 PR：

```bash
git checkout -b feature/xyz origin/test   # 从 test（base）拉新分支
git cherry-pick <C的hash> <D的hash>        # 只摘你的两个提交
git push -u origin feature/xyz
```

PR(test ← feature/xyz) 的提交列表 = **恰好那 2 个提交** ✓。因为新分支直接基于 test，你摘的提交是它祖先链上唯一"没有"的东西。

代价：cherry-pick 生成新 hash，dev 上的 C/D 依然游离——**这是止血不是治病**。

## 长期预防（三条纪律）

1. **统一同步方式**：要么 test 定期 `git merge dev`（一次 merge 提交，历史收敛），要么所有进 test 的改动一律走 PR。**最忌"手工把代码复制到另一边重新提交"**——那会让平行历史永久化，以后每个 PR 都越来越脏；
2. **发 PR 前预检**：`git log --oneline origin/test..dev` 看一眼"真实会带过去什么"，脏了先治再发；
3. **保护发布分支的流程**：test 若是发布分支，只允许 release PR 更新（分支保护开了，流程约定再补这一条）。

## 一句话

> PR 的提交列表问的是"这些 hash 在不在 base 的家谱里"，不是"这些改动 base 有没有"。让家谱收敛（merge/PR 单通道），别让同样的改动以不同 hash 在两条平行线上各自繁衍。

---

*关联：[文件即接口：agent自动化的调度解耦模式](Notes/文件即接口：agent自动化的调度解耦模式/)——同样的思想：契约看"身份"不看"内容"。*
