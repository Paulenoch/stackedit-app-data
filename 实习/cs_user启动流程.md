好的！我来总结一下启动这个 GAS 项目（cs/user）的完整流程：

## 项目启动完整流程

### 1️⃣ 配置文件设置

修改 etc/user.yml 配置文件中的配置中心（Config Center）配置：

gas.config:

config_center:

connect_timeout_seconds: 5

namespaces:

- group: customer_service

project: cs

# 使用模板格式，支持环境变量替换

namespace: <PROJECT_NAME>_<MODULE_NAME>_<ENV>

non_live_secret: 8550ca1eded00f9da7602643b9dbca64dc5fde409be1311059db92fb0d0c5622

关键点：

-   namespace 使用模板格式：<PROJECT_NAME>_<MODULE_NAME>_<ENV>

-   实际会根据环境变量解析为：cs_user_test（test 环境）

-   如果不需要配置中心，可以注释掉整个 gas.config 配置块

----------

### 2️⃣ 启动 Spex 监听服务（前提条件）

在启动项目前，需要先启动 spex socket 监听：

# 创建目录（如果不存在）

mkdir -p /Users/yingte.dai/run/spex

# 启动 spex 监听（后台运行）

socat -d -d -d UNIX-LISTEN:/Users/yingte.dai/run/spex/spex.sock,reuseaddr,fork \

TCP:agent-tcp.spex.test.shopee.io:9299 &

说明：

-   spex.sock 是 GAS 应用与 Spex 服务通信的 Unix Socket

-   需要将本地 socket 代理到 Spex 服务器

----------

### 3️⃣ 设置必要的环境变量

启动项目需要以下环境变量：

# 环境配置

env=test # 运行环境：test/live

PROJECT_NAME=cs # 项目名称

MODULE_NAME=user # 模块名称

cid=sg # 集群 ID：sg/cn/vn 等

# 服务配置

saas_id=392348641336307168 # SaaS ID（测试环境）

PORT_user=9091 # HTTP 服务端口

INDEX=1 # Snowflake ID 生成器节点索引

# Spex 配置

SP_UNIX_SOCKET=/Users/yingte.dai/run/spex/spex.sock # Spex socket 路径

----------

### 4️⃣ 编译项目（如果需要）

# 进入项目目录

cd /Users/yingte.dai/code/seller-server/cs/user

# 使用 spkit 编译

spkit build

# 或者使用 go build

go build -o bin/user ./cmd/user

----------

### 5️⃣ 启动项目

使用完整的环境变量启动：

env=test \

PROJECT_NAME=cs \

MODULE_NAME=user \

saas_id=392348641336307168 \

SP_UNIX_SOCKET=/Users/yingte.dai/run/spex/spex.sock \

cid=sg \

PORT_user=9091 \

INDEX=1 \

./bin/user

或者使用 spkit（推荐）：

# 设置环境变量

export env=test

export PROJECT_NAME=cs

export MODULE_NAME=user

export saas_id=392348641336307168

export SP_UNIX_SOCKET=/Users/yingte.dai/run/spex/spex.sock

export cid=sg

export PORT_user=9091

export INDEX=1

# 启动

spkit run

----------

### 6️⃣ 验证启动成功

#### 检查端口监听：

lsof -iTCP:9091 -sTCP:LISTEN

#### 测试健康检查接口：

curl http://localhost:9091/health

# 预期返回："pong"

#### 测试业务接口（以 OpenAPI 为例）：

curl "http://localhost:9091/user/open/api/v1/blacklist/buyers?page=1&size=5" \

-H "region: SG" \

-H "saas-id: 392348641336307168"

----------

## 常见问题排查

### ❌ 问题 1: "region is nil"

原因：缺少必需的 HTTP header

解决：请求时添加 -H "region: SG" 和 -H "saas-id: 392348641336307168"

### ❌ 问题 2: "no such namespace"

原因：配置中心命名空间配置错误

解决：

1.  检查配置中心是否存在对应的命名空间（如：cs_user_test）

1.  确认 namespace 配置格式正确

1.  或者注释掉配置中心配置，使用本地配置

### ❌ 问题 3: "must have node INDEX"

原因：缺少 INDEX 环境变量

解决：添加环境变量 INDEX=1

### ❌ 问题 4: "saas id is nil"

原因：缺少 saas_id 环境变量

解决：添加环境变量 saas_id=392348641336307168

----------

## 目录结构说明

/Users/yingte.dai/code/seller-server/cs/user/

├── bin/

│ └── user # 编译后的可执行文件

├── cmd/

│ └── user/ # main 入口

├── etc/

│ └── user.yml # 配置文件

├── internal/

│ ├── http/ # HTTP 路由和控制器

│ ├── service/ # 业务逻辑

│ └── ...

├── mod/

│ └── user/ # GAS 模块定义

└── .spkit.yml # GAS 引擎版本配置

----------

## 快速启动脚本

你可以创建一个启动脚本 start.sh：

#!/bin/bash

# 启动 spex 监听（如果未运行）

if ! pgrep -f "spex.sock" > /dev/null; then

echo "Starting spex listener..."

mkdir -p /Users/yingte.dai/run/spex

socat -d -d -d UNIX-LISTEN:/Users/yingte.dai/run/spex/spex.sock,reuseaddr,fork \

TCP:agent-tcp.spex.test.shopee.io:9299 &

sleep 2

fi

# 设置环境变量并启动项目

env=test \

PROJECT_NAME=cs \

MODULE_NAME=user \

saas_id=392348641336307168 \

SP_UNIX_SOCKET=/Users/yingte.dai/run/spex/spex.sock \

cid=sg \

PORT_user=9091 \

INDEX=1 \

./bin/user

然后执行：

chmod +x start.sh

./start.sh

----------

这就是完整的启动流程！项目成功启动后会监听在 9091 端口，可以通过 HTTP 接口访问各种服务。🚀
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTYzOTU3MDYwNV19
-->