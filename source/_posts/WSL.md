---
title: WSL
tags: 
categories:
  - [Tools]
---



```
net stop LxssManager 停止子系统服务
net start LxssManager 开启子系统服务
```





---
### linux 查看进程相关参数的命令

`lsof -i:3306` 查看3306端口

`netstat -tap |grep mysql` 查看mysql进程占用的端口

---
### zsh
1. 安装 `sudo apt-get install zsh`
2. 设置默认shell `chsh -s /bin/zsh`，重启(WSL)生效
> 查看当前默认shell `echo $SHELL`
> 查看已安装shell `cat /etc/shells`



---
### oh-my-zsh
> 先安装git `apt`
> 1. 安装，去官网下载install.sh ~~已经copy了官网脚本：[oh-my-zsh   install.sh~~](http://47.94.212.95:20000/upload/2020/10/oh-my-zsh-install-486bace8f443481b98be1f9d4700f222.sh)
> `  sh -c "$(curl -fsSL http://47.94.212.95:20000/upload/2020/10/oh-my-zsh-install-486bace8f443481b98be1f9d4700f222.sh)"  `
2. 主题列表
`https://github.com/ohmyzsh/ohmyzsh/wiki/Themes-%28legacy%29`
3. oh-my-zsh的配置文件: `~/.zshrc`, 下方有模板
   
   > `ZSH_THEME="agnoster"`
4. 插件
   > 语法高亮,官方md
   > `https://github.com/zsh-users/zsh-syntax-highlighting/blob/master/INSTALL.md`
   > 1. (找个路径)下载插件
   > `git clone https://github.com/zsh-users/zsh-syntax-highlighting.git`
   >
   > 2. 编辑 ~/.zshrc
   > `echo "source ${(q-)PWD}/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh" >> ${ZDOTDIR:-$HOME}/.zshrc`
   >
   > 3. source生效
   > `source ./zsh-syntax-highlighting/zsh-syntax-highlighting.zsh`

   > 命令补全plugin
   > 1. zsh-autosuggestions
   > > 1. `git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions`
   > > 2. 在~/.zshrc添加插件：`plugins=(git zsh-autosuggestions)`
   > > 3. 重启终端
   > 2. incr
   > > ![incr](http://47.94.212.95:20000/upload/2020/10/image-664ef5bfb6a7471ea0eafbac580441b7.png)
   > > 1. `wget http://mimosa-pudica.net/src/incr-0.2.zsh    ~/.oh-my-zsh/plugins/incr/incr*.zsh`
   > > 2. 在~/.zshrc添加：`source ~/.oh-my-zsh/plugins/incr/incr*.zsh`
   > > 3. 重启终端



---
### 更换apt国内镜像

1. 备份原配置文件

   `sudo cp /etc/apt/sources.list /etc/apt/sources.list.def`

2. 编辑配置文件

   `sudo vim /etc/apt/sources.list`

   > deb http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse
   > deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
   > deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
   > deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
   > deb http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
   > deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial main restricted universe multiverse
   > deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-security main restricted universe multiverse
   > deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-updates main restricted universe multiverse
   > deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-proposed main restricted universe multiverse
   > deb-src http://mirrors.ustc.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse

   > deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
   > deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
   > deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
   > deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
   > deb http://mirrors.aliyun.com/ubuntu/ xenial universe
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
   > deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
   > deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
   > deb http://archive.canonical.com/ubuntu xenial partner
   > deb-src http://archive.canonical.com/ubuntu xenial partner
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
   > deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
   > deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse

   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial main restricted
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates main restricted
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial universe
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates universe
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial multiverse
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-updates multiverse
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-backports main restricted universe multiverse
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security main restricted
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security universe
   > deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu/ xenial-security multiverse

   > deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
   > deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
   > deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
   > deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
   > deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
   > deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
   > deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
   > deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
   > deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
   > deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse

3. 更新软件包

   `sudo apt-get update`

4. 修复损坏的软件包，尝试卸载出错的软件包，重装正确版本

   `sudo apt-get -f install`

5. 升级系统中所有软件包

   `sudo apt-get -y upgrade`

---
### ~/.zshrc模板
```
# If you come from bash you might have to change your $PATH.
# export PATH=$HOME/bin:/usr/local/bin:$PATH

# Path to your oh-my-zsh installation.
  export ZSH="/home/yeyuntian/.oh-my-zsh"

# Set name of the theme to load --- if set to "random", it will
# load a random theme each time oh-my-zsh is loaded, in which case,
# to know which specific one was loaded, run: echo $RANDOM_THEME
# See https://github.com/robbyrussell/oh-my-zsh/wiki/Themes
ZSH_THEME="robbyrussell"

# Set list of themes to pick from when loading at random
# Setting this variable when ZSH_THEME=random will cause zsh to load
# a theme from this variable instead of looking in ~/.oh-my-zsh/themes/
# If set to an empty array, this variable will have no effect.
# ZSH_THEME_RANDOM_CANDIDATES=( "robbyrussell" "agnoster" )

# Uncomment the following line to use case-sensitive completion.
# CASE_SENSITIVE="true"

# Uncomment the following line to use hyphen-insensitive completion.
# Case-sensitive completion must be off. _ and - will be interchangeable.
# HYPHEN_INSENSITIVE="true"

# Uncomment the following line to disable bi-weekly auto-update checks.
# DISABLE_AUTO_UPDATE="true"

# Uncomment the following line to change how often to auto-update (in days).
# export UPDATE_ZSH_DAYS=13

# Uncomment the following line to disable colors in ls.
# DISABLE_LS_COLORS="true"

# Uncomment the following line to disable auto-setting terminal title.
# DISABLE_AUTO_TITLE="true"

# Uncomment the following line to enable command auto-correction.
# ENABLE_CORRECTION="true"

# Uncomment the following line to display red dots whilst waiting for completion.
# COMPLETION_WAITING_DOTS="true"

# Uncomment the following line if you want to disable marking untracked files
# under VCS as dirty. This makes repository status check for large repositories
# much, much faster.
# DISABLE_UNTRACKED_FILES_DIRTY="true"

# Uncomment the following line if you want to change the command execution time
# stamp shown in the history command output.
# You can set one of the optional three formats:
# "mm/dd/yyyy"|"dd.mm.yyyy"|"yyyy-mm-dd"
# or set a custom format using the strftime function format specifications,
# see 'man strftime' for details.
# HIST_STAMPS="mm/dd/yyyy"

# Would you like to use another custom folder than $ZSH/custom?
# ZSH_CUSTOM=/path/to/new-custom-folder

# Which plugins would you like to load?
# Standard plugins can be found in ~/.oh-my-zsh/plugins/*
# Custom plugins may be added to ~/.oh-my-zsh/custom/plugins/
# Example format: plugins=(rails git textmate ruby lighthouse)
# Add wisely, as too many plugins slow down shell startup.
plugins=(
  git
)

source $ZSH/oh-my-zsh.sh

# User configuration

# export MANPATH="/usr/local/man:$MANPATH"

# You may need to manually set your language environment
# export LANG=en_US.UTF-8

# Preferred editor for local and remote sessions
# if [[ -n $SSH_CONNECTION ]]; then
#   export EDITOR='vim'
# else
#   export EDITOR='mvim'
# fi

# Compilation flags
# export ARCHFLAGS="-arch x86_64"

# ssh
# export SSH_KEY_PATH="~/.ssh/rsa_id"

# Set personal aliases, overriding those provided by oh-my-zsh libs,
# plugins, and themes. Aliases can be placed here, though oh-my-zsh
# users are encouraged to define aliases within the ZSH_CUSTOM folder.
# For a full list of active aliases, run `alias`.
#
# Example aliases
# alias zshconfig="mate ~/.zshrc"
# alias ohmyzsh="mate ~/.oh-my-zsh"
```



## install mysql

安装`sudo apt install mysql-server-5.7`

连接`mysql -p`

密码`123456`

添加数据库`create database melopozDB`

查看db`show databases`

配置文件在

mysqld 在 `/etc/mysql/mysql.conf.d/mysqld.cnf`

mysql   在 `/etc/mysql/conf.d/mysql.cnf`

client    在 `/etc/mysql/debian.cnf`

设置默认字符集`utf8mb4`

> 在 【client】下追加：
>
> default-character-set=utf8
>
> 在 【mysqld】下追加：
>
> character-set-server=utf8
>
> 在 【mysql】 下追加：
>
> default-character-set=utf8

