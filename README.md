# Git Hooks

我们想要在git操作时候进行一些定制化操作，比如在git commit时候检查一下提交内容是否合规、git push时候检查一下资源文件大小等等，开发者和管理员们可以指定git在不同事件、不同动作下执行特定的脚本。这里简单实现其中的commit-msg。

### Git Hooks 简介：

http://git-scm.com/book/en/v2在一书中将hooks划分为如下类型

- 客户端的hook：此类hook在提交者（committer）的计算机上被调用执行。此类hook又分为如下几类：
  - 代码提交相关的工作流hook：提交类hook作用在代码提交的动作前后，通常用于运行完整性检查、提交信息生成、信息内容验证等功能，也可以用来发送通知。
  - Email相关工作流hook：Email类hook主要用于使用Email提交的代码补丁。像是Linux内核这样的项目是采用Email进行补丁提交的，就可以使用此类hook。工作方式和提交类hook类似，而且项目维护者可以用此类hook直接完成打补丁的动作。
  - 其他类：包括代码合并、签出（check out）、rebase、重写（rewrite）、以及软件仓库的清理等工作。
- 服务器端hook：此类hook作用在服务器端，一般用于接收推送，部署在项目的git仓库主干（main）所在的服务器上。Chacon将服务器端hook分为两类：
  - 接受触发类：在服务器接收到一个推送之前或之后执行动作，前触发常用于检查，后触发常用于部署。
  - 更新：类似于前触发，不过更新类hook是以分支（branch）作为作用对象，在每一个分支更新通过之前执行代码。

#### hook列表如下：

| Hook名称           | 触发指令                         | 描述                                                         | 参数的个数与描述                                             |
| ------------------ | -------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| applypatch-msg     | `git am`                         | 可以编辑commit时提交的message。通常用于验证或纠正补丁提交的信息以符合项目标准。 | (1) 包含预备commit信息的文件名                               |
| pre-applypatch     | `git am`                         | 虽然这个hook的名称是“打补丁前”，不过实际上的调用时机是打补丁之后、变更commit之前。如果以非0的状态退出，会导致变更成为uncommitted状态。可用于在实际进行commit之前检查代码树的状态。 | 无                                                           |
| post-applypatch    | `git am`                         | 本hook的调用时机是打补丁后、commit完成提交后。因此，本hook无法用于取消进程，而主要用于通知。 | 无                                                           |
| pre-commit         | `git commit`                     | 本hook的调用时机是在获取commit message之前。如果以非0的状态退出则会取消本次commit。主要用于检查commit本身（而不是message） | 无                                                           |
| prepare-commit-msg | `git commit`                     | 本hook的调用时机是在接收默认commit message之后、启动commit message编辑器之前。非0的返回结果会取消本次commit。本hook可用于强制应用指定的commit message。 | 1. 包含commit message的文件名。2. commit message的源（message、template、merge、squash或commit）。3. commit的SHA-1（在现有commit上操作的情况）。 |
| commit-msg         | `git commit`                     | 可用于在message提交之后修改message的内容或打回message不合格的commit。非0的返回结果会取消本次commit。 | (1) 包含message内容的文件名。                                |
| post-commit        | `git commit`                     | 本hook在commit完成之后调用，因此无法用于打回commit。主要用于通知。 | 无                                                           |
| pre-rebase         | `git rebase`                     | 在执行rebase的时候调用，可用于中断不想要的rebase。           | 1. 本次fork的上游。2. 被rebase的分支（如果rebase的是当前分支则没有此参数） |
| post-checkout      | `git checkout` 和 `git clone`    | 更新工作树后调用checkout时调用，或者执行 git clone后调用。主要用于验证环境、显示变更、配置环境。 | 1. 之前的HEAD的ref。 2. 新HEAD的ref。 3. 一个标签，表示其是一次branch checkout还是file checkout。 |
| post-merge         | `git merge` 或 `git pull`        | 合并后调用，无法用于取消合并。可用于进行权限操作等git无法执行的动作。 | (1) 一个标签，表示是否是一次标注为squash的merge。            |
| pre-push           | `git push`                       | 在往远程push之前调用。本hook除了携带参数之外，还同时给stdin输入了如下信息：” ”（每项之间有空格）。这些信息可以用来做一些检查，比如说，如果本地（local）sha1为40个零，则本次push是一个删除操作；如果远程（remote）sha1是40个零，则是一个新的分支。非0的返回结果会取消本次push。 | 1. 远程目标的名称。 2. 远程目标的位置。                      |
| pre-receive        | 远程repo进行`git-receive-pack`   | 本hook在远程repo更新刚被push的ref之前调用。非0的返回结果会中断本次进程。本hook虽然不携带参数，但是会给stdin输入如下信息：” ”。 | 无                                                           |
| update             | 远程repo进行`git-receive-pack`   | 本hook在远程repo每一次ref被push的时候调用（而不是每一次push）。可以用于满足“所有的commit只能快进”这样的需求。 | 1. 被更新的ref名称。2. 老的对象名称。3. 新的对象名称。       |
| post-receive       | 远程repo进行`git-receive-pack`   | 本hook在远程repo上所有ref被更新后，push操作的时候调用。本hook不携带参数，但可以从stdin接收信息，接收格式为” ”。因为hook的调用在更新之后进行，因此无法用于终止进程。 | 无                                                           |
| post-update        | 远程repo进行`git-receive-pack`   | 本hook仅在所有的ref被push之后执行一次。它与post-receive很像，但是不接收旧值与新值。主要用于通知。 | 每个被push的repo都会生成一个参数，参数内容是ref的名称        |
| pre-auto-gc        | `git gc –auto`                   | 用于在自动清理repo之前做一些检查。                           | 无                                                           |
| post-rewrite       | `git commit –amend`,`git-rebase` | 本hook在git命令重写（rewrite）已经被commit的数据时调用。除了其携带的参数之外，本hook还从stdin接收信息，信息格式为” ”。 | 触发本hook的命令名称（amend或者rebase）                      |



### 简单实现commit-msg

在`git init`初始化后自动生成git hooks文件，进入文件夹后可以看到每个文件都有一个`.sample` 后缀，Git决定是否执行一个hook文件是通过其文件名来判定的， `.sample` 代表不执行，所以在实现过程中需要去掉后缀才能正常使用。

#### commit-msg实现代码

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
  process.exit(1)//打断
}

```

#### 效果

`commit-msg` 是在我们编辑完一个commit的消息后进行调用判断

![Image text](https://raw.githubusercontent.com/fafa123hua/img-folder/master/commit-msg%E6%95%88%E6%9E%9C.png)

#### 同步

为了让git hooks文件可以同步给其他人，并且无需手动配置，这里主要用两种方式实现：

##### 链接自定义文件

##### Husky

