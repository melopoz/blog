---
title: git双密钥
tags: git双密钥
categories: other
date: 2021/09/04 18:00
updated: 2021/09/04 18:00
---



# 个人机器配置双秘钥登录github和企业gitlab



1. 个人电脑创建秘钥，并将公钥添加到github/gitlab

   - 生成密钥到指定路径：`ssh-keygen -t rsa -C 1329340043@qq.com -f ~/.ssh/id_rsa_github`
   - 将公钥添加到github/gitlab：`cat ~/.ssh/id_rsa_github.pub`，https://github.com/settings/keys，【new SSH key】

   <img src="https://raw.githubusercontent.com/melopoz/pics/master/img/761630753693_.pic_hd.jpg" style="zoom:50%;" />

2. 配置config文件，指定git域名和ssh认证时使用的秘钥

   Host和HostName一致

   ```
   Host github.com
   HostName github.com
   User 登录邮箱
   IdentityFile /Users/didi/.ssh/id_rsa_github
   
   Host git.xiaojukeji.com
   HostName git.xiaojukeji.com
   User 登录邮箱
   IdentityFile /Users/didi/.ssh/id_rsa
   ```

3. 测试ssh

   `ssh -T git@github.com`，`ssh -T git@公司git域名`

