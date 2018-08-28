
## 创建用户并设置角色和权限
```
rabbitmqctl add_user root root123

# 在 vhost "/" 下为root设置权限
rabbitmqctl set_permissions -p / root ".*" ".*" ".*"

# 设置为管理员角色
rabbitmqctl set_user_tags root administrator
```



