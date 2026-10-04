# 2026.10.04

当前部署以 `gochri/dailysync-rev` 的 `develop / ed194fd` 为准，Docker 仍是“一次性执行”的设计；

下文后续 Todo 尚未实施，不涉及 `main` 的后续开发。

针对后续想要实现的 **Docker 持久化**，以下是接下来的 Todo 事项清单、当前的简单使用介绍，以及关键的补充建议。

---

## 当前的简单使用介绍（As-Is 运行方式）

在当前未做进一步修改的代码状态下，Docker 的使用方式本质上是“容器化封装的单次任务”。

**1. 初始化配置**

* 将仓库中的 `.env.example` 复制一份并重命名为 `.env`，放在项目**根目录**下。
* 打开 `.env`，填入您的佳明中国区、国际区账号密码，以及相关的密钥配置。
* **代理提示**：如使用宿主的 HTTP 代理端口，需在 `.env` 中显式配置容器代理，并保持代理运行。容器访问宿主使用 `host.docker.internal`，不能填宿主的 `127.0.0.1`；实际端口不同则对应调整。

```dotenv
HTTP_PROXY=http://host.docker.internal:
HTTPS_PROXY=http://host.docker.internal:
NO_PROXY=localhost,127.0.0.1,::1
```

**2. 确认同步方向**
在项目根目录（包含 `docker-compose.yml` 的目录）修改或保持：

```yml
    command: yarn sync_global
```

**3. 启动单次同步**
确认 Docker Desktop 与宿主代理已启动，在项目根目录（包含 `docker-compose.yml` 的目录）执行：

```bash
docker compose up --build daily-sync
```

* **运行表现**：Docker 会构建镜像，将 `.env` 注入环境变量，拉起容器，执行一次 `yarn sync_global`（同步国际区到国区）。执行完毕后，终端会输出同步结果，随后**容器自动退出**；`Exited (0)` 表示正常完成。
* **再次执行**：容器已创建、已停止且配置未变时，执行 `docker start -a daily-sync` 即可，无需每次重新构建镜像。
* **修改 `.env` 后**：已有容器不会通过 `docker start` 或 `docker restart` 更新环境变量。镜像已构建时，执行 `docker compose up --no-build --force-recreate daily-sync`，使新配置生效并运行一次。

**4. 配置宿主机轮询（当前的定时方案）**
如需定时同步，可在 Linux 宿主机上配置 `crontab` 来定时拉起容器。以下是 Linux 示例；macOS 定时运行需单独配置，最小启动不依赖定时任务：

```bash
# 执行 crontab -e 并添加以下规则（例如每 3 小时执行一次）
0 */3 * * * cd /您的项目完整路径 && docker compose up daily-sync >> /var/log/garmin_sync.log 2>&1

```

---

**排查记录（2026-10-04）**：当 API 直连出现 `ECONNRESET / socket hang up`，依赖库将其掩盖为 `Cannot read properties of undefined (reading 'status')`。显式配置宿主代理后，两区资料及活动读取成功，原 `yarn sync_global` 完整执行并以 `0` 退出。

---

## 二、接下来的 Todo 事项清单（To-Be：Docker 持久化与安全改造）

为了将项目改造为**内部自轮询的常驻服务**，您需要按顺序进行以下改造：

### [Todo] 实现 Docker 持久化与内部轮询（轻量级方案）

* **操作**：在 `docker-compose.yml` 的 `daily-sync` 服务下，手动注入常驻循环脚本来覆盖默认的退出行为。添加 `command` 字段：

```yaml
services:
  daily-sync:
    image: 'daily-sync:latest'
    # ... 其他保持不变 ...
    command: /bin/sh -c "while true; do yarn sync_global; echo '等待下一次同步...'; sleep 10800; done"

```

* **目的**：将“一次性容器”转化为“常驻后台服务”进程。容器启动后不再退出，而是由 Shell 控制每隔 10800 秒（3 小时）左右自动执行一次同步。以后您只需要执行 `docker-compose up -d` 让它在后台一直跑即可。

### [Todo] 统一本地开发环境解析（可选）
