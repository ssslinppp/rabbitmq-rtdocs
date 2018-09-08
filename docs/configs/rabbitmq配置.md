# rabbitmq配置

[RabbitMQ Configuration](https://www.rabbitmq.com/configure.html)  

## 环境变量
- `RABBITMQ_`前缀（SHELL变量）: ；
- `rabbitmq-env.conf`配置文件：`$RABBITMQ_HOME/etc/rabbitmq/目录`；
- 优先级：`RABBITMQ_` > `rabbitmq-env.conf` > `默认配置`

[Customise RabbitMQ Environment](https://www.rabbitmq.com/configure.html#customise-environment)     
[RabbitMQ Environment Variables](https://www.rabbitmq.com/configure.html#define-environment-variables)    

默认配置位置：`$RABBITMQ_HOME/sbin/rabbitmq-defaults`
```
[root@rmq-node1 rabbitmq]# rpm -ql rabbitmq-server |grep defaults
/usr/lib/rabbitmq/bin/rabbitmq-defaults
/usr/lib/rabbitmq/lib/rabbitmq_server-3.6.12/sbin/rabbitmq-defaults
```
默认配置文件：
```
#!/bin/sh -e
##  The contents of this file are subject to the Mozilla Public License
##  Version 1.1 (the "License"); you may not use this file except in
##  compliance with the License. You may obtain a copy of the License
##  at http://www.mozilla.org/MPL/
##
##  Software distributed under the License is distributed on an "AS IS"
##  basis, WITHOUT WARRANTY OF ANY KIND, either express or implied. See
##  the License for the specific language governing rights and
##  limitations under the License.
##
##  The Original Code is RabbitMQ.
##
##  The Initial Developer of the Original Code is GoPivotal, Inc.
##  Copyright (c) 2012-20
15 Pivotal Software, Inc.  All rights reserved.
##

### next line potentially updated in package install steps
SYS_PREFIX=

### next line will be updated when generating a standalone release
ERL_DIR=

CLEAN_BOOT_FILE=start_clean
SASL_BOOT_FILE=start_sasl

if [ -f "${RABBITMQ_HOME}/erlang.mk" ]; then
    # RabbitMQ is executed from its source directory. The plugins
    # directory and ERL_LIBS are tuned based on this.
    RABBITMQ_DEV_ENV=1
fi

## Set default values

BOOT_MODULE="rabbit"

CONFIG_FILE=${SYS_PREFIX}/etc/rabbitmq/rabbitmq
LOG_BASE=${SYS_PREFIX}/var/log/rabbitmq
MNESIA_BASE=${SYS_PREFIX}/var/lib/rabbitmq/mnesia
ENABLED_PLUGINS_FILE=${SYS_PREFIX}/etc/rabbitmq/enabled_plugins

PLUGINS_DIR="${RABBITMQ_HOME}/plugins"

# RABBIT_HOME can contain a version number, so default plugins
# directory can be hard to find if we want to package some plugin
# separately. When RABBITMQ_HOME points to a standard location where
# it's usually being installed by package managers, we add
# "/usr/lib/rabbitmq/plugins" to plugin search path.
case "$RABBITMQ_HOME" in
    /usr/lib/rabbitmq/*)
        PLUGINS_DIR="/usr/lib/rabbitmq/plugins:$PLUGINS_DIR"
        ;;
esac

CONF_ENV_FILE=${SYS_PREFIX}/etc/rabbitmq/rabbitmq-env.conf  # 指定了配置文件的位置

```


## 配置文件

## 运行时参数和策略
