> git worktree 命令可在不同文件夹中打开同一个 git 仓库的不同分支，很好的解决不同分支 node_modules 依赖冲突的问题。

将一个项目不同平台的版本放在 git 仓库的不同分支，是一种常见的做法。比如我最近在考虑开发 Gitview （[项目链接](https://github.com/entronad/gitview) ）的小程序版，计划将原先 React Native 版的代码放到名为 react-native 的分支，小程序版在一个新的名为 weixin 的分支中开发，master 分支中将只放简介和各分支的索引链接。

由于 node_modules 文件夹是在 .gitignore 之中的，git 不会对其有任何记录或操作，因此不同分支不会有自己独立的 node_modules，使用 `git checkout` 命令切换时，项目里还是同一个 node_modules 文件夹。

此时在各分支直接执行 `npm install` 的话，各分支 package.json 中对应的依赖都会被放到同一个 node_modules 文件夹。如果同一个依赖不同分支的版本不一致，则会冲突覆盖，发生问题。

类似问题还会出现在以下场景：

- 你想将项目的 react 版本从 15 升级到 16 ，并尝试使用一些 16 的新特性，因此建立了一个新的分支，但在这期间，你还要在原分支上维护老版本的项目，互相切换时 react 版本会冲突覆盖
- dev 分支中是全新的内容，但你有些代码片段希望从原来的 master 分支直接拷贝，互相切换后看不到其它分支的代码

以上问题，究其原因是 `git checkout` 命令是在同一个文件夹中切换不同分支。

怎么解决呢，一个思路是不同的分支 clone 到不同的文件夹，但这样就是相互完全独立的仓库了，不能merge。

那有没有办法能在不同的文件夹中打开同一个仓库的不同分支呢，这就要介绍今天的主角：`git worktree` 命令了：

```
git worktree add [-f] [--detach] [--checkout] [--lock] [-b <new-branch>] <path> [<commit-ish>]
git worktree list [--porcelain]
git worktree lock [--reason <string>] <worktree>
git worktree move <worktree> <new-path>
git worktree prune [-n] [-v] [--expire <expire>]
git worktree remove [-f] <worktree>
git worktree unlock <worktree>
```

每个命令详细的介绍请看文档（[链接](https://git-scm.com/docs/git-worktree) ），下面以对 Gitview （[项目链接](https://github.com/entronad/gitview) ）改造为例进行讲解：

原本 Gitview 只有一个 master 分支，为 React Native 版本的代码，本地仓库为 gitview 文件夹，现在希望将 React Native 版本的代码移动到 react-native 分支，新建 weixin 分支进行小程序开发，master 分支作为索引。

改造步骤如下：

1. 将 gitview 文件夹的全部内容移动到 gitview/gitview-master文件夹
2. 在 gitview-master 目录下执行 `git worktree add ../gitview-react-native -b react-native`
3. 在 gitview/gitview-react-native 目录下执行 `git push --set-upstream origin react-native`
4. 删除 gitview/gitview-master 下除了 .git 文件夹的全部内容，再添加所需内容，并add、commit、push
5. 在 gitview-master 目录下执行 `git worktree add ../gitview-weixin -b weixin`
6. 在 gitview/gitview-weixin 目录下添加相应内容，并add、commit、push
7. 在 gitview/gitview-weixin 目录下执行 `git push --set-upstream origin weixin`

这样 gitview 项目在本地和 GitHub 远端都有了 master、react-native、weixin 三个分支，在本地三个分支分别被放置在 gitview/gitview-master、gitview/gitview-react-native、gitview/gitview-weixin 文件夹，但它们同属一个仓库，相互merge没有问题，在对应文件夹下打开 git bash 就是在对应的分支，无需通过 `git checkout` 命令。

注意在执行 `git worktree add` 的时候，不要将目录指定在当前目录下（./），而应该指定在当前目录的同级目录（../），否则新分支的文件夹又会被添加到原分支中。

使用 `git worktree` 建议将 git 升级到最新版本，因为其中的好几个命令是最近几个版本才加入的。事实上，`git worktree` 命令是2015年9月的2.6.0版本才推出的。这其中的原因个人认为可能是，在过去的编程语言中，项目的依赖库不在项目目录中， `git checkout` 不会遇到依赖不同的问题，而npm 管理的项目中，依赖被放在项目目录中的 node_modules 文件夹中，故随着前端开发和 npm 的兴起，在不同文件夹中打开不同分支愈发变成了刚需。