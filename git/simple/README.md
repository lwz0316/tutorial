Git 简易教程
=====

本教程将简单介绍Git的使用 
> Git: everything-is-local

参考文档

- [Pro Git（中文版）](http://git.oschina.net/progit/)
- [GIT分支管理是一门艺术](http://kb.cnblogs.com/page/132209)

## Git 简介

Git 诞生自2005年，诞生的原因是因为 Linux 的分布式版本控制系统的供应商收回了免费使用的权利，故而‘自力更生’——开发一套属于自己的版本控制系统。

他们的目标是
    
- 速度
- 简单的设计
- 对非线性开发模式的强力支持（允许上千个并行开发的分支）
- 完全分布式
- 有能力高效管理类似 Linux 内核一样的超大规模项目（速度和数据量）

目前为止 Git 版本是 `2.8.2`，在高度易用的同时，仍然保留着初期设定的目标，速度飞快，极其适合管理大项目，它还有着令人难以置信的非线性分支管理系统，可以应付各种复杂的项目开发需求。

    与 Linux 一样 Git 是免费且开源的，这非常棒！

与其他的集中式版本控制系统(如Subversion) 不一样,Git 是分布式版本控制系统。 

### 集中式版本控制系统
> Centralized Version Control Systems，简称 CVCS

一切都依赖于中央服务器，不管是提交修改还是更新本地仓库。如果中央服务器单点宕机，那么恢复前的一段时间内就无法提交更新了。

更糟的情况是，如果服务器磁盘发生故障又恰巧没有备份,就有丢失数据的风险乃至彻底丢失整个项目的所有历史记录，而客户端偶然提取出来的本地数据就成了恢复数据的希望，这只是希望。

### 分布式版本控制系统
> Distributed Version Control System，简称 DVCS

客户端并不只提取最新版本的文件快照，而是把代码仓库完整地镜像下来。这么一来，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复。因为每一次的提取操作，实际上都是一次对代码仓库的完整备份。

## 初识 Git

### 空间换时间

Git 和其他版本控制系统的主要差别在于，Git 只关心文件数据的整体是否发生变化，而大多数其他系统则只关心文件内容的具体差异。

Git 更像是个小型的文件系统，每次提交更新会对所有文件的指纹（SHA-1）信息进行扫描，如果文件变化，则会生成文件的快照（其实就是变化后的整个文件），并保存快照的索引；如果文件没有变化，则Git不会再次保存文件快照，只会对上次保存的快照索引做链接。

Git 版本控制就是以这种简单粗暴的空间换时间的方式，实现飞快的速度。

- 设计同学要小心使用 Git

		Git 是对整个文件做快照，所以 Git 对大的二进制文件不友好，经常修改会造成本地仓库过大的问题。

### 近乎所有操作都是本地执行

除了与远程仓库同步数据外，其他的操作都是本地执行的。因为 Git 在本地磁盘上就保存着所有当前项目的历史纪录，所以处理起来飞快。

举个例子，就一个恢复历史版本的操作，条件苛刻一点，没有网络
- Git

	切换到历史版本的提交节点，一切就是这么简单。

- SVN
   
    使用 SVN 的同学已哭晕在厕所...

### 轻量级的分支和标签操作

在很多版本控制系统中，这是个昂贵的过程，常常需要创建一个源代码目录的完整副本，对大型项目来说会花费很长时间。

在 Git 中新建一个分支或标签非常的简单，但是操作简单并不意味着功能简单，相反，Git 的分支操作非常的高效，Git 的分支可谓是难以置信的轻量级，它的新建操作几乎可以在瞬间完成，并且在不同分支间切换起来也差不多一样快。

Git 鼓励在工作流程中频繁使用分支与合并，理解分支的概念并熟练运用后，你才会意识到为什么 Git 是一个如此强大而独特的工具，并从此真正改变你的开发方式。

## 使用 Git
### 初次运行 Git 前的配置
Git 提供了一个叫做 git config 的工具, 专门用来配置或读取相应的工作环境变量。

#### 用户信息

**最重要**的配置是你个人的用户名称和电子邮件地址。每次 Git 提交时都会引用这两条信息，说明是谁提交了更新，所以会随更新内容一起被永久纳入历史记录：

		# 全局配置提交者的姓名和邮箱
		# 配置文件在 ~/.gitconfig 中
		$ git config --global user.name "Wenzhu Liu"
    	$ git config --global user.email "lwz0316@gmail.com"

使用 `--global` 选项，就会在所有项目中使用默认设置的用户名和邮箱，如果要在某个项目设置特定的名字或者邮箱，则不要加上 `--global` 选项重新配置即可。**

		# 配置当前项目的提交者的邮箱
		# 新的设定保存在当前项目的 `.git/config` 文件里
    	$ git config user.email "liuwenzhu@ppdai.com"

#### 查看配置信息

查看已有的所有配置信息

		$ git config --list
		user.name=Wenzhu Liu
		user.email=lwz0316@gmail.com
		core.autocrlf=true
		...

查看某条配置信息

		$ git config user.name
		Wenzhu Liu

#### 获取帮助

想了解 Git 的各式工具该怎么用，可以阅读它们的使用帮助

	$ git help <verb>
    $ git <verb> --help
    $ man git-<verb>

例如查看 `config` 命令怎么使用，可以运行

	$ git config --help

这些帮助信息都是不需要联网的。非常赞哦！

### Git 仓库的获取

有两种获取 Git 项目仓库的方法。
- 在现存的目录下，通过导入所有文件来创建新的 Git 
- 从已有的 Git 仓库克隆出一个新的镜像仓库来。

#### 创建新仓库

定位到项目所在的目录，然后执行

	$ git init

初始化后，在当前目录下就会出现一个名为 `.git` 的目录。就说明 Git 仓库创建成功了。

让我们将当前目录下的一些文件纳入版本控制。

	$ git add README.md
	$ git commit -m 'add readme file'


#### 克隆已有仓库

克隆仓库需要执行 `git clone [url]` 命令，

	$ git clone https://github.com/lwz0316/tutorial.git

执行完命令后，会在本地创建一个以 url 的最后一个 `/` 和 `.git`之间的字符串命名的目录。例如 `tutorial`。

查看 clone 下来的仓库目录，你会看到项目中包含一个 `.git` 目录和所有工作区的文件，它已经准备好后续的开发和使用了。

如果希望 clone 的时候自定义目标目录名称，可以在末尾指定新的名字

	$ git clone https://github.com/lwz0316/tutorial.git mytutorial

唯一的差别就是，新建的目录变成了 `mytutorial`。

### 文件状态

现在我们手上已经有了一个真实项目的 Git 仓库，并从这个仓库中取出了所有文件的工作拷贝。接下来，对这些文件作些修改，在完成了一个阶段的目标之后，提交本次更新到仓库。

在 Git 中，一个文件有三种状态

- 已修改（modified）
- 已暂存（staged）
- 已提交（committed）

以及它们对应的位置分别是

- 工作目录（working directory）。指从项目中取出某个版本的所有文件和目录，用以开始后续工作的文件夹。我们可以对工作目录下的文件进行编辑。
- 暂存区域（staging area）。指下一次要提交的文件快照的索引（SHA-1）清单列表。
- 本地仓库（git directory/repository）。从暂存区域中已经提交的文件快照和索引，也就是历史记录。

暂存区域和本地仓库是放在项目的 Git 目录—— `.git`文件夹，他是用来保存元数据和对象数据库的地方。这个目录非常重要，每次克隆镜像仓库的时候，实际拷贝的就是这个目录里面的数据。

基本的 Git 工作流程如下：
- 在工作目录中修改某些文件。
- 对修改后的文件进行快照，然后保存到暂存区域。
- 提交更新，将保存在暂存区域的文件快照永久转储到 Git 目录中。

**工作目录**下的所有文件都只有两种状态

- 已跟踪

	已经纳入版本控制管理的文件，上次快照中有他们的记录，工作一段时间后，它们的状态可能是未更新，已修改或者已放入暂存区。

- 未跟踪。

	除了已跟踪文件，其他的文件都是未跟踪。既没有上次文件快照，也没有放入暂存区域中。

使用 Git 时的文件状态变化周期为

![文件的状态变化周期](img/18333fig0201-tn.png)


#### 检查当前文件的状态

要确定当前工作区的文件处于什么状态，可以使用 

	$ git status

- 如果当前工作区非常的干净，没有任何需要提交或者跟踪的文件，那么就会显示像下面一样的信息

		$ git status
		On branch master
		nothing to commit, working directory clean

该命令还显示了当前所在的分支是 master。

- 如果当前工作区中只有已修改但是没有提交的文件，那么就会显示像下面一样的信息

		$ git status
		On branch master
		Changes not staged for commit:
		  (use "git add <file>..." to update what will be committed)
		  (use "git checkout -- <file>..." to discard changes in working directory)
		
		        modified:   README.md
		
		no changes added to commit (use "git add" and/or "git commit -a")

上面的信息`Changes not staged for commit`可以告诉我们，`README.md` 文件被修改了，但是我们没有将其放入暂存区域中准备提交，还提示我们可能可以进行操作的一些命令。如

		$ git add <file> ... 
		# 将待提交的文件添加到暂存区中，以便下次提交	

Git 不会自动将之纳入跟踪范围，除非你明明白白地告诉它“我需要跟踪该文件”，因而不用担心把临时文件什么的也归入版本管理。

- 工作区中只有未跟踪（未纳入版本控制）的文件，那么就会显示像下面一样的信息 

		$ git status
		On branch master
		Untracked files:
		  (use "git add <file>..." to include in what will be committed)
		
		        README.md
		
		nothing added to commit but untracked files present (use "git add" to track)

上面的信息 `Untracked files` 告诉我们这个文件还没有被跟踪，即加入到版本控制中去。

#### 跟踪新文件

使用命令 `git add` 开始跟踪一个新文件,所以要跟踪 desc.txt 文件，需要执行命令

	$ git add desc.txt

其实 git add 的潜台词就是把目标文件快照放入暂存区域，也就是 add file into staged area，同时未曾跟踪过的文件标记为需要跟踪。这样就好理解后续 add 操作的实际意义了。

#### 暂存已修改文件
也是使用 `git add` 将修改后的 README.md 文件添加到暂存区中，以便下次提交。

在执行 `git add` 命令前，我们先执行 `git status`

	$ git status
	On branch master
	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)
	
	        new file:   desc.txt
	
	Changes not staged for commit:
	  (use "git add <file>..." to update what will be committed)
	  (use "git checkout -- <file>..." to discard changes in working directory)
	
	        modified:   README.md

可以看到 README.md 文件出现在 `Changes not staged for commit` 下面，表示文件已经被修改，但是还没有放到暂存区域中。

执行 `git add` 命令后，再执行 `git status` 命令

	$ git add README.md
	$ git status
	On branch master
	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)
	
	        modified:   README.md
	        new file:   desc.txt

我们可以看到，现在两个文件都加入到了暂存区域中去了。

现在我们再次修改下 README.md 文件，删除掉刚才添加的记录，然后再看看状态

	$ git status
	On branch master
	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)
	
	        modified:   README.md
	        new file:   desc.txt
	
	Changes not staged for commit:
	  (use "git add <file>..." to update what will be committed)
	  (use "git checkout -- <file>..." to discard changes in working directory)
	
	        modified:   README.md

发生了什么？ README.md 出现了两次！一次已暂存，一次未暂存。如果现在提交代码，那么 Git 会将 README.md 添加纪录的版本入库，而删除记录的版本则没有被提交，这也就是暂存区域的作用。

	Git 的提交动作其实就是将到暂存区域的记录永久入库的过程

现在我们执行 `git add` 命令，更新暂存区中 README.md 的修改，然后再看看状态

		$ git add README.MD
		$ git status
		On branch master
		Changes to be committed:
		  (use "git reset HEAD <file>..." to unstage)
		
		        modified:   README.md
		        new file:   desc.txt

#### 查看未暂存和已暂存文件的差异
实际上 `git status` 只显示修改过的文件，并没有显示出改动的地方，这个时候需要用到另一个命令 `git diff`

- 查看已暂存的文件差异

		$ git diff --cached 
		diff --git a/README.md b/README.md
		index 8b13789..8fe54e6 100644
		--- a/README.md
		+++ b/README.md
		@@ -1 +1,2 @@
		-
		+THIS IS TEST REPO
		+====

	此命令是比较**已经暂存**起来的文件和**上次提交**时的快照之间的差异

	Git 1.6.1 及更高版本还允许使用 `git diff --staged`， 更好记一点。

- 查看未暂存文件差异

		$ git diff
		diff --git a/README.md b/README.md
		index 8fe54e6..c4463be 100644
		--- a/README.md
		+++ b/README.md
		@@ -1,2 +1,3 @@
		 THIS IS TEST REPO
		 ====
		+> Author: @liuwenzhu

	可以看到 README.md 文件做出的修改。此命令是比较**工作目录**中当前的文件和**暂存区域**快照之间的差异。

#### 提交更新

现在暂存区域已经准备提交了，在此之前，请一定要确认还有什么修改过的或新建的文件还没有 `git add` 过，否则提交的时候不会记录这些还没暂存起来的变化。

所以，在每次提交前，先用 `git status` 查看下是否还有未暂存的文件（除非是你不想提交）

使用提交命令 `git commit`

	$ git commit -m 'update README.md'
	[master 9ee9d95] update README.md
 	1 file changed, 3 insertions(+), 1 deletion(-)

可以看到，提交后它会告诉你，当前是在哪个分支（master）提交的，本次提交的完整 SHA-1 校验和是什么（9ee9d95），以及在本次提交中，有多少文件修订过，多少行添改和删改过。

#### 跳过使用暂存区域

是不是觉得要提交非常繁琐？ Git 考虑到这一点了，现在我们来看看如何简化这一提交流程。

只需要在提交的时候，给 `git commit` 后面加上 `-a` 选项，Git 就会自动把所有已经跟踪过的文件暂存起来一并提交。

	$ git commit -a -m 'update README, add company name'
	[master 3cbfa1b] update README, add company name
 	1 file changed, 2 insertions(+), 1 deletion(-)

注意，这个命令只会把所有**已经跟踪**过的文件暂存起来一并提交,如果有新的文件，Git 并**不会将新的文件进行跟踪**。

#### 忽略某些文件

我们总会有些文件无需纳入 Git 的管理，比如在我们编写代码的时候，有些自动生成的文件（如 日志文件、 build 文件、编译时创建的临时文件），我们也不希望他们出现在未跟踪列表中。

我们可以创建一个忽略配置文件，名为 `.gitignore`的文件，列出要忽略的文件。嗯，有人专门在 [Github](https://github.com/) 上创建了一个[项目](https://github.com/github/gitignore) ，其收集了几乎所有开发语言会用到的忽略配置文件。我们自己创建`.gitignore`的时候不妨先参考它。

	要养成一开始就设置好 .gitignore 文件的习惯，以免将来误提交这类无用的文件。


> 文件 `.gitignore` 的格式规范如下：
> 
> - 所有空行或者以注释符号 ＃ 开头的行都会被 Git 忽略。
> - 可以使用标准的 glob 模式匹配。
> - 匹配模式最后跟反斜杠（/）说明要忽略的是目录。
> - 要忽略指定模式以外的文件或目录，可以在模式前加上惊叹号（!）取反。
> 
> 所谓的 glob 模式是指 shell 所使用的简化了的正则表达式。星号（*）匹配零个或多个任意字符；[abc] 匹配任何一个列在方括号中的字符（这个例子要么匹配一个 a，要么匹配一个 b，要么匹配一个 c）；问号（?）只匹配一个任意字符；如果在方括号中使用短划线分隔两个字符，表示所有在这两个字符范围内的都可以匹配（比如 [0-9] 表示匹配所有 0 到 9 的数字）。
> 
> 我们再看一个 .gitignore 文件的例子：
> 
> 	# 此为注释 – 将被 Git 忽略
> 	    # 忽略所有 .a 结尾的文件
> 	    *.a
> 	    # 但 lib.a 除外
> 	    !lib.a
> 	    # 仅仅忽略项目根目录下的 TODO 文件，不包括 subdir/TODO
> 	    /TODO
> 	    # 忽略 build/ 目录下的所有文件
> 	    build/
> 	    # 会忽略 doc/notes.txt 但不包括 doc/server/arch.txt
> 	    doc/*.txt

`.gitignore` 只能忽略未跟踪的文件，如果在添加 `.gitignore` 前将不需要进行跟踪的文件提交了怎么办？

答案就是，将其删除，然后提交修改。再将其添加到 `.gitignore` 中。以后就不会出现它了，提交的历史变得清爽了。如何删除文件，不再进行版本控制，将在下一小节——移除文件中介绍

再次提醒一下：

	要养成一开始就设置好 .gitignore 文件的习惯，以免将来误提交这类无用的文件。

#### 移除文件

要从 Git 中移除某个文件，就必须要从已跟踪文件清单中移除（确切地说，是从暂存区域移除），然后提交。可以用 `git rm` 命令完成此项工作，并连带从工作目录中删除指定的文件，这样以后就不会出现在未跟踪文件清单中了。

- 如果只是简单地从工作目录中手工删除文件

		$ git rm desc.txt
		$ git status
		On branch master
		Changes to be committed:
		  (use "git reset HEAD <file>..." to unstage)
		
		        deleted:    desc.txt
		
		$ git commit -m 'delete desc.txt'

	最后提交的时候，该文件就不再纳入版本管理了。工作区的该文件也被删除了。
	
- 另外一种情况是，我们想把文件从 Git 仓库中删除，但仍然希望保留在当前工作目录中。换句话说，仅是从跟踪清单中删除。可以用 `--cached` 选项即可

		$ git rm --cached desc.txt
		$ git status
		On branch master
		Changes to be committed:
		  (use "git reset HEAD <file>..." to unstage)
		
		        deleted:    desc.txt
		
		Untracked files:
		  (use "git add <file>..." to include in what will be committed)
		
		        desc.txt
	然后 `git commit` 即可，可以看到本地工作区还保留着 `desc.txt` 文件。这个时候，`desc.txt` 就是未跟踪状态，我们可以在 `.gitignore` 中重新忽略它。


### 查看提交历史

在提交了若干更新后，或者你 Clone 了某个项目，想回顾下提交历史，可以使用 `git log` 命令查看

	$ git log
	commit f6cb19981d5312cd6e9b803e908d44f5f80a8c9e
	Author: Liu Wenzhu <lwz0316@gmail.com>
	Date:   Fri May 20 10:16:29 2016 +0800
	
	    delete desc file
	
	commit 3cbfa1b189462fd52527ced09cc7346adb760e62
	Author: Liu Wenzhu <lwz0316@gmail.com>
	Date:   Fri May 20 08:05:40 2016 +0800
	
	    update README, add company name
	
	...

默认不用参数的话，`git log` 会按照提交时间列出所有的更新，最近的历史在最上面。

`git log` 有许多选项可以帮助我们搜索感兴趣的提交，下面介绍几个最常用的

- `git log -N` 其中 N 是自然数，表示显示 N 条数据
- `git log -p` 表示在显示基本信息的同时也显示内容的差异
- `git log --stat` 仅显示简要的增改行数统计
- `git log --graph` 可以看到开头多出一些 ASCII 字符串表示的简单图形，形象地展示了每个提交所在的分支及其分化衍合情况

使用 `git help log` 查看其他 log 相关的命令吧！

### 撤销操作
谁都想有后悔药吃，Git 也提供了一些让我们后悔的命令。 
	
	请注意，有些撤销操作是不可逆的，所以请务必谨慎小心，一旦失误，就有可能丢失部分工作成果。

#### 修改最后一次提交

有时候我们提交完了才发现漏掉了几个文件没有加，或者提交信息写错了。想要撤消刚才的提交操作，可以使用 `--amend` 选项重新提交

	$ git commit --amend
	
然后修改提交信息，并保存。

如果刚才提交时忘了暂存某些修改，可以先补上暂存操作，然后再运行 `--amend`

	$ git add forgotten_files
	$ git commit --amend 

请注意，**不要在同步到远程仓库后还进行撤销操作**，因为重新修改的提交的索引是不一样的，因为修改的时间戳改变了，故索引值也随之改变，如果这时候推送到远程仓库，那么远程仓库会有两条修改都一致但是提交信息不一致的记录，这会使其他开发者困惑。

#### 取消已经暂存的文件

你可能已经注意到当你将文件放入暂存区然后查看状态时，打印的信息

	$ git status
	On branch master
	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)

        	modified:   README.md

是的，上面（）中的提示
	
	use "git reset HEAD <file>..." to unstage

我们可以使用上面的命令让暂存取中的文件状态重新回到未暂存状态

	$ git reset HEAD README.md
	Unstaged changes after reset:
	M       README.md

	$ git status
	On branch master
	Changes not staged for commit:
	  (use "git add <file>..." to update what will be committed)
	  (use "git checkout -- <file>..." to discard changes in working directory)
	
	        modified:   README.md
	
	no changes added to commit (use "git add" and/or "git commit -a")

#### 取消对文件的修改

是的，细心的同学肯定在上面的执行结果中，又捕捉到了一条有意思的信息
	
	use "git checkout -- <file>..." to discard changes in working directory

可以取消对文件的修改

	$ git checkout -- README.md
	$ git status
	On branch master
	nothing to commit, working directory clean

> 其实 Git 会给你很多的提示，只要你善于观察。 

### 远程仓库的使用

要参与任何一个 Git 项目的协作，必须要了解该如何管理远程仓库。远程仓库是指托管在网络上的项目仓库，可能会有好多个，其中有些你只能读，另外有些可以写。

管理远程仓库的工作，包括添加远程库，移除废弃的远程库，管理各式远程库分支，定义是否跟踪这些分支，等等。

#### 查看当前的远程库

要查看当前的配置有哪些远程仓库，可以使用 `git remote` 命令，它会列出每个远程仓库的别名

	$ git remote
	origin

说明只有一个别名为 `origin` 的远程仓库，但是信息量太少了，可以在后面加个 `-v`

	$ git remote -v
	origin  https://github.com/lwz0316/tutorial.git (fetch)
	origin  https://github.com/lwz0316/tutorial.git (push)

可以看到 拉取的仓库和推送的仓库及他们的地址，虽然都是同一个。

#### 添加远程仓库

添加一个新的远程仓库，可以制定一个别名，以便将来使用或者自己识别使用命令 `git remote add [alise] [url]`即可

	$ git remote add ppdai http://192.168.211.238/liuwenzhu/tutorial.git

	$ git remote -v
	origin  https://github.com/lwz0316/tutorial.git (fetch)
	origin  https://github.com/lwz0316/tutorial.git (push)
	ppdai   http://192.168.211.238/liuwenzhu/tutorial.git (fetch)
	ppdai   http://192.168.211.238/liuwenzhu/tutorial.git (push)

可以看到我们成功添加了 `ppdai` Git 服务器的远程仓库

#### 从远程仓库抓取数据

将远程的仓库的数据抓取到本地

	git fetch [remote-name]
	
如果是克隆了一个仓库，此命令会自动将远程仓库归于 origin 名下。所以，git fetch origin 会抓取从你上次克隆以来别人上传到此远程仓库中的所有更新（或是上次 fetch 以来别人提交的更新）。

> 有一点很重要，需要记住，fetch 命令只是将远端的数据拉到本地仓库，并不自动合并到当前工作分支，只有当你确实准备好了，才能手工合并。

可以使用 `git pull` 命令自动抓取数据下来，然后将远端分支自动合并到本地仓库中**当前分支**。

#### 推送数据到远程仓库
项目进行到一个阶段，要同别人分享目前的成果，可以将本地仓库推送到远程仓库。
	
	git push [remote-name] [branch-name]

如果在你推数据前，已经有其他人推送了若干更新，那你的推送操作就会被驳回。你必须先把他们的更新抓取到本地，合并到自己的项目中，然后才可以再次推送

#### 查看远程仓库信息

我们可以通过命令 `git remote show [remote-name]` 查看某个远程仓库的详细信息

	$ git remote show origin
	* remote origin
	  Fetch URL: https://github.com/lwz0316/tutorial.git
	  Push  URL: https://github.com/lwz0316/tutorial.git
	  HEAD branch: master
	  Remote branch:
	    master tracked
	  Local branch configured for 'git pull':
	    master merges with remote master
	  Local ref configured for 'git push':
	    master pushes to master (up to date)

它友善地告诉你如果是在 `master` 分支，就可以用 `git pull` 命令抓取数据合并到本地。另外还列出了所有处于跟踪状态中的远端分支。

#### 远程仓库的删除和重命名
想修改远程仓库在本地的别名，可以使用下面的命令

	git remote rename [old name] [new name]

对远程仓库的重命名，也会使对应的分支名称发生变化，比如原来的 origin/master 分支现在成了 ppdai/master

删除远程仓库，可以使用命令 

	git remote rm [name]

### 打标签
人们在发布某个软件版本（比如 v1.0 等等）的时候，经常这么做。

#### 列显已有的标签
列出现有标签的命令非常简单
	
	git tag


我们可以用特定的搜索模式列出符合条件的标签

	$ git tag -l 'v1.*'
	v1.1
    v1.2

#### 新建标签
Git 的标签分两种

- 轻量级的（lightweight）。轻量级标签就像是个不会变化的分支，实际上它就是个指向特定提交对象的引用。

	轻量级标签实际上就是一个保存着对应提交对象的校验和信息的文件,要创建它，直接

		$ git tag v1.4-lw


- 含附注的（annotated, **推荐使用**）。含附注标签，实际上是存储在仓库中的一个独立对象，它有自身的校验和信息，包含着标签的名字，电子邮件地址和日期，以及标签说明，标签本身也允许使用 GNU Privacy Guard (GPG) 来签署或验证。

	创建一个含附注类型的标签非常简单

		$ git tag -a v1.4 -m 'release 1.4'

	用 `-a` （译注：取 annotated 的首字母）指定标签名字即可,`-m` 选项则指定了对应的标签说明


#### 后期加注标签
可以在后期对早先的某次提交加注标签

只要在创建标签的命令后面加上对应提交对象的校验值（SHA-1）即可（前几位字符就可以了）

	$ git tag -a v1.2 3cbfa1b

#### 分享标签

默认情况下，`git push` 并不会把标签传送到远端服务器上，只有通过显式命令才能分享标签到远端仓库。

	git push [remote-name] [tag-name]

如果要一次推送所有本地新增的标签上去，可以使用 --tags 选项：

	git push [remote-name] --tags

现在，其他人克隆共享仓库或拉取数据同步后，也会看到这些标签。


## Git 分支


## Git 工作流
