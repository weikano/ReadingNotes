### 第2章 Git基础

#### 2.1 取得项目的git仓库

#### 2.2 记录每次更新到仓库

```shell
//比较工作目录中的当前文件和暂存区域快照之间的差异，即修改后还没有暂存起来的变化内容
git diff
//查看暂存文件和上次提交时的快照之间的差异
git diff --cached
//或者
git diff --staged
//跳过使用暂存区域的方式，即跳过git add步骤来提交
git commit -a
//从git中移除文件。首先要从暂存区中移除，然后删除工作区域中的文件
git remove
//另外一种情况是，我们想把文件从 Git 仓库中删除（亦即从暂存区域移除），但仍然希望保留在当前工作目录中。换句话说，仅是从跟踪清单中删除。比如一些大型日志文件或者一堆 .a 编译文件，不小心纳入仓库后，要移除跟踪但不删除文件，以便稍后在 .gitignore 文件中补上，用 --cached 选项即可：
git rm --cached logxx.log
```

#### 2.3 查看提交历史

```shell
git log
```

#### 2.4 撤销操作

```shell
//修改最后一次提交
//想要撤销刚才的提交操作
git commit -amend
//如果刚才提交时忘了暂存某些修改，那么首先补上暂存操作
git add forgortten_file
git commit -amend
//取消已经暂存的文件
git reset HEAD file_tobe_cancel_staged
//取消对文件的修改
git checkout -- file_tobe_revert
```

#### 2.5 远程仓库的使用

**查看当前的远程库**

```shell
//显示远程库,-v显示对应的克隆地址
git remote -v
```

**添加远程仓库**

```shell
//添加一个新的远程仓库，可以指定一个shortname以便使用
git remote add [shortname] [url]
//要抓取所有paul有的，本地没有的信息
git fetch paul
```

**从远程库抓取数据**

```shell
//拉取所有本地仓库中没有的数据
git fetch [remote-name]
```

**推送数据到远程仓库**

```shell
//git push [remote-name] [branch-name]
//默认remote-name为origin, branch-name为master，所以也可以只用git push
git push orgin master
```

**查看远程仓库信息**

```shell
//比如git remote show paul
//git remote show origin
git remote show [remote-name]
```

**远程仓库重命名和删除**

```shell
git remote rename paul pb
git remote rm pb
```

#### 2.6 打标签

**列出已有标签**

```shell
git tag 
```

