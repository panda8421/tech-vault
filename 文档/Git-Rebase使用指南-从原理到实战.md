# Git Rebase 使用指南：把提交历史"拉直"的魔法，附一次真实变基实录

> 前言：同样是合并分支，`merge` 是"原地打结"，`rebase` 是"拆了重排"。本文先用一张图、几句话讲清 rebase 的本质，然后用我最近一次**真实的生产实操**（把功能分支变基到最新的 `develop/machine_S115`）带你走一遍完整流程——包括为什么最后要用 `--force-with-lease` 强推。

<!-- more -->

## 目录

- [一、一句话理解 Rebase（变基）](#一一句话理解-rebase变基)
- [二、一张图看懂：merge vs rebase](#二一张图看懂merge-vs-rebase)
- [三、实战案例：一次完整的变基实录](#三实战案例一次完整的变基实录)
- [四、终端输出逐行解读](#四终端输出逐行解读)
- [五、进阶：交互式 rebase 整理提交](#五进阶交互式-rebase-整理提交)
- [六、黄金法则：别动公共历史](#六黄金法则别动公共历史)
- [七、总结](#七总结)

---

## 一、一句话理解 Rebase（变基）

你的功能分支是从 `main` 某个旧点上"长"出来的。**rebase 就是把你的提交一个个摘下来，接到目标分支的最新顶端重新提交一遍**——基底变了，所以叫"变基"。

它和 merge 的本质区别：

| | merge | rebase |
|---|---|---|
| 做法 | 两个分支原地汇合，产生一个"合并提交" | 摘下你的提交，重放到目标分支最新处 |
| 历史 | 保留分叉的真实形状 | 重写成一条直线 |
| 提交 hash | 不变 | **全部改变**（重放生成的都是新提交） |
| 适用 | 公共分支合并 | 私有分支同步主干 |

一句话：**rebase 后你的分支，看起来就像你从最新代码上一次开发出来的。**

---

## 二、一张图看懂：merge vs rebase

```text
【merge：原地汇合】

main    C1 ── C2 ────────── C3 ──── M
               \                  ╱
feature         F1 ── F2 ────────
                             M = 合并提交（有两个父节点）
历史保留分叉的真实形状


【rebase：摘下重放】

main    C1 ── C2 ── C3 ── F1' ── F2'
                 ╲
                  F1 ── F2   （虚线：旧提交被重写丢弃）
你的提交被搬到 main 顶端，历史变成一条直线
```

merge 像把两条绳子打了个结；rebase 像把你的绳子拆下来，接在主绳末端——事后看全程只有一条绳。

---

## 三、实战案例：一次完整的变基实录

### 3.1 想要干嘛？（真实场景）

我手上两个分支：

- **`develop/machine_S115`**：同事维护的主开发分支，一直有新更新
- **`feature/emerge_stop_secon_recover`**：我的功能分支，但基于旧代码

**需求**：继续在 feature 分支上开发，但又不想丢掉 S115 的新更新。与其等最后合并时面对一堆冲突，不如现在就把 feature 分支变基到云端最新的 S115 上。

### 3.2 动手前的自查

1. **基准选云端的 `origin/develop/machine_S115`**——不用先切到它，fetch 之后远程镜像就是最新，少两次分支切换
2. **`git status` 确认工作区干净**——有未提交改动先 commit 或 stash
3. **心里有数：完成后推送要 `--force-with-lease`**——因为 rebase 会重写历史

### 3.3 完整操作（终端实录）

```bash
# 0) 自查：工作区必须干净
$ git status
On branch feature/emerge_stop_secon_recover
Your branch is up to date with 'origin/feature/emerge_stop_secon_recover'.
nothing to commit, working tree clean

# 1) 一步同步云端所有分支（不用切分支、不动本地代码）
$ git fetch origin

# 2) 变基：把我的提交重放到云端 S115 最新处
$ git rebase origin/develop/machine_S115
Successfully rebased and updated refs/heads/feature/emerge_stop_secon_recover.

# 3) 历史被重写，普通 push 会被拒——用安全强推
$ git push --force-with-lease origin feature/emerge_stop_secon_recover
 + 2f36c33...adcf74c feature/emerge_stop_secon_recover -> feature/emerge_stop_secon_recover (forced update)
```

就这三条命令，一次成功、零冲突，收工。

---

## 四、终端输出逐行解读

| 终端输出 | 怎么读它 |
|---|---|
| `nothing to commit, working tree clean` | 没有未提交改动，**变基前提满足**，可以动手 |
| `git fetch origin` 无任何输出 | 正常！fetch 静默完成，所有 `origin/xxx` 镜像已更新到云端最新 |
| `Successfully rebased and updated ...` | 一次成功、**零冲突**——历史已重写成"一条直线" |
| `+ 2f36c33...adcf74c ... (forced update)` | 强推成功，`+` 和 `forced update` 表明云端指针被移到新历史上 |
| `remote: Create pull request ...` | Bitbucket 主动给出 PR 链接——合并回 S115 时点它走正常审查 |

### 4.1 为什么用 --force-with-lease 而不是 --force？

- `--force` 是**无脑覆盖**：云端有别人的新提交也会被抹掉，同事的成果直接消失；
- `--force-with-lease` 会先检查"云端还是我上次看到的那个版本吗"，**不是就拒绝推送**。

本次变基后云端 feature 分支只有我自己在用，所以它能安全通过。养成习惯：**能用 `--force-with-lease` 就永远别用 `--force`**。

### 4.2 收尾必做：编译验证

变基后的代码是"最新基底 + 我的改动"的混合体，**接口变更、宏被删**这类问题只有编译能暴露。所以我最后一步是立刻编译一遍固件确认通过，再点 remote 给出的 PR 链接发起合并请求。

> 这一步经常被忽略：rebase "成功"只代表 git 层面无冲突，不代表代码逻辑层面兼容。尤其是嵌入式/MCU 项目，基底更新后外设驱动接口变了是常事。

---

## 五、进阶：交互式 rebase 整理提交

除了同步主干，rebase 的另一个高频用法是**整理自己的提交历史**（合并碎提交、改提交信息、删废提交）：

```bash
# 整理最近 3 个提交
git rebase -i HEAD~3
```

打开编辑器后会看到提交清单，把想做的操作写进每行行首：

| 指令 | 作用 | 一句话类比 |
|---|---|---|
| `pick` | 保留该提交（默认） | 原样通过 |
| `reword` | 保留提交，但修改提交信息 | 改简历措辞 |
| `squash` | 并入上一个提交，信息合并保留 | 两件事并成一件 |
| `fixup` | 并入上一个提交，丢弃本条信息 | 默默合并 |
| `drop` | 直接删除该提交 | 这段黑历史不存在 |
| `edit` | 停在该提交，方便修改内容 | 暂停检查 |

重放途中遇到冲突的标准三步（会循环出现）：

```bash
# 1) 打开标了 conflict 的文件，手动改成想要的样子
# 2) 标记已解决
git add 文件名
# 3) 继续重放下一个
git rebase --continue

# 后悔药：一键回到 rebase 之前
git rebase --abort
```

rebase 是**一段一段重放**的，所以冲突可能不止出现一次——耐心循环到它说 `Successfully rebased`。

---

## 六、黄金法则：别动公共历史

rebase 会**重写提交历史**。如果对已经 push 到公共分支（如 `main`、共享分支）的提交做 rebase，别人 pull 下来会撞出大量重复冲突，团队历史直接乱套。

> **历史一旦公开，就是团队契约——公共分支用 merge，私有分支随便 rebase。**

记住这个分工，你就不会踩坑：

- ✅ 自己的 feature 分支：变基同步主干、整理提交，随意
- ✅ 提 PR 前把自己的分支 rebase 到最新：审阅者看到的是干净的 diff
- ❌ 别人的分支、已发布的 release 分支：碰都不要碰

---

## 七、总结

回到开头的案例，整个流程浓缩成三句话：

1. **同步**：`git fetch origin` 一步拿到云端所有分支最新状态，不用切分支
2. **变基**：`git rebase origin/develop/machine_S115` 把自己的提交重放到最新主干上
3. **强推**：历史重写后用 `git push --force-with-lease` 安全更新云端，最后编译验证

选型口诀：**"公共分支 merge，私有分支 rebase；强推只认 --force-with-lease；rebase 完必须编译。"**

---

*本文案例来自笔者在 MCU 固件项目中的真实操作记录，终端输出均为原文。如果对你有帮助，欢迎点赞收藏～*
