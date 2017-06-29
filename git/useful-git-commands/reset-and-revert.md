# 小命令解决大问题系列（一） —— 重置与回滚

## 写在前面的话

在我们使用 Git 进行代码的版本控制过程中，经常会出现需要重置或者回滚的情况，许多同事很彷徨，不知道如何去做。这也是我写这篇文章的目的，跟着我一步一步来做，重置与回滚其实很简单。

## 重置与回滚的区别
什么时候用重置，什么时候用回滚的界限，其实就是 **有没有 `push`** 到远端仓库。
- 没有 `push` 到远端仓库，就用重置 `reset` 命令即可。`reset` 是用来重置本地仓库 `commit` 的命令。重置后，除了你和 Git 外，谁都不知道（ `git reflog` 命令可以恢复重置，后面会提到）。
- 已经 `push` 到远端仓库，就应该用 `revert` 命令了。`revert` 后会新增加一个 `commit`，`commit` 内容就是之前提交内容的反操作，就和你自己修改了内容提交一样，远端仓库认为这是一个新的提交，从而应用到远端分支上。

需要注意的是，`revert` 后，如果再 `merge` 之前的分支，就会导致 `revert`掉的内容合并不上, 除非你想回滚 回滚的回滚，否则可能需要用 `cherry-pick` 命令来进行操作。

为什么 `push` 后，用 `reset` 不能呢？
之前也说到了，`reset` 只是用来重置到之前的 `commit`，并不是新增加一个 `commit`，当你 reset 后，你再 `push`，远端仓库就会认为，远端的版本比你本地版本更“新”，需要你本地仓库先 `pull` ，才能 `push`。

```
* reset 示意图
A -> B -> C                      A -> B --> C(丢弃)
          /     => reset(B) =>        /
        HEAD                        HEAD


* revert 示意图

A -> B -> C                       A -> B -> C -> C' 
          /     => revert(B) =>                  /
        HEAD                                   HEAD

C' 表示 C 这条记录的反操作，即 (添加)' => 删除，(增加)' => 减少
```

知道了两者的区别，我们来看看如何操作。

## 重置 reset
### 命令使用
1. 在重置之前，我们需要先看看要回到哪条 commit 上去

    ```shell
    $ git log
    
    commit dca454e48b9f8b200ae973397c164cf1c525e11c
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Mon Jun 26 14:21:54 2017 +0800
    
        修改 index 页面样式 & 配置不同环境下 fis 的 domain
    
    commit 02643da5336f3a3bb65ef213ed373db095188c99
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Thu Jun 22 18:39:55 2017 +0800
    
        修复 样式复用导致的样式问题
    ```
    
    记录一下 commit 后面的 sha1 值（一般前 7 位即可）, 这个是 commit 的唯一标识，我们需要用它来回滚到指定的 commit。
    
    ps: 其实，如果只需要显示前 3条记录，只需要在 git log 后面添加 -3 就可以了 `git log -3`。关于 `git log` 的其他用法，运行 `git help log` 即可自动用浏览器打开文档。
    
    ```shell
    $ git reset 02643da5
    Unstaged changes after reset:
    M       css/register.css
    M       fis-conf.js
    M       index.html
    M       js/attestation/improve.js
    M       js/controllers/newindexController.js
    D       register.html
    
    ```
    
    可以看到，这些文件的修改还保留着，也就是说，02643da5（当前 HEAD） 与还原前 HEAD 之间改变的文件显示出来了。他们的状态是 `Unstaged` （未暂存）也就是在 `git add` 之前的状态。

2. 我们现在加一个 `--soft` 到命令上

    ```shell
    $ git reset --soft 02643da5
    # 没有东西显示出来，运行下 git status
    
    $ git status
    On branch develop
    Your branch is up-to-date with 'origin/develop'.
    Changes to be committed:
      (use "git reset HEAD <file>..." to unstage)
    
            modified:   css/register.css
            modified:   fis-conf.js
            modified:   index.html
            modified:   js/attestation/improve.js
            modified:   js/controllers/newindexController.js
            deleted:    register.html
    
    ```
    可以看到，02643da5（当前 HEAD） 与还原前 HEAD 之间改变的文件 被 add 到暂存区了。
    
    `--soft` option 其实等同于 在 `git reset` 后，执行 `git add` 操作。

3. 再次的，加一个 `--hard` 到命令上, 试试看效果

    ```shell
    $ git reset --hard 02643da5
    HEAD is now at 10dab47 修改
    
    $ git status
    On branch develop
    Your branch is up-to-date with 'origin/develop'.
    nothing to commit, working directory clean
    ```
    
    可以看到，`--hard` 命令其实就是丢弃两个 commit 之间的修改，直接强制回到指定版本。

### 扩展知识
- 撤销重置
    重置完成后，又不想重置了，怎么破？没关系，`git reflog` 与 `git reset` 配合就可以实现撤销重置了。且看如下操作：

    ```shell
    $ git reflog
    02643da HEAD@{0}: reset: moving to 02643da5
    10dab47 HEAD@{1}: checkout: moving from test to develop
    99f53d2 HEAD@{2}: merge develop: Merge made by the 'recursive' strategy.
    4be83b6 HEAD@{3}: checkout: moving from develop to test
    ...
    ```

    可以看到，每一步的操作，都被记录下来了，看到第一列的 sha1 值了么，还原操作就靠他了。

    ```shell
    git reset --hard 10dab47
    HEAD is now at 10dab47 修改
    ```
    搞定了，就是这么简单。


## 回滚 revert
### 命令使用
1. 看准了需要回滚的 commit 的 sha1 值，我们就来执行命令了
    ```shell
    $ git revert 04ad0de
    ```
    会出来一个 编辑 commit 信息的界面，编辑完后，保存就好了。
使用 `git log` 命令查看一下是否真的 revert 好了
    ```shell
    $ git log -1
    commit 7b229b68b934dc85d2aa15808e30265d9c81d766
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:32:17 2017 +0800
    
        Revert "add feature 3.8-1"

        This reverts commit 04ad0de8e8478834182374b978c39443f576607b.
    ```

2. 一次回滚多个提交
和 SVN 不同，git 回滚是以 commit 为单位的，回滚只是回滚指定 commit 的内容，而不是指定的 commit 之后的全部 commit。

    ```
    A -> B -> C -> D                            A -> B -> C -> D -> B'
                   /      =>   revert(B)   =>                       /   
                 HEAD                                             HEAD
    ```

    如果要一次回滚多个提交，怎么做呢？分两种情况
        
    - 多个提交连续，则可以使用 `..` 符，如上面例子，想恢复到 A 提交，则可以运行
        
    ```
    $ git revert A..D
    # A 表示停止 revert 的 commit
    ```
    
    期间会出来多个编辑 commit 信息的界面，编辑完后，保存就可以了。记得，commit sha1 一定要从前到后哦。
    ```shell
    $ git revert 9e7c5713bf5f5c8b82ddef32eed6cb44edc96ddf..HEAD
    
    $ git log -6
    commit e96a6db7592f6971f432d2cac18b4d6648133252
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:52:16 2017 +0800
    
        Revert "add feature 3.8-1"
    
        This reverts commit 04ad0de8e8478834182374b978c39443f576607b.
    
    commit 707385b76897c90c4a6d7ded61afebdae487e18b
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:52:13 2017 +0800
    
        Revert "add new line"
    
        This reverts commit 9191a635e0a26859461ce6dc53a889bc6dec23c4.
    
    commit 20a0f617b2d77e0b53f4175fba7ef45a2be07d50
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:52:09 2017 +0800
    
        Revert "new feature app3.8-1{1}"
    
        This reverts commit 0f7e179f5e0a7d50b16c285339ca4e97f1efafe2.
    
    commit 0f7e179f5e0a7d50b16c285339ca4e97f1efafe2
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:36:39 2017 +0800
    
        new feature app3.8-1{1}
    
    commit 9191a635e0a26859461ce6dc53a889bc6dec23c4
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:36:12 2017 +0800
    
        add new line
    
    commit 04ad0de8e8478834182374b978c39443f576607b
    Author: 刘汶竹 <lwz0316@gmail.com>
    Date:   Wed Jun 28 15:26:09 2017 +0800
    
        add feature 3.8-1
    ```

    通过上面的信息可以看到，回滚是从最“新”一步一步回滚到最后的

    ```
    A -> B -> C -> D                            A -> B -> C -> D -> D' -> C' -> B'
                   /      =>   revert(A..D)   =>                                /   
                 HEAD                                                         HEAD
    ```
    
    - 多个提交不连续，指定每个提交
    
    ```
    A -> B -> C -> D                            A -> B -> C -> D -> C' -> B'
                   /      =>   revert(C B)   =>                           /   
                 HEAD                                                   HEAD
    ```
    
    ```shell
    $ git revert 9191a6 04ad0d
    error: could not revert 9191a63... add new line
    hint: after resolving the conflicts, mark the corrected paths
    hint: with 'git add <paths>' or 'git rm <paths>'
    hint: and commit the result with 'git commit'
    ```

    Oophs! 出错了呢，我们来看看是为啥

    ```shell
    On branch feature/app3.8
    Your branch is ahead of 'origin/feature/app3.8' by 3 commits.
      (use "git push" to publish your local commits)
    You are currently reverting commit 9191a63.
      (fix conflicts and run "git revert --continue")
      (use "git revert --abort" to cancel the revert operation)
    
    Unmerged paths:
      (use "git reset HEAD <file>..." to unstage)
      (use "git add <file>..." to mark resolution)
    
            both modified:   README.md
    ```    

    可以看到，原来是合并冲突了，编辑下冲突的文件， 然后运行

    ```shell
    $ git add .
    $ git revert --continue
    ```

    然后跳出一个信息编辑框，编辑信息保存，就好了。

    其实并不是每个 revert 都会 冲突的，遇到冲突了，解决下，然后执行上面的两步，就可以了。

3. 回滚的提交中，遇到 `merge` 的提交
我们先在 `git log` 命令中添加 `--graph` 看一下哪个 `commit` 是 `merge` 来的。  

    ```shell
    $ git log --graph
    ...
    |
    *   commit 5106220bd099192bdd5d6ba9bc0179e607d8bbca
    |\  Merge: 1e992b9 6cfb91e
    | | Author: liuwenzhu <lwz0316@gmail.com>
    | | Date:   Wed Apr 13 14:21:22 2016 +0800
    | |
    | |     Merge branch 'release/1.2' into develop
    | |
    | * commit 6cfb91e25cc71e29bd19c7cc5cb8cc1f2fd8d2e6
    | | Author: liuwenzhu <lwz0316@gmail.com>
    | | Date:   Wed Apr 13 14:20:39 2016 +0800
    | |
    | |     fixbug test
    | |
    * | commit 1e992b962f21f00d02585e4d706dd64677395c2a
    | | Author: liuwenzhu <lwz0316@gmail.com>
    | | Date:   Wed Apr 13 14:19:39 2016 +0800
    | |
    ```

    可以看到，`5106220b` 这个 `commit` 是合并来的，有两个父 `commit`, 分别是 `1e992b9` 和 `6cfb91e`， 这个时候，我们执行回滚命令
    
    ```shell
    $ git revert 5106220b
    error: Commit 5106220bd099192bdd5d6ba9bc0179e607d8bbca is a merge but no -m option was given.
    fatal: revert failed
    ```

    恩，如你所见，报错了，说的是 `5106220b` 这个 `commit` 是有父分支的，必须使用 `-m` 指令。那，`-m` 是什么鬼？不知道怎么用命令，那么就找帮助喽 
    ```
    $ git help revert
    ```
    
    可以看到官方的解释
    
    > -m parent-number
    >  -- mainline parent-number
    > Usually you cannot revert a merge because you do not know which side of the merge should be considered the mainline. This option specifies the parent number (starting from 1) of the mainline and allows revert to reverse the change relative to the specified parent.
    
    > Reverting a merge commit declares that you will never want the tree changes brought in by the merge. As a result, later merges will only bring in tree changes introduced by commits that are not ancestors of the previously reverted merge. This may or may not be what you want.

    大概的意思就是 指定你要回滚到哪个父 `commit`，左边的是 1， 右边的是 2。
    
    还记得 `5106220b` 的两个父 `commit` `1e992b9`(1) 和 `6cfb91e`(2) 么？

    如果想要会滚到 `6cfb91e`， 那么就执行
    ```shell
    $ git revert 5106220b -m 2
    ```

### 扩展知识
- `cherry-pick` 用于把指定的 `commit` 的修改应用到当前分支

    ```
          E -> F                                     E -> F
         /                                         /
    A -> B -> C -> D                              A -> B -> C -> D -> E
                  /     =>   cherry-pick(E) =>                       /   
                HEAD                                              HEAD
    ```

    试一试
    ```shell
    $ git cherry-pick c076192
    error: could not apply c076192... new feature app3.8-1{3}
    hint: after resolving the conflicts, mark the corrected paths
    hint: with 'git add <paths>' or 'git rm <paths>'
    hint: and commit the result with 'git commit'
    ```
    
    Oops! 冲突了, 让我来解决一下
    ```shell
    $ git add .
    $ git cherry-pick --continue
    # 保存弹出的提交信息编辑框中的内容
    [feature/app3.8 ecd46ac] new feature app3.8-1{3}
     Date: Wed Jun 28 18:12:42 2017 +0800
     1 file changed, 3 insertions(+), 1 deletion(-)
    ```
    
    搞定。
    
    如果有多个 `commit` 需要应用 那么就在 `git cherry-pick` 后面添加多个 `commit sha1` 中间用 空格隔开即可，如
    ```shell
    $ git cherry-pick ecd46ac 28f9446 0f7e179f
    ```


