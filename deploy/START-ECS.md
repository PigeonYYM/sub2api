# `start-ecs.sh` 启动流程说明

`deploy/start-ecs.sh` 是一个**单机一键启动脚本**：在一台全新的 Ubuntu ECS（root 用户）上，自动安装并拉起 PostgreSQL、Redis，生成配置、编译后端二进制，最后以前台进程运行 Sub2API。进程退出时会自动回收所有依赖服务。

> 与 `run-ecs.sh` 的区别：`start-ecs.sh` 会**自行编译** Go 二进制（`build_binary`），无需提前在本地交叉编译上传；`run-ecs.sh` 则要求二进制已存在。

---

## 关键路径与变量

| 变量 | 默认值 | 用途 |
|------|--------|------|
| `WORK_DIR` | `/root/workspace/sub2api` | 代码与运行根目录 |
| `DATA_DIR` | `/var/lib/sub2api` | 持久化数据根目录 |
| `PG_DATA` | `${DATA_DIR}/postgres` | PostgreSQL 数据目录 |
| `REDIS_DATA` | `${DATA_DIR}/redis` | Redis 数据目录 |
| `RUNTIME_DIR` | `${DATA_DIR}/runtime` | 运行时文件（socket / pid / 日志 / redis.conf） |
| `ENV_FILE` | `${WORK_DIR}/.env` | 环境变量配置 |
| `BINARY` | `${WORK_DIR}/sub2api` | 编译产物 |

脚本头部 `set -euo pipefail`：任一命令失败即退出、引用未定义变量报错、管道中任一环节失败即失败。

---

## 启动流程总览

`main()`（`start-ecs.sh:231`）按以下顺序执行：

```
检查 root 权限
  └─ install_deps      # 1. 安装依赖
  └─ discover_pg       # 2. 发现 PostgreSQL 版本
  └─ setup_dirs        # 3. 创建数据目录并设权限
  └─ init_postgres     # 4. 初始化 PG 数据目录（首次）
  └─ start_postgres    # 5. 启动 PostgreSQL
  └─ generate_env      # 6. 生成 .env（首次）
  └─ ensure_db         # 7. 创建数据库和用户（首次）
  └─ start_redis       # 8. 启动 Redis
  └─ build_binary      # 9. 编译后端二进制（首次）
  └─ trap cleanup ...  #    注册退出钩子
  └─ start_app         # 10. 前台运行 Sub2API（阻塞）
```

整个流程是**幂等**的：第二次运行时已存在的资源会被跳过，仅重新启动服务。

---

## 分步详解

### 0. 前置检查（`main:232`）
非 root 用户直接报错退出（`Must run as root`）。后续 `initdb` / `pg_ctl` 通过 `gosu postgres` 降权运行，Redis / 编译则以 root 执行。

### 1. `install_deps`（`:21`）
按需安装系统依赖（已安装则跳过）：
- 缺 `psql` → 安装 `postgresql postgresql-client redis-server gosu openssl git curl`
- 缺 `redis-server` → 单独补装
- 缺 `gosu` → 单独补装

使用 `apt-get -qq` + `DEBIAN_FRONTEND=noninteractive` 静默安装。

### 2. `discover_pg`（`:37`）
在 `/usr/lib/postgresql` 下找出最高版本目录（`sort -V | tail -n 1`），推导出 `PG_BIN`（如 `/usr/lib/postgresql/16/bin`）。后续所有 PG 命令都用这个绝对路径，避免版本歧义。

### 3. `setup_dirs`（`:43`）
创建 `PG_DATA / REDIS_DATA / RUNTIME_DIR`，预创建 `postgres.log`，并设置属主与权限：
- `PG_DATA`、`RUNTIME_DIR` → `postgres:postgres`
- `REDIS_DATA` → `redis:redis`
- `PG_DATA` 权限 `700`（initdb 要求），`RUNTIME_DIR` 为 `755`

### 4. `init_postgres`（`:53`）
若 `${PG_DATA}/PG_VERSION` 已存在则跳过（说明已初始化）。否则用 `gosu postgres initdb` 初始化：
- 本地连接 `--auth-local=trust`（同机免密）
- 远程连接 `--auth-host=scram-sha-256`
- 编码 `UTF8`，locale `C.UTF-8`

### 5. `start_postgres`（`:67`）
`pg_ctl -w start`（`-w` 等待就绪），日志写入 `runtime/postgres.log`，并通过 `-o` 传入运行参数：
- 仅监听 `127.0.0.1:5432`
- Unix socket 放在 `RUNTIME_DIR`（后续 `psql -h ${RUNTIME_DIR}` 走 socket 连接）

### 6. `generate_env`（`:144`）
若 `.env` 已存在则保留不动。首次则生成随机凭据并写入：
- `DATABASE_PASSWORD`、`REDIS_PASSWORD`、`JWT_SECRET`、`TOTP_ENCRYPTION_KEY`、`ADMIN_PASSWORD` 全部用 `openssl rand -hex` 随机生成
- 固定项：`AUTO_SETUP=true`、`SERVER_HOST=0.0.0.0`、`SERVER_PORT=8080`、`SERVER_MODE=release`、`ADMIN_EMAIL=admin@sub2api.local`、`TZ=Asia/Shanghai`
- 文件权限 `600`，并在日志中打印一次性 **管理员密码**

> 注意顺序：`generate_env` 在 `ensure_db` / `start_redis` **之前**，因此后两者能从 `.env` 读到刚生成的密码并复用，保证三方密码一致。

### 7. `ensure_db`（`:76`）
通过 socket 查询 `pg_database` 判断 `sub2api` 库是否存在；存在则跳过。否则：
- 从 `.env` 读取 `DATABASE_PASSWORD`（读不到则临时随机生成）
- `CREATE USER sub2api` + `CREATE DATABASE sub2api OWNER sub2api`
- 用 `sed` 把实际密码回写到 `.env`，确保配置与数据库一致

### 8. `start_redis`（`:103`）
若 `redis.pid` 存在且进程存活则跳过。否则：
- 从 `.env` 读取（或随机生成）`REDIS_PASSWORD`
- 动态生成 `runtime/redis.conf`：仅听 `127.0.0.1:6379`、`daemonize yes`、`appendonly yes`、`requirepass`
- 启动后最多 ping 探活 30 次（每次间隔 1s）确认就绪
- 将密码回写 `.env`

### 9. `build_binary`（`:186`）
若 `${BINARY}` 已是可执行文件则跳过。否则进入 `backend/`：
- `go mod download`
- `CGO_ENABLED=0 go build -o ${BINARY} ./cmd/server`

### 10. `start_app`（`:199`）
- `set -a && source .env`：把 `.env` 全部导出为环境变量供应用读取
- 后台启动二进制并记录 `APP_PID`，再 `wait` 阻塞在前台
- 进程会一直运行，直到收到信号或自身退出

---

## 停止与清理（`cleanup` `:208`）

`main` 在 `start_app` 前注册了 `trap cleanup EXIT INT TERM`。无论是 `Ctrl+C`、`kill` 还是应用自行退出，都会触发清理，且通过 `_CLEANED_UP` 标志保证**只执行一次**：

1. `kill -TERM` 应用进程并等待退出
2. 用 `.env` 中的 Redis 密码执行 `redis-cli shutdown`
3. `pg_ctl -m fast stop` 关闭 PostgreSQL

所有清理步骤都带 `|| true`，确保即使某项失败也不会阻断后续关闭。

---

## 幂等性小结

| 资源 | 跳过条件 |
|------|----------|
| 系统依赖 | 对应命令已存在 |
| PG 初始化 | `PG_DATA/PG_VERSION` 存在 |
| `.env` | 文件已存在 |
| 数据库 | `pg_database` 中已有 `sub2api` |
| Redis | `redis.pid` 进程存活 |
| 二进制 | `BINARY` 可执行 |

因此重复运行 `start-ecs.sh` 是安全的——只会重新拉起服务，不会覆盖已有数据或配置。

---

## 使用方式

```bash
sudo bash /root/workspace/sub2api/deploy/start-ecs.sh
```

首次启动后，在日志中查看一次性管理员密码，访问 `http://ECS_IP:8080`。按 `Ctrl+C` 即可优雅停止全部服务。

---

## 启动后操作

`start-ecs.sh` 是**前台阻塞**运行的（`start_app` 里 `wait`），终端关闭或 `Ctrl+C` 会触发 `cleanup` 把 App / Redis / PostgreSQL 全部关掉。因此启动后通常还需要：验证、配反代、后台常驻。

### 1. 验证服务

```bash
# 端口是否监听（应看到 8080 / 5432 / 6379）
ss -ltnp | grep -E '8080|5432|6379'

# 后端是否响应
curl -i http://127.0.0.1:8080/

# PostgreSQL 日志
tail -f /var/lib/sub2api/runtime/postgres.log
```

管理员密码只在首次启动日志里打印一次，也可从配置读回：

```bash
grep -E 'ADMIN_EMAIL|ADMIN_PASSWORD' /root/workspace/sub2api/.env
```

### 2. 配置 Nginx 反代（80 → 8080，推荐）

```bash
sudo bash /root/workspace/sub2api/deploy/setup-nginx.sh
```

之后访问 `http://ECS_IP/`（80 端口），无需再带 `:8080`。

### 3. 后台常驻

#### 方式 A：systemd（有 systemd 的标准 ECS / VM 实例）

用 `run-ecs-background.sh` 注册成开机自启服务。它**默认指向 `run-ecs.sh`**，需用 `RUN_SCRIPT` 覆盖指向本脚本：

```bash
sudo RUN_SCRIPT=/root/workspace/sub2api/deploy/start-ecs.sh \
     SERVICE_NAME=sub2api-ecs \
     bash /root/workspace/sub2api/deploy/run-ecs-background.sh start
```

日常管理：

```bash
sudo bash deploy/run-ecs-background.sh status     # 查看状态
sudo bash deploy/run-ecs-background.sh logs       # journalctl -f 实时日志
sudo bash deploy/run-ecs-background.sh restart    # 重启
sudo bash deploy/run-ecs-background.sh stop        # 停止
sudo bash deploy/run-ecs-background.sh uninstall   # 卸载服务
```

> systemd 服务设了 `Restart=always`。由于 `start-ecs.sh` 幂等（二进制/数据已存在即跳过编译和初始化），重启只会重新拉起 PG/Redis/App，不会重编译或覆盖数据。

#### 为什么有时用不了 `systemctl`？

| 原因 | 说明 |
|------|------|
| 环境无 systemd | 容器化 ECS、Serverless ECS、精简镜像或 Docker 容器通常不以 systemd 作 PID 1，执行 `systemctl` 会报 `System has not been booted with systemd as init system (PID 1)` |
| 脚本主动拦截 | `run-ecs-background.sh` 的 `require_systemd` 检测不到 `systemctl` 会直接退出 |

普通阿里云 ECS（VM 实例）一般自带 systemd；只有上述无 systemd 的环境用不了，此时改用方式 B。

#### 方式 B：nohup（无 systemd 环境）

```bash
# 后台运行并把日志写到文件，关闭 SSH 也不会停
sudo nohup bash /root/workspace/sub2api/deploy/start-ecs.sh \
     > /var/log/sub2api-start.log 2>&1 &

# 查看日志
tail -f /var/log/sub2api-start.log
```

停止（向脚本进程发 TERM，触发 `cleanup` 优雅关闭全部服务）：

```bash
sudo pkill -TERM -f deploy/start-ecs.sh
```

> 注意：nohup 方式不会开机自启，机器重启后需手动再次执行。
