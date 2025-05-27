# GitHub 仓库权限

---

我们可以随意 `clone` 和关联一个**Github的公开仓库**，但是不能随意 `push` 到别人仓库的**远程分支**，除非有特定的权限（是仓库的协作者/成员）

<br>

## 一、GitHub 权限机制

| 你能做什么？            | 是否需要权限？ | 说明                                             |
| ----------------------- | -------------- | ------------------------------------------------ |
| `git clone` 公共仓库    | ❌ 不需要       | 所有人都可以克隆公开项目                         |
| `git fetch` 公共仓库    | ❌ 不需要       | 拉取代码更新不需要权限                           |
| `git push` 到公共仓库   | ✅ 需要权限     | 只有**协作者（collaborator）**或成员才能推送代码 |
| 提交 Pull Request（PR） | ❌ 不需要       | Fork 后改代码可以向原项目发起 PR                 |

<br>

> 举个例子

你看到一个 GitHub 项目，比如：

```
https://github.com/someone/project
```

你可以：

```
git clone https://github.com/someone/project.git
cd project
```

然后本地改动代码、提交、切分支都没问题。

但是你不能：

```
git push origin main
```

GitHub 会提示类似：

```
remote: Permission to someone/project.git denied to your-username.
fatal: unable to access 'https://github.com/someone/project.git/': The requested URL returned error: 403
```

因为你**不是这个仓库的协作者**。

<br>

## 二、贡献代码的方式

你可以使用 GitHub 的 **Fork + Pull Request 工作流**

1. `Fork` 仓库

在 GitHub 上点击原仓库右上角的 **“Fork”** 按钮，把它复制一份到你自己的Github账号下，这时候你的Github中会有一个新的项目（即远程仓库）。

2. `Clone` 你的仓库**副本**

```
git clone https://github.com/你的用户名/项目名.git
```

3. 在你的**副本里**开发和提交

```
git checkout -b my-feature
# 改代码
git add .
git commit -m "添加一个功能"
git push origin my-feature
```

4. 发起 **Pull Request（PR）**

回到 GitHub 页面，点击 “Compare & pull request”，发起 PR 到原始项目。

> 只有原项目的维护者有权决定是否合并你的 PR