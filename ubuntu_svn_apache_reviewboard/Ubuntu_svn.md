# Ubuntu SVN 配置

> Ubuntu 环境下搭建 Subversion

## Install

	$sudo apt-get install subversion

## Config

1. 创建一个仓库

		$ mkdir /home/svn/repo

2. 初始化一个仓库

		$ svnadmin create /home/svn/repo
		
		$ ls /home/svn/repo
		conf  db  format  hooks  locks  README.txt

3. 配置

		$ cd /home/svn/repo/conf
		$ vi svnserve.conf

		---- svnserve.conf 文件修改如下
		
				# anon-access = read
				# auth-access = write
				# password-db = passwd
				# authz-db = authz
				# realm = My First Repository
		
				改为（注意 # 后面的空格要去掉）
				
				anon-access = none   # 匿名用户访问 = 否
				auth-access = write  # 登录用户访问 = 读
				password-db = passwd # 密码配置文件 = passwd 文件
				authz-db = authz     # 授权配置文件 = authz
				realm = ubuntu_svn   # 认证域，同一个认证域采用同一套 authz 和 passwd
		---- 保存并关闭 svnserve.conf

		$ vi authz

		---- authz 文件修改如下

				[groups]
				admin = admin
				dev_android = android1,android2
				dev_ios = ios1,ios2
				guest = guest

				[/]
				@admin = rw
				* = r
				
				[/android]
				@dev_android = rw
				
				[/ios]
				@dev_ios = rw
		---- 保存并关闭 authz	

		$vi passwd
			
		---- passwd 文件修改如下

		[users]
		admin = admin
		android1 = 12345678
		android2 = 12345678
		ios1 = 12345678
		ios2 = 12345678
		guest = guest
		---- 保存并关闭 passwd

4. 重启服务

		$ killall svnserve # kill 掉所有的 svn 服务
		$ svnserve -d -r /home/svn/repo # 启动 svn 服务

5. 查看仓库信息

		$ svn info svn://localhost/
		Path: .
		URL: svn://localhost
		Relative URL: ^/
		Repository Root: svn://localhost

	仓库成功创建~~~

6. 设置提交强制输入注释

	参考 [强制svn 提交代码者 添加注释信息](http://blog.chinaunix.net/uid-25525723-id-371708.html)



		
	

		



	
	

	