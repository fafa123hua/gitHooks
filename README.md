# Git Hooks

我们想要在 git 操作时候进行一些定制化操作，比如在 git commit 时候检查一下提交内容是否合规、git push 时候检查一下资源文件大小等等，开发者和管理员们可以指定 git 在不同事件、不同动作下执行特定的脚本。这里简单实现其中的 commit-msg。

### Git Hooks 简介：

http://git-scm.com/book/en/v2在一书中将hooks划分为如下类型

- 客户端的 hook：此类 hook 在提交者（committer）的计算机上被调用执行。此类 hook 又分为如下几类：
  - 代码提交相关的工作流 hook：提交类 hook 作用在代码提交的动作前后，通常用于运行完整性检查、提交信息生成、信息内容验证等功能，也可以用来发送通知。
  - Email 相关工作流 hook：Email 类 hook 主要用于使用 Email 提交的代码补丁。像是 Linux 内核这样的项目是采用 Email 进行补丁提交的，就可以使用此类 hook。工作方式和提交类 hook 类似，而且项目维护者可以用此类 hook 直接完成打补丁的动作。
  - 其他类：包括代码合并、签出（check out）、rebase、重写（rewrite）、以及软件仓库的清理等工作。
- 服务器端 hook：此类 hook 作用在服务器端，一般用于接收推送，部署在项目的 git 仓库主干（main）所在的服务器上。Chacon 将服务器端 hook 分为两类：
  - 接受触发类：在服务器接收到一个推送之前或之后执行动作，前触发常用于检查，后触发常用于部署。
  - 更新：类似于前触发，不过更新类 hook 是以分支（branch）作为作用对象，在每一个分支更新通过之前执行代码。

#### hook 列表如下：

| Hook 名称          | 触发指令                         | 描述                                                                                                                                                                                                                                                                                            | 参数的个数与描述                                                                                                                                          |
| ------------------ | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| applypatch-msg     | `git am`                         | 可以编辑 commit 时提交的 message。通常用于验证或纠正补丁提交的信息以符合项目标准。                                                                                                                                                                                                              | (1) 包含预备 commit 信息的文件名                                                                                                                          |
| pre-applypatch     | `git am`                         | 虽然这个 hook 的名称是“打补丁前”，不过实际上的调用时机是打补丁之后、变更 commit 之前。如果以非 0 的状态退出，会导致变更成为 uncommitted 状态。可用于在实际进行 commit 之前检查代码树的状态。                                                                                                    | 无                                                                                                                                                        |
| post-applypatch    | `git am`                         | 本 hook 的调用时机是打补丁后、commit 完成提交后。因此，本 hook 无法用于取消进程，而主要用于通知。                                                                                                                                                                                               | 无                                                                                                                                                        |
| pre-commit         | `git commit`                     | 本 hook 的调用时机是在获取 commit message 之前。如果以非 0 的状态退出则会取消本次 commit。主要用于检查 commit 本身（而不是 message）                                                                                                                                                            | 无                                                                                                                                                        |
| prepare-commit-msg | `git commit`                     | 本 hook 的调用时机是在接收默认 commit message 之后、启动 commit message 编辑器之前。非 0 的返回结果会取消本次 commit。本 hook 可用于强制应用指定的 commit message。                                                                                                                             | 1. 包含 commit message 的文件名。2. commit message 的源（message、template、merge、squash 或 commit）。3. commit 的 SHA-1（在现有 commit 上操作的情况）。 |
| commit-msg         | `git commit`                     | 可用于在 message 提交之后修改 message 的内容或打回 message 不合格的 commit。非 0 的返回结果会取消本次 commit。                                                                                                                                                                                  | (1) 包含 message 内容的文件名。                                                                                                                           |
| post-commit        | `git commit`                     | 本 hook 在 commit 完成之后调用，因此无法用于打回 commit。主要用于通知。                                                                                                                                                                                                                         | 无                                                                                                                                                        |
| pre-rebase         | `git rebase`                     | 在执行 rebase 的时候调用，可用于中断不想要的 rebase。                                                                                                                                                                                                                                           | 1. 本次 fork 的上游。2. 被 rebase 的分支（如果 rebase 的是当前分支则没有此参数）                                                                          |
| post-checkout      | `git checkout` 和 `git clone`    | 更新工作树后调用 checkout 时调用，或者执行 git clone 后调用。主要用于验证环境、显示变更、配置环境。                                                                                                                                                                                             | 1. 之前的 HEAD 的 ref。 2. 新 HEAD 的 ref。 3. 一个标签，表示其是一次 branch checkout 还是 file checkout。                                                |
| post-merge         | `git merge` 或 `git pull`        | 合并后调用，无法用于取消合并。可用于进行权限操作等 git 无法执行的动作。                                                                                                                                                                                                                         | (1) 一个标签，表示是否是一次标注为 squash 的 merge。                                                                                                      |
| pre-push           | `git push`                       | 在往远程 push 之前调用。本 hook 除了携带参数之外，还同时给 stdin 输入了如下信息：” ”（每项之间有空格）。这些信息可以用来做一些检查，比如说，如果本地（local）sha1 为 40 个零，则本次 push 是一个删除操作；如果远程（remote）sha1 是 40 个零，则是一个新的分支。非 0 的返回结果会取消本次 push。 | 1. 远程目标的名称。 2. 远程目标的位置。                                                                                                                   |
| pre-receive        | 远程 repo 进行`git-receive-pack` | 本 hook 在远程 repo 更新刚被 push 的 ref 之前调用。非 0 的返回结果会中断本次进程。本 hook 虽然不携带参数，但是会给 stdin 输入如下信息：” ”。                                                                                                                                                    | 无                                                                                                                                                        |
| update             | 远程 repo 进行`git-receive-pack` | 本 hook 在远程 repo 每一次 ref 被 push 的时候调用（而不是每一次 push）。可以用于满足“所有的 commit 只能快进”这样的需求。                                                                                                                                                                        | 1. 被更新的 ref 名称。2. 老的对象名称。3. 新的对象名称。                                                                                                  |
| post-receive       | 远程 repo 进行`git-receive-pack` | 本 hook 在远程 repo 上所有 ref 被更新后，push 操作的时候调用。本 hook 不携带参数，但可以从 stdin 接收信息，接收格式为” ”。因为 hook 的调用在更新之后进行，因此无法用于终止进程。                                                                                                                | 无                                                                                                                                                        |
| post-update        | 远程 repo 进行`git-receive-pack` | 本 hook 仅在所有的 ref 被 push 之后执行一次。它与 post-receive 很像，但是不接收旧值与新值。主要用于通知。                                                                                                                                                                                       | 每个被 push 的 repo 都会生成一个参数，参数内容是 ref 的名称                                                                                               |
| pre-auto-gc        | `git gc –auto`                   | 用于在自动清理 repo 之前做一些检查。                                                                                                                                                                                                                                                            | 无                                                                                                                                                        |
| post-rewrite       | `git commit –amend`,`git-rebase` | 本 hook 在 git 命令重写（rewrite）已经被 commit 的数据时调用。除了其携带的参数之外，本 hook 还从 stdin 接收信息，信息格式为” ”。                                                                                                                                                                | 触发本 hook 的命令名称（amend 或者 rebase）                                                                                                               |

### 简单实现 commit-msg

在`git init`初始化后自动生成 git hooks 文件，进入文件夹后可以看到每个文件都有一个`.sample` 后缀，Git 决定是否执行一个 hook 文件是通过其文件名来判定的， `.sample` 代表不执行，所以在实现过程中需要去掉后缀才能正常使用。

#### commit-msg 实现代码

```js
#!/usr/bin/env node
const fs = require('fs')

const msg = fs.readFileSync(process.argv[2], 'utf-8').trim() // 索引 2 对应的 commit 消息文件
const commitRE =
  /^(feat|fix|docs|style|refactor|perf|test|workflow|build|ci|chore|release|workflow)(\(.+\))?: .{1,100}/

if (!commitRE.test(msg)) {
  console.log()
  console.error('不合法的 commit 消息格式，请使用正确的提交格式：')
  console.error("feat: add 'comments' option")
  console.error('fix: handle events on blur (close #28)')
  console.error(
    '详情请查看 git commit 提交规范：https://juejin.cn/post/6844903672162304013'
  )
  process.exit(1)
}
```

#### 效果

`commit-msg` 是在我们编辑完一个 commit 的消息后进行调用判断

![Image text](https://raw.githubusercontent.com/fafa123hua/img-folder/master/commit-msg%E6%95%88%E6%9E%9C.png)

#### 同步

为了让 git hooks 文件可以同步给其他人，并且无需手动配置，这里主要用两种方式实现：

##### 链接自定义文件

##### Husky
