---
title: 鱼人宝宝都能看懂的Git知识
date: 2023-09-02 16:06:03
categories: 知识
tags: [Git]
---
## 闲言碎语

之前的工作中基本没有用到过Git，所有代码都是通过SaaS平台编辑管理，在上家T司本来想着后续推动一下用Git管理但是没机会，好在现在这家NT司就是用Git来管理项目的。

常用的操作比如博客管理就是用到了add、commit、push，至于里边的公钥配置啥的原理以及分支管理都搞得不太明白，刚好有机会搞一下

<!--more-->

## Git

实际工作中会遇到关于文件备份、代码还原、协同开发、追溯代码问题和负责人等场景，之前的解决方案是类似于SVN\CVS之类的集中式版本控制工具

集中式版本控制工具会将版本库集中存放在中央服务器中，team里每个人工作时从中央服务器下载代码，必须联网才能工作，个人修改后提交到中央版本库

比集中式更进一步的是分布式版本控制工具，即Git，它没有中央服务器的概念，每个人的电脑上都是一个完整的版本库，无需联网就能工作，多人协作时只需要将各自的修改推送给对方，就能互相看到对方的修改

日常经常会有一个远程仓库来托管作为最终线上部署的内容，开发人员从远程仓库拉取到本地进行开发，最终再合并到远程仓库上

## 工作流程

<img src="/images/git1.png" width="50%" height="50%">

Clone - 从远程仓库中克隆代码到本地仓库

Checkout - 从本地仓库中检出一个分支然后进行修订

Add - 在提交前现将代码提交到暂存区

Commit - 提交到本地仓库，本地仓库中保存修改的各个历史版本

Fetch - 从远程仓库抓取到本地仓库，不进行任何的合并动作

Pull - 从远程仓库拉取到本地仓库，自动进行合并Merge，然后放到工作区，相当于Fetch+Merge

Push - 从本地仓库到远程仓库

## 常用命令

### 基本环境配置

``` bash
git config --global user.name "username"  # 设置
git config --global user.email "no@thanks.com"
git config --global user.name  # 查看
git config --global user.email

git log --pretty=oneline --all --graph --abbrev-commit  # 输出提交日志

# 解决Git Bash中文乱码问题
git config --global core.quotepath false
# 在${git_home}/etc/bash 的.bashrc文件中加入
# export LANG="zh_CN.UTF-8"
# export LC_ALL="zh_CN.UTF-8"
```

### 本地仓库

``` bash
# 在任意一个文件夹下
git init  # 初始化当前目录为一个git仓库
# 执行后文件夹下出现.git文件
```

### 文件的基本状态

Git工作目录（本地仓库所在目录）下对于文件的修改（增加、删除、更新）并执行Git命令的过程中，文件会处于不同的状态：

<img src="/images/git2.png" width="50%" height="50%">

untracked - 新创建的文件，位于工作区，处于未跟踪状态

staged - untracked状态的文件执行git add，则变更为已暂存状态，即将文件提交到了暂存区

commit - staged状态的文件执行git commit，则变更为已提交状态，即将文件提交到了本地仓库

unstaged - 修改已有的文件，文件位于工作区，处于未暂存状态

### 基础指令

``` bash
git status  # 查看文件状态
git add .  # 将工作区文件存放到暂存区(to be committed)
git commit -m ""  # 将暂存区文件提交到本地仓库
git log  # 查看提交日志
git log --all  # 显示所有分支
git log --pretty=oneline  # 将提交信息显示为一行
git log --abbrev-commit  # 简化commitid
git log --graph  # 图形化呈现

git reset --hard {commitID}  # 版本回退待commitid对应的版本
git reflog  # 查看已经删除的提交记录

# 忽略文件.gitignore
# *.a  # 所有.a结尾文件都不用Git管理
# !lib.a  # 除了lib.a这个文件外的.a结尾文件不用Git管理
# /TODO  # 只忽略当前目录下的TODO文件夹，不忽略子文件
# build/  # 忽略所有build文件夹下的文件
# doc/*.txt  # 忽略doc文件夹下的.txt结尾文件
# doc/**/*.pdf # 忽略doc文件夹下及所有子文件的.pdf结尾文件
```

### 分支指令

``` bash
git branch  # 查看分支
git branch {branchname}  # 新建分支branchname
git checkout {branchname}  # 切换到branchname分支
git checkout -b {branchname}  # 新建分支并切换到新分支branchname
git merge {branchname}  # 将分支branchname合并到当前分支
git branch -d {branchname}  # 删除分支branchname
git branch -D {branchname}  # 强制删除没有merge的分支branchname
```

### 常见开发规范

master分支 - 线上主分支

develop分支 - 从master创建的分支，一般作为开发部门的主要分支，
如果没有其他并行开发不同期上线要求，都可以在此版本上进行开发，
所有人的阶段开发完成后，先合并到develop分支，再合并到master分支上线

feature分支 - 从develop创建的分支，一般是同期并行开发但不同期上线时创建的分支，
分支上的研发任务完成后合并到develop分支

hotfix分支 - 从master创建的分支，一般作为线上BUG修复使用，
修复完成后需要合并到master、develop分支

可能还有其他如test分支（用于代码测试）、pre分支（用于预发布测试）等

## 托管服务

又称为远程仓库，常见有Github、码云Gitee、GitLab，其中Github和Gitee为国外国内的SaaS服务，而GitLab是一个开源项目，可以用于Git私服的搭建

本地仓库连接远程仓库时，可以使用用户名密码方式，也可以使用SSH密钥认证方式（常用）

``` bash
ssh-keygen -t rsa  # 生成客户端SSH密钥
~/.ssh/id_rsa.pub  # 公钥
# 将公钥上传到远程仓库xx@xx.com
ssh -T xx@xx.com  # 验证是否配置成功

git remote add origin xx@xx.com:xx/xx.git  # 添加一个远程仓库
git remote  # 查看已经配置的远程仓库
git push origin master:master  # 将本地仓库master分支推送到远程仓库origin的master分支
# 如果远程分支和本地分支名称相同，可以只写一个
git push --set-upstream origin master:master  # 推送并关联分支
git push  # 已经建立关联的分支可以省略
git push -f  # 强制覆盖

git branch -vv  # 查看本地分支详情（可显示与远程仓库分支的绑定关系）

git clone xx@xx.com:xx/xx.git  # 克隆远程仓库到本地
git fetch {remotename}{branchname}  # 抓取远程分支到本地但并不进行合并
git pull {remotename}{branchname}  # 抓取远程分支到本地并和当前分支合并
```

### SSH

Secure Shell，安全外壳，是一种网络安全协议，通过加密和认证机制实现安全的访问和文件传输等业务。

传统的远程登录和文件传输方式，如Telnet、FTP，使用明文传输数据，存在很多安全隐患。

#### Telnet

Telnet是一种应用层协议，使用于互联网及局域网中，使用虚拟终端的形式，提供双向、以文字字符串为主的命令行接口交互功能。属于TCP/IP协议族的其中之一，是互联网远程登录服务的标准协议和主要方式，常用于服务器的远程控制，可供用户在本地主机运行远程主机上的工作。

#### FTP

File Transfer Protocol，文件传输协议，是在计算机网络的用户端和服务器之间传输文件的应用层协议。FTP的目标时提供文件的共享性，而非直接使用远程计算机，使用TCP传输而非UDP

#### TCP

Transmission Control Protocol，传输控制协议，定义了两台计算机之间进行可靠的传输而交换的数据和确认信息的格式，以及计算机为了确保数据的正确到达而采取的措施。协议规定了TCP软件怎样识别给定计算机上的多个目的进程如何对分组重复这类差错进行恢复。协议还规定了两台计算机如何初始化一个TCP数据流传输以及如何结束这一传输。

#### UDP

User Datagram Protocol，用户数据报协议，是一个简单的面向数据报的传输层协议。提供的是非面向连续的、不可靠的数据流传输。UDP不提供可靠性，也不提供报文到达确认、排序以及流量控制等功能，它知识吧应用程序传给IP层的数据报发送出去，但不能保证它们到达目的地。因此报文可能会丢失、重复以及乱序，但由于UDP在传输数据报前不用在客户和服务器之间建立连接，且没有超时重发等机制，故而传输速度很快。常见的是视频、音乐应用应用的基本都是UDP

### SSH工作方式

SSH由服务器和客户端组成，为了建立安全的SSH通道，双方需要先建立TCP连接，然后协商使用的版本号和各类算法，并生成相同的会话密钥用于后续的对称加密。在完成用户认证后，双方即可建立会话进行数据交互，主要的工作流程包括以下几个阶段：

<img src="/images/git3.png" width="50%" height="50%">

#### 连接建立

SSH依赖端口进行通信，在未建立SSH连接时，SSH服务器会在指定端口（22）侦听连接请求，SSH客户端向服务器该指定端口发起连接请求后，双方建立一个TCP连接，后续通过该端口进行通信

#### 版本协商

SSH协议目前存在SSH1.x和2.0版本，2.0协议相比于1.x来说，在结构上做了扩展，可以支持更多的认证方法和密钥交换方法，同时提高了服务能力

#### 算法协商

SSH工作过程中需要使用多种类型的算法，包括用于产生会话密钥的密钥交换算法、用于数据信息加密的对称加密算法、用于进行数字签名和认证的公钥算法那、用于数据完整性保护的HMAC算法等

#### 密钥交换

SSH服务器和客户端通过密钥交换算法，动态生成共享会话密钥和会话ID，建立加密通道。会话密钥主要用于后续数据传输的加密，会话ID用于在认证过程中标识该SSH连接。

由于SSH服务器和客户端需要持有相同的会话密钥用于后续的对称加密，为了保证密钥交换的安全性，SSH使用一种安全的方式生成会话密钥：

<img src="/images/git4.png" width="50%" height="50%">

1. SSH服务器生成素数G、P、服务器私钥b，并计算得到服务器公钥y=(G^b)%P
2. SSH服务器将素数G、P、服务器公钥y发送给SSH客户端
3. SSH客户端生成客户端私钥a，计算得到客户端公钥x=(G^a)%P
4. SSH客户端将公钥x发送给SSH服务器
5. SSH服务器计算得到对称密钥K=(x^b)%P，SSH客户端计算得到对称密钥K=(y^a)%P，数学定律可以保证两者相同（RSA算法，欧拉定理）

#### 用户认证

SSH客户端向SSH服务器发起认证庆祝，SSH服务器对SSH客户端进行认证，认证方式有：

1. 密码认证：客户端通过用户名密码的方式进行人恩正，将加密后的用户名密码发给服务器，服务器解密后与本地保存的用户名密码进行对比，并向客户端返回认证结果消息
   1. SSH客户端向SSH服务器发送登录请求
   2. SSH服务器将服务器公钥发送给SSH客户端
   3. SSH客户端输入密码，使用服务器公钥加密后发送给SSH服务器
   4. SSH服务器收到密文，使用服务器私钥解密得到密码，验证
   
   如果有人在a步骤中截获SSH客户端发送的登录请求后，冒充SSH服务器将伪造的公钥发送给SSH客户端，就能够获得该用户的密码，所以，在首次登录SSH服务器时，SSH客户端上会提示公钥指纹，并询问用户是否确认登录，用户确认后公钥将被保存并信任，下次访问时，SSH客户端将会核对SSH服务器发来的公钥与本地保存的是否相同。这种方式适用于公布了公钥指纹的SSH服务器以及已登陆过正确SSH服务器的SSH客户端
2. 密钥认证：客户端通过用户名、公钥及公钥算法等信息来与服务器认证
   1. 再进行SSH连接之前，SSH客户端需要先生成自己的公钥私钥对，并将自己的公钥存放在SSH服务器上
   2. SSH客户端向SSH服务器发送登录请求
   3. SSH服务器更具请求中的用户名等信息在本地搜索客户端的公钥，并用这个公钥加密一个随机数发送给客户端
   4. SSH客户端使用自己的私钥对返回的信息进行解密，并发送给SSH服务器
   5. SSH服务器验证SSH客户端解密的信息是否正确
3. 密码-密钥认证：致用户需要同时满足密码和密钥认证才能登录
4. all认证：只要满足密码和密钥认证中的一种即可

#### 会话请求

认证通过后，SSH客户端向服务器发送会话请求，请求服务器提供耨中类型的服务

#### 会话交互

会话建立后，SSH服务器和客户端在该会话上进行数据信息的交互，双发发送的数据均使用会话密钥进行加解密

## IDEA操作

1. 配置Git，File - Settings - Version Control - Git - Path to Git executable：Git应用所在的路径
2. 远程仓库创建
3. 本地仓库初始化，VCS（Version Control） - Import into Version Control - Create Git Repository - 选择本地文件夹
4. 提交，Git - √ - 勾选需要提交的文件 - Commit Message填写 - Commit
5. 查看Log：左下角Version Control - Log
6. 添加远程仓库：VCS - Git - Push - Define_remote改为远程仓库链接
7. 克隆远程仓库：VCS - Checkout from Version Control - Git - 远程仓库链接 - Clone
8. 创建分支：左下角Version Control - Log - 版本中右键new一个分支

<img src="/images/git5.png" width="50%" height="50%">

<img src="/images/git6.png" width="50%" height="50%">





















