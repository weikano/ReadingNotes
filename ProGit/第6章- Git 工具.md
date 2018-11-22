### 第6章- Git 工具

#### 6.3 储藏(Stashing)

> 当你正在进行项目中某一部分的工作，里面的东西处于一个比较杂乱的状态，而你想转到其他分支上进行一些工作。问题是，你不想提交进行了一半的工作，否则以后你无法回到这个工作点。解决这个问题的办法就是`git stash`命令。

- 储藏你的工作

  ```shell
  git stash
  //查看现有储藏历史
  git stash list
  //重新应用储藏历史stach@{2}
  git stash apply stash@{2}
  //取消储藏补丁
  git stash show -p stash@{0} | git apply -R
  ```

- 从储藏中创建分支

  ```shell
  //恢复储藏的工作然后在新的分支上继续当时的工作
  git stash branch testchanges
  ```

- 重写提交历史

  ```shell
  git commit -m "first commit"
  //do some work
  git add --all
  //修改上一次提交
  git commit --amend -m "fixed commit"
  ```

#### 6.4 使用Git调试

- 文件标注

  > 如果你在追踪代码中的缺陷想知道这是什么时候为什么被引进来的，文件标注会是你的最佳工具。它会显示文件中对每一行进行修改的最近一次提交。因此，如果你发现自己代码中的一个方法存在缺陷，你可以用`git blame`来标注文件，查看那个方法的每一行分别是由谁在哪一天修改的。
  >
  > ```shell
  > //1,3为行数，-L为行限制(Line), test.md为文件
  > git blame -L 1,3 test.md
  > ```

- 二分查找

  > 例如你刚刚推送了一个代码发布版本到产品环境中，对代码为什么会表现成那样百思不得其解。你回到你的代码中，还好你可以重现那个问题，但是找不到在哪里。你可以对代码执行bisect来寻找。首先你运行`git bisect start`启动，然后你用`git bisect bad`来告诉系统当前的提交已经有问题了。然后你必须告诉bisect已知的最后一次正常状态是哪次提交，使用`git bisect good [good_commit]`

#### 6.6 子模块

- 添加子模块到当前repository

  ```shell
  git submodule add git://github.com/chneukirchen/rack.git
  //这样在当前项目下会有一个rack子目录
  ```

- clone一个带子模块的项目

  > clone后会包含子项目的目录，但是里面没有文件
  >
  > ```shell
  > git submodule init
  > git submodule update
  > ```

#### 6.7 子树合并

