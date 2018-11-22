### 第3章-Git 分支

#### 3.2 分支的新建与合并

- 分支的新建与切换

  ```shell
  //新建分支并切换到分支中
  git checkout -b branchname 
  //等同于下列两行命令
  git branch branchname
  git checkout branchname
  ```

- 分支的合并

  ```shell
  //将dev分支中的commit合并到master中
  //首先切换到master分支
  git checkout master
  //merge dev分支
  git merge dev
  //处理conflict...
  //将所有变动添加到暂存区，表示conflict已解决，addcommit 之后可以删除dev分支
  git branch -d dev
  ```

#### 3.3 分支的管理

```shell
git branch [option]
-v 显示每个分支最后一次提交记录
--merged 显示已经合并过的分支
--not-merged 显示还没有被合并的分支
-d 删除分支，后接分支名，比如git branch -d dev
```

#### 3.4 利用分支进行开发的工作流程

#### 3.5 远程分支

> 远程分支（remote branch）是对远程仓库中的分支的索引。它们是一些无法移动的本地分支；只有在 Git 进行网络交互时才会更新。远程分支就像是书签，提醒着你上次连接远程仓库时上面各分支的位置。
>
> 我们用 `(远程仓库名)/(分支名)` 这样的形式表示远程分支。比如我们想看看上次同 `origin` 仓库通讯时 `master` 分支的样子，就应该查看`origin/master` 分支。如果你和同伴一起修复某个问题，但他们先推送了一个 `iss53` 分支到远程仓库，虽然你可能也有一个本地的 `iss53` 分支，但指向服务器上最新更新的却应该是 `origin/iss53` 分支。
>
> 一次 Git 克隆会建立你自己的本地分支 master 和远程分支 origin/master，并且将它们都指向 `origin` 上的 `master` 分支。

- 推送本地分支

  ```shell
  git push (远程仓库名) (分支名)
  ```

- 跟踪远程分支

  > 从远程分支 `checkout` 出来的本地分支，称为 *跟踪分支* (tracking branch)。跟踪分支是一种和某个远程分支有直接联系的本地分支。在跟踪分支里输入 `git push`，Git 会自行推断应该向哪个服务器的哪个分支推送数据。同样，在这些分支里运行 `git pull` 会获取所有远程索引，并把它们的数据都合并到本地分支中来。
  >
  > 在克隆仓库时，Git 通常会自动创建一个名为 `master` 的分支来跟踪 `origin/master`。这正是 `git push` 和 `git pull` 一开始就能正常工作的原因。
  >
  > ```shell
  > git checkout -b [分支名] [远程名]/[分支名]
  > //比如 git checkout -b localdev origin/dev
  > //或者使用--track简化
  > git checkout --track origin/dev
  > ```

- 删除远程分支

  ```shell
  git push [远程名] :[分支名]
  //比如 git push origin :dev，origin后面接空格
  ```

#### 3.6 分支的rebase

> 假设当前有master和dev两个分支
>
> master: master init -> master 1st commit -> master 2nd commit
>
> dev分支为master init之后checkout，然后又进行了一次提交
>
> dev: master init -> dev 1st commit
>
> 然后在dev分支上使用master进行rebase
>
> git checkout dev
>
> git rebase master
>
> 解决冲突调用git add --all，然后git rebase --continue
>
> 这时master的分支不改变，dev分支如下所示：
>
> dev: master init->master 1st commit->master 2nd commit->dev 1st commit
>
> rebase是将master分支记录放在历史最前端，然后将dev分支记录添加在后面
>
> 然后git checkout master切换, 调用git merge dev将分支合并，查看git log发现master的提交记录为后来的dev提交记录