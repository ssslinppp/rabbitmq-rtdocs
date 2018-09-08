
## 创建用户并设置角色和权限
```
rabbitmqctl add_user root root123

# 在 vhost "/" 下为root设置权限
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"

# 设置为管理员角色
rabbitmqctl set_user_tags root administrator
```

## 集群状态report
包括各node的status、cluster_status、Connections、Channels、Queues、Exchanges、Bindings、Consumers、Application environment等等
```
rabbitmqctl report > report.txt
```


