## CS 项目 Build & 启动完整流程

### 1. 安装 spkit（如果未安装）

export platform="$(uname -s| tr '[:upper:]' '[:lower:]')"

wget "https://spkit.shopee.io/spkit/stable/spkit-$platform" -O "$GOPATH/bin/spkit"

chmod a+x "$GOPATH/bin/spkit"

spkit version

### 2. 进入项目目录并创建必要目录

cd /Users/yingte.dai/code/seller-server/cs/user

mkdir -p internal/spex/gen1/go

### 3. 编译项目

spkit build

# 会生成 bin/user 或 bin/xxx 可执行文件

### 4. 启动 spex 代理（新终端窗口，保持运行）

mkdir -p ~/run/spex

socat -d -d -d UNIX-LISTEN:/Users/yingte.dai/run/spex/spex.sock,reuseaddr,fork TCP:agent-tcp.spex.test.shopee.io:9299

### 5. 运行服务（原终端）

cd /Users/yingte.dai/code/seller-server/cs/user

# 启动服务（根据项目调整环境变量）

env=test \

PROJECT_NAME=cs \

MODULE_NAME=user \

saas_id=392348641336307168 \

SP_UNIX_SOCKET=/Users/yingte.dai/run/spex/spex.sock \

cid=sg \

PORT_user=9091 \

./bin/user

### 6. 测试服务

# 测试 ping

curl http://localhost:9091/ping

# 根据项目找其他可用的 API 测试

curl http://localhost:9091/路径

### 常见问题处理

-   缺少目录：mkdir -p internal/spex/gen1/go

-   端口冲突：改 PORT_user=9092

-   编译报错：检查 VPN 连接，确保能访问 git.garena.com

直接复制命令执行即可！
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTgxNjY0OTk4NV19
-->