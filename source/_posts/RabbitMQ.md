---
title: RabbitMQ
tags: 
categories:
  - [MQ]
---





查看所有的队列：`rabbitmqctl list_queues`

清除所有的队列：`rabbitmqctl reset`

---

停止：`service rabbitmq-server stop`

启动：`service rabbitmq-server start`

重启:   `service rabbitmq-server restart`

查看状态：`service rabbitmq-server status`

关闭应用：`rabbitmqctl stop_app`

启动应用：`rabbitmqctl start_app`

查看插件打开情况：`rabbitmq-plugins list`

启动监控管理器：`rabbitmq-plugins enable rabbitmq_management`

关闭监控管理器：`rabbitmq-plugins disable rabbitmq_management`

查看状态：`rabbitmqctl status`

集群同步：

> 所有节点的值相同：`/var/lib/rabbitmq/.erlang.cookie`
>
> 加入集群：
>
> host1 和 host2，在 host2 上操作
>
> 先停止：`rabbitmqctl -n rabbit stop_app`
>
> 加入：`rabbitmqctl -n rabbit join_cluster rabbit@$rabbit_hostname1`
>
> 再启动：`rabbitmqctl -n rabbit start_app`

查看集群状态：`rabbitmqctl cluster_status`

多应用使用  `rabbitmqctl -n rabbit_ceilometer`





#### user相关命令

- 创建用户

  `rabbitmqctl add_user {用户名} {密码}`

- 设置权限

  `rabbitmqctl set_user_tags {用户名} {权限}`

- 查看用户列表

  `rabbitmqctl list_users`

- 给用户授权

  `rabbitmqctl  set_permissions -p vhost1 user1 '.*' '.*' '.*' `

  > demo: `rabbitmqctl  set_permissions -p / admin '.*' '.*' '.*' `

- 查看权限

  `rabbitmqctl list_user_permissions user1`

  `rabbitmqctl list_permissions -p vhost1`

- 清除权限

  `rabbitmqctl clear_permissions [-p VHostPath] User`

- 删除用户

  `rabbitmqctl delete_user Username`

- 修改密码

  `rabbitmqctl change_password Username Newpassword`

- 清除用户权限

  `rabbitmqctl  clear_permissions  [-p VHostPath]  admin`



#### 用户角色分类  (权限列表)

- **none**：无法登录控制台

  不能访问 management plugin，通常就是普通的生产者和消费者。

- **management**：普通管理者

  仅可登陆管理控制台(启用management plugin的情况下)，无法看到节点信息，也无法对policies进行管理。

  用户可以通过AMQP做的任何事外加： 

  1. 列出自己可以通过AMQP登入的virtual hosts
  2. 查看自己的virtual hosts中的queues, exchanges 和 bindings
  3. 查看和关闭自己的channels 和 connections
  4. 查看有关自己的virtual hosts的“全局”的统计信息，包含其他用户在这些virtual hosts中的活动。

- **policymaker**：策略制定者

  management可以做的任何事外加：

  1. 查看、创建和删除自己的virtual hosts所属的policies和parameters

- **monitoring**：监控者

  management可以做的任何事外加：

  1. 列出所有virtual hosts，包括他们不能登录的virtual hosts
  2. 查看其他用户的connections和channels
  3. 查看节点级别的数据如clustering和memory使用情况
  4. 查看真正的关于所有virtual hosts的全局的统计信息
  5. 同时可以查看rabbitmq节点的相关信息(进程数，内存使用情况，磁盘使用情况等)

- **administrator**：超级管理员

  policymaker和monitoring可以做的任何事外加:

  1. 创建和删除virtual host
  2. 查看、创建和删除users
  3. 查看创建和删除permissions
  4. 关闭其他用户的connections

