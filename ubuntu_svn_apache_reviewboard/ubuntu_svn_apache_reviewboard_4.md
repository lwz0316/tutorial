# Ubuntu SVN Apache ReviewBoard 配置 (四)

> 使用 SVN pre-commit hook 来进行自动创建/更新 review request

由于本人没学过 `Python`，故直接使用`Shell`脚本来调用 `rbt post` 命令来实现 review request 的创建/更新功能。

在 `svn` 的 `pre-commit` 与 `post-commit` hook 中本人是比较倾向于 `pre-commit` 的，大家可以根据自己项目的情况进行选择。

## 配置
### pre-commit
1. 进入到服务器的svn 仓库中
	
		$ cd repo
	    $ ls
	    conf  dav  db  format  hooks  locks  README.txt

2. 打开 `pre-commit` 文件

		$ cd hooks
        # 注意，第一次配置 pre-commit 时需要将 pre-commit.tmp 文件名改为 pre-commit 或者直接 创建 pre-commit 文件。
        $ vi pre-commit

        --- pre-commit

			#!/bin/sh

			REPOS="$1"
			TXN="$2"

            # 脚本的路径可以自己指定
			RPS=/home/svn/repo/hooks/svn-rbt-post-script.sh
			$RPS $REPOS $TXN
			exit $?

        --- save & close

### 自动执行 rbt 命令 的脚本
1. 在 hook 中创建 `svn-rbt-post-script.sh` 文件
	
		$ touch svn-rbt-post-script.sh
		$ vi svn-rbt-post-script.sh

		--- svn-rbt-post-script.sh
			
			#!/bin/bash
			
			# SVN PRE-COMMIT HOOK RBT POST SCRIPT
			#
			# 该脚本用在 pre-commit 中，功能是使用 rbt post 命令向 ReviewBoard 提交 Diff文件，创建 Review Request。
			# 具体的步骤可以划分为以下两步
			#  1. 获取 提交的 log, 通过对 log 的解析，可以得到待执行的命令，log 的格式定义如下：
			#    [request[:${id}]|review:${id}] [summary]
			#      - request 表示创建或者更新，如果后面没有 id, 则表示创建，否则表示更新
			#      - review 表示id对应的 review request 已经通过(ship)了，id 不能为空
			#      - summary 表示该review request 的摘要，当类型为创建时不能为空
			#       NOTE: summary 之前必须要有一个空格
			#      eg.
			#          svn commit -m "request create new review request." # 表示新建一个review request
			#          svn commit -m "request:33 update exist review request by id 33." # 表示更新 id 为 33 的 review request
			#          svn commit -m "review:33 review request was 'ship it', and can be commit the change to subversion." # 表示 review request 已经通过了，将代码改变提交到 svn 中
			#   2. 通过 rbt post 命令创建或者更新 reveiw request。如果是 review 命令，
			#    则直接执行 strict_review $REPOS $TXN 让 svn 入库
			#
			# 以下参数是 pre-commit 中自带的参数，直接拿过来使用
			#   [1] REPOS-PATH   (the path to this repository)
			#   [2] TXN-NAME     (the name of the txn about to be committed)
			
			REPOS="$1"
			TXN="$2"
			
			# 声明 svnlook 命令
			SVNLOOK=/usr/bin/svnlook
			
			# 通过 svnlook 获取提交的 log 信息
			LOG=`$SVNLOOK log -t "$TXN" "$REPOS"`
			
			# 声明 svn diff 文件缓存目录
			SVN_DIFF_DIR=/tmp/svn/diff
			
			# 声明 rbt 命令
			RBT=/usr/local/bin/rbt
			
			# import conf varible
			source /home/svn/repo/conf/svn_rbt.conf
			
			cd $WORKSPACE
			
			#### @begin functions
			
			# 用指定的分割符分割字符串，返回一个数组字符串，以空格为分隔符
			# eg.
			#   result=$(split "a,b" ",")
			#   arr=($result)
			#   echo ${#arr[*]} # => 2
			function split()
			{
			  OLD_IFS="$IFS"
			  IFS=$2
			  arr=($1)
			  IFS="$OLD_IFS"
			  echo ${arr[@]}
			}
			
			function get_id_in_cmd()
			{
			  arr=($(split $1 ":"))
			  if [ ${#arr[@]} -gt 1 ]; then
			    echo ${arr[1]}
			  else
			    echo "-1"
			  fi
			}
			
			# 获取 log 中的命令部分
			# 参数为 log 信息
			function get_cmd()
			{
			  # 用一个取巧的方法，由于 shell 的参数是通过 空格 分割的，故直接返回第一个参数即可
			  echo $1
			}
			
			# 获取 log 中的 msg 部分
			# 参数为 log 信息
			function get_msg()
			{
			   arr=($1)
			   len=${#arr[@]}
			   echo ${arr[@]:1:$len}
			}
			
			# 校验 log msg 的正确性
			# 参数为 log 的 msg 部分
			# 返回 0-正确; 1-错误
			function validate_log_msg()
			{
			  len=`echo $1 | grep "[a-zA-Z0-9]" | wc -c`
			  if [ "$len" -lt 5 ]; then
			    echo -e "\nOops! you must input more than 5 chars as comment!." 1>&2
			    return 1
			  fi
			  return 0
			}
			
			# 分割 命令和 id, 格式为 cmd:id
			# 若 id 不存在，则默认为 -1
			# 参数为 cmd:id
			# 结果为 (cmd id) 数组
			function split_cmd_and_id()
			{
			  cmd_id=($(split $1 ":"))
			  cmd=${cmd_id[0]}
			
			  if [ ${#cmd_id[@]} -gt 1 ]; then
			    id=${cmd_id[1]}
			  else
			    id="-1"
			  fi
			  echo "$cmd $id"
			}
			
			# 创建本次上传的 svn diff 文件
			# 该文件用来提交给 ReviewBoard
			# 参数为 diff 文件名，默认路径为 $SVN_DIFF_DIR
			function create_svn_diff_file()
			{
			  if [ ! -d $SVN_DIFF_DIR ]; then
			    # 文件夹不存在
			    mkdir -p $SVN_DIFF_DIR
			  fi
			
			  diff_file="$SVN_DIFF_DIR/$1"
			
			  # 获取 svn diff 信息，并写入 diff_file
			  $SVNLOOK diff -t "$TXN" "$REPOS" > $diff_file
			
			  echo $diff_file
			}
			
			# 新建一个 review request
			# 调用 rbt post 命令
			function create_review_request()
			{
			  log_msg=`get_msg "$LOG"`
			  validate_log_msg "$log_msg"
			  if [ "$?" = "1" ]; then
			    return 1
			  fi
			
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  summary=`get_msg "$LOG"`
			    return 1
			  fi
			
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  summary=`get_msg "$LOG"`
			  $RBT post --server "$RB_SERVER" \
			            --cache-location $RB_CACHE_LOCATION \
			            --summary "$summary" \
			            --description "$summary" \
			            --diff-filename $diff_file \
			            --username $RB_USERNAME \
			            --password $RB_PWD \
			            --submit-as $svn_author \
			            --svn-username $SVN_USERNAME \
			            --svn-password $SVN_PWD \
			            --repository $SVN_REPO \
			            --repository-url $SVN_REPO_URL \
			            $PUBLISH \
			            $DEBUG
			}
			
			# 新建一个 review request
			# 调用 rbt post 命令
			function update_review_request()
			{
			  r_id=$1
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  desc=`get_msg "$LOG"`
			  $RBT post --server "$RB_SERVER" \
			            --cache-location $RB_CACHE_LOCATION \
			            --diff-filename $diff_file \
			            --change-description "$desc" \
			            --username $RB_USERNAME \
			            --password $RB_PWD \
			            --submit-as $svn_author \
			            --svn-username $SVN_USERNAME \
			            --svn-password $SVN_PWD \
			            --repository $SVN_REPO \
			            --repository-url $SVN_REPO_URL \
			            -r $r_id \
			            $PUBLISH \
			
			# 新建一个 review request
			# 调用 rbt post 命令
			function create_review_request()
			{
			  log_msg=`get_msg "$LOG"`
			  validate_log_msg "$log_msg"
			  if [ "$?" = "1" ]; then
			    return 1
			  fi
			
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  summary=log_msg
			  $RBT post --server "$RB_SERVER" \
			            --cache-location $RB_CACHE_LOCATION \
			            --summary "$summary" \
			            --description "$summary" \
			            --diff-filename $diff_file \
			            --username $RB_USERNAME \
			            --password $RB_PWD \
			            --submit-as $svn_author \
			            --svn-username $SVN_USERNAME \
			            --svn-password $SVN_PWD \
			            --repository $SVN_REPO \
			            --repository-url $SVN_REPO_URL \
			            $PUBLISH \
			            $DEBUG
			}
			
			
			# 分割 命令和 id, 格式为 cmd:id
			# 若 id 不存在，则默认为 -1
			# 参数为 cmd:id
			# 结果为 (cmd id) 数组
			function split_cmd_and_id()
			{
			  cmd_id=($(split $1 ":"))
			  cmd=${cmd_id[0]}
			
			  if [ ${#cmd_id[@]} -gt 1 ]; then
			    id=${cmd_id[1]}
			  else
			    id="-1"
			  fi
			  echo "$cmd $id"
			}
			
			# 创建本次上传的 svn diff 文件
			# 该文件用来提交给 ReviewBoard
			# 参数为 diff 文件名，默认路径为 $SVN_DIFF_DIR
			function create_svn_diff_file()
			{
			  if [ ! -d $SVN_DIFF_DIR ]; then
			    # 文件夹不存在
			    mkdir -p $SVN_DIFF_DIR
			  fi
			
			  diff_file="$SVN_DIFF_DIR/$1"
			 
			  # 获取 svn diff 信息，并写入 diff_file
			  $SVNLOOK diff -t "$TXN" "$REPOS" > $diff_file
			
			  echo $diff_file
			}
			
			# 新建一个 review request
			# 调用 rbt post 命令
			function create_review_request()
			{
			  log_msg=`get_msg "$LOG"`
			  validate_log_msg "$log_msg"
			  if [ "$?" = "1" ]; then
			    return 1
			  fi
			
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  summary=log_msg
			  $RBT post --server "$RB_SERVER" \
			            --cache-location $RB_CACHE_LOCATION \
			            --summary "$summary" \
			            --description "$summary" \
			            --diff-filename $diff_file \
			            --username $RB_USERNAME \
			            --password $RB_PWD \
			            --submit-as $svn_author \
			            --svn-username $SVN_USERNAME \
			            --svn-password $SVN_PWD \
			            --repository $SVN_REPO \
			            --repository-url $SVN_REPO_URL \
			            $PUBLISH \
			            $DEBUG
			}
			
			# 更新一个 review request
			# 调用 rbt post 命令
			# 参数 review id
			function update_review_request()
			{
			  log_msg=`get_msg "$LOG"`
			  validate_log_msg "$log_msg"
			  if [ "$?" = "1" ]; then
			    return 1
			  fi
			
			  r_id=$1
			  svn_author=`$SVNLOOK author -t "$TXN" "$REPOS"`
			  diff_file=`create_svn_diff_file "$svn_author.diff"`
			  desc=log_msg
			  $RBT post --server "$RB_SERVER" \
			            --cache-location $RB_CACHE_LOCATION \
			            --diff-filename $diff_file \
			            --change-description "$desc" \
			            --username $RB_USERNAME \
			            --password $RB_PWD \
			            --submit-as $svn_author \
			            --svn-username $SVN_USERNAME \
			            --svn-password $SVN_PWD \
			            --repository $SVN_REPO \
			            --repository-url $SVN_REPO_URL \
			            -r $r_id \
			            $PUBLISH \
			            $DEBUG
			}
			
			function commit_review()
			{
			  log_msg=`get_msg "$LOG"`
			  validate_log_msg "$log_msg"
			  if [ "$?" = "0" ]; then
			    strict_review $REPOS $TXN
			    return $?
			  fi
			  return 1
			}
			
			# 校验 log 中的命令格式是否正确
			# 参数为 log 的命令
			function exec_cmd()
			{
			  cmd_id=($(split_cmd_and_id $1))
			  cmd=${cmd_id[0]}
			  id=${cmd_id[1]}
			
			  if [ "request" = "$cmd"  ]; then
			    if [ "-1" = "$id" ]; then
			      # 创建一个新的 review 请求
			      create_review_request
			      if [ "$?" = "0" ]; then
			        echo -e "\ncreate review request [success]" 1>&2
			      fi
			    else
			      # 更新已经存在的 review 请求
			      update_review_request $id
			      if [ "$?" = "0" ]; then
			        echo -e "\nupdate review request [success]" 1>&2
			      fi
			    fi
			
			  elif [ "review" = "$cmd" ]; then
			    if [ "$id" = "-1" ]; then
			      echo -e "invalidate review id" 1>&2
			    else
			      commit_review
			      return $?
			    fi
			
			  else
			    echo -e "invalidate cmd" 1>&2
			  fi
			
			  return 1
			}
			
			##### @end functions
			
			CMD=`get_cmd $LOG`
			exec_cmd $CMD


        --- save & close

2. 为了使得脚本可以进行配置，故将其中的一些路径和其他的参数抽离出来，放在 `conf/svn_rbt.conf`中

		$ touch ../conf/svn_rbt.conf
		$ vi ../conf/svn_rbt.conf

        --- svn_rbt.conf

			# workspace
			# 默认用户文件夹，rbt 需要的一些缓存文件都是放在这里面的
			WORKSPACE=~
			
			# ReviewBoard server url. for --server
			RB_SERVER=http://review.xxx.com
			
			# ReviewBoard access username & password. for --username & --password
			RB_USERNAME=admin
			RB_PWD=admin
			
			# ReviewBoard cache location. for --cache-location
			RB_CACHE_LOCATION=.cache/rbtools/apicache.db
			
			# SVN access username & password. for --svn-username & --svn-password
			SVN_USERNAME=guest
			SVN_PWD=guest
			
			# SVN repository & repository url. for --repository & --repository-url
			SVN_REPO=repo
			SVN_REPO_URL=http://review.xxx.com/svn
			
			# rbt debug
			DEBUG=-d
			#DEBUG= 
			
			# rbt publish immediately
			PUBLISH=-p
			#PUBLISH=

        --- save & close

   