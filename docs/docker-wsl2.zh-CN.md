# Docker 部署（WSL2 验证版）

本分支的 Docker 版本以 Windows 侧 `win` 分支的 WARP 修复为基线：启动时 WARP 不可用会先降级服务、后台继续重试；配额冷却会持久化；出口按惩罚类型轮换；流式响应带空闲看门狗。Compose 默认在本地构建镜像，不依赖 GHCR 上是否已有对应版本。

已在 **WSL2 Ubuntu 24.04 + Docker 29.9.2（x86_64）** 上完成构建、启动、WARP 注册、模型请求和 SSE 流式验证。

## 快速开始

```bash
cp config.example.json config.json
# 编辑 config.json：
#   - server_keys 换成自己的本地 API Key
#   - webui.password 换成强密码
#   - 按需填写 zen_keys / go_keys，或保留 anonymous
docker compose up -d --build
docker compose ps
curl -s http://127.0.0.1:28080/healthz
```

- API：`http://127.0.0.1:28080`
- WebUI：`http://127.0.0.1:28081`
- 日志：`docker compose logs -f`

Windows 访问时不要用 `localhost`。本机 WSL 配置为 `localhostForwarding=false`，请先取 WSL 地址：

```powershell
wsl hostname -I
```

然后访问 `http://<WSL_IP>:28080` 和 `http://<WSL_IP>:28081`。例如本次验证环境是 `172.30.196.173:28080`。

## 端口

Compose 只改宿主机映射，容器内仍然是 8080/8081：

| 变量 | 默认值 | 作用 |
| --- | --- | --- |
| `OPENCODE2API_PORT` | `28080` | 宿主机 API 端口，映射到容器 8080 |
| `OPENCODE2API_WEBUI_PORT` | `28081` | 宿主机 WebUI 端口，映射到容器 8081 |
| `OPENCODE2API_LISTEN` | 空 | 覆盖容器内 API 监听地址 |
| `OPENCODE2API_WEBUI_LISTEN` | 空 | 覆盖容器内 WebUI 监听地址 |
| `OPENCODE2API_IMAGE` | `opencode2api:local` | 构建出的镜像名 |

默认用 28080/28081 是为了避开 WSL 开发环境里常见的 8080/8081 代理与 WebUI 占用。宿主端口被占用时改这两个变量即可：

```bash
OPENCODE2API_PORT=38080 OPENCODE2API_WEBUI_PORT=38081 docker compose up -d
```

镜像健康检查固定请求容器内 `127.0.0.1:8080/healthz`。如果修改配置里的容器内 API 端口，需要同时改端口映射和 Dockerfile 的健康检查。

首次启动时，入口脚本只在配置仍是示例值 `127.0.0.1:8080` 时把它改成 `0.0.0.0:8080`，其他监听地址原样保留。

## 配置与状态

- 首次启动把宿主机 `config.json` 复制到命名卷 `opencode2api-state`，之后**只使用卷内文件**；直接改宿主机文件不会生效。
- 只提供环境变量、没有挂载配置文件的平台（如 InstaCloud）可以用 `CONFIG_JSON` 传入整段配置。它只在 `config.json` 不存在时生效，写入权限为 `0600`，内容不会进日志：

  ```bash
  CONFIG_JSON='{"listen":"0.0.0.0:8080","server_keys":["sk-your-key"],"anonymous":false,"webui":{"enabled":true,"listen":"0.0.0.0:8081","username":"admin","password":"a-strong-password"}}'
  ```

  配置里如果仍是示例监听地址 `127.0.0.1:8080`，入口脚本仍会改成 `0.0.0.0:8080`。`webui` 块不能省：不写时 WebUI 默认关闭（`enabled` 默认 false），28081 端口会拒绝连接；启用时 `username` 必填，`password` 至少 10 位。要修改 `server_keys`、密码等结构复杂的配置，优先用 WebUI，或删掉状态卷后重新提供 `CONFIG_JSON`：

  ```bash
  docker compose down -v && docker compose up -d
  ```

- 后续修改优先用 WebUI。需要重新导入宿主机文件时：

  ```bash
  docker compose cp config.json opencode2api:/var/lib/opencode2api/config.json
  docker compose restart
  ```

- 容器以普通用户运行，根文件系统只读，只有状态卷和 `/tmp` 可写。
- 状态卷里除了 `config.json`，还有 WARP 冷却记录 `config.json.warp-state.json`（0600）、模型目录缓存、可用性缓存和轮换状态。删除卷会同时清掉配置、Key 和冷却记录。
- `config.json` 含上游 Key，已在 `.gitignore` 中；不要提交或在日志里打印。

备份与恢复：

```bash
docker run --rm -v bold-salamander_opencode2api-state:/data -v "$PWD":/backup alpine \
  tar czf /backup/opencode2api-state.tgz -C /data .
```

卷名前缀来自 Compose 项目名，可用 `docker volume ls` 确认。

## WARP

WARP 是进程内用户态 WireGuard + 用户态网络栈：不需要 TUN、root、`NET_ADMIN` 或容器路由权限，只需要能出站访问 Cloudflare 的 HTTPS 注册接口和 UDP `2408`。因此镜像的普通用户、只读根文件系统配置不需要为 WARP 放宽；不要改用 host 网络或添加 `--cap-add`。

本 WSL2 环境实测（`stack=ipv6`，`source_ipv6` 为空）可以完成注册，出口是 `2a09:bac5:...` 的 IPv6 地址；请求先遇到 429 后自动换出口并成功返回。WARP 冷却状态写入状态卷，重建容器但保留卷时仍然有效。

需要注意：

- 启动时最多等 90 秒完成首次 WARP 注册。仍失败则**先不带 WARP 启动**（记录 `warp_startup_fallback`），健康检查和服务照常可用，后台按 15 秒、30 秒、1 分钟、2 分钟、之后每 5 分钟重试；注册成功后热切换到 WARP 池。
- 通过 WebUI 保存 WARP 配置是失败即回滚：最多等 45 秒，失败会返回错误，旧运行时继续服务。
- WSL2 的 NAT 经常没有全局 IPv6 默认路由。若你的环境无法建立隧道，把 `warp.enabled` 设为 `false`，或改用可用的 `proxies`；启动回退只影响免费模型通道。
- `warp.source_ipv6` 需要一段真正路由到宿主/容器的 IPv6 前缀（通常是 `/64`）。Docker 默认 NAT 下一般拿不到这段前缀；本验证使用空值。
- `warp.stack=ipv4` 的出口是 Cloudflare 共享 NAT 地址，轮换经常仍命中已用尽的免费额度。
- 配额 429 会按 `/64`（IPv4 按 `/32`）固定冷却 24 小时，`Retry-After` 可延长但不超过 24 小时；普通限流冷却 1 分钟、最多 5 分钟；传输故障冷却 90 秒。冷却记录在状态卷里，重启不会把已用尽的网段重新放回轮换。

## 验证命令

```bash
# 就绪与资源摘要
curl -s http://127.0.0.1:28080/healthz | jq .

# 模型列表（把 KEY 换成 server_keys 里的值）
curl -s -H "Authorization: Bearer KEY" http://127.0.0.1:28080/v1/models | jq '.data | length'

# 非流式请求
curl -s http://127.0.0.1:28080/v1/chat/completions \
  -H "Authorization: Bearer KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"beam-free","messages":[{"role":"user","content":"say ok"}]}'

# SSE 流式请求
curl -N http://127.0.0.1:28080/v1/chat/completions \
  -H "Authorization: Bearer KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"beam-free","messages":[{"role":"user","content":"count 1 to 5"}],"stream":true}'
```

本次验证结果：`/healthz` ready；`/v1/models` 正常；WebUI 返回 200；非流式请求在 WARP 轮换后返回 200；SSE 收到 9 个 `data:` 事件并以 `[DONE]` 结束；服务主进程为非 root，根文件系统只读。用伪造的 `source_ipv6` 测试时，约 32 秒后进入 `warp_startup_fallback` 并正常提供健康检查。

## 常见问题

| 现象 | 处理 |
| --- | --- |
| `bind: address already in use` | `ss -lntp \| grep -E '28080\|28081'`，改 `OPENCODE2API_PORT` / `OPENCODE2API_WEBUI_PORT`。 |
| Windows 打不开页面 | 用 `wsl hostname -I` 得到的 IP，不要用 `localhost`；检查 Windows 防火墙。 |
| 构建时 Alpine 包下载失败 | 网络瞬断，重跑 `docker compose build`；必要时给 Docker 配置代理后 `--no-cache` 重建。 |
| 配置保存失败，提示目录不可写 | 不要只读挂载整个配置目录；保持状态卷可写，或把 `CONFIG_PATH` 指到卷内。 |
| 启动等很久才 ready | 首次 WARP 注册在等 90 秒上限；WSL 无 IPv6 时属预期，可先关闭 WARP。 |
| 免费模型一直 429 | 出口额度已用尽；等待冷却、增加可用代理，或换有 IPv6 前缀的宿主。 |

## 升级与清理

```bash
docker compose build --pull
docker compose up -d
docker compose down          # 保留配置和 WARP 冷却状态
docker compose down -v       # 连状态卷一起删除，谨慎
```
