# free-arm-vm 节点 Tailscale UDP 丢包导致 Kong 反复重建：排障分析与解决方案

- **文档版本**：v1.1.0
- **更新时间**：2026-11-23
- **v1.1 变更**：新增「Option 2：借道阿里云 VM 中转」，附三段链路实测数据（2026-11-23）
- **交付仓库**：`https://github.com/nvd11/my-argocd-manifests`
- **目标集群**：腾讯云业务集群 `tencent-dp1-cluster`
- **涉及节点**：`free-arm-vm`（OCI 新加坡 ARM 4C24G，Tailscale `100.105.130.0`，公网 `129.221.153.223`）、`vm-0-2-debian`（腾讯云 CP，Tailscale `100.77.64.95`，公网 `43.139.214.231`）
- **现象**：`free-arm-vm` 上的 Kong pod 频繁因 `NodeNotReady` 被驱逐重建，8 天累计 556 次节点翻转

---

## 1. 问题现象

Cindy 反馈 OCI ARM VM 到腾讯云 VM 的 Tailscale 连接"四分之一丢包"。进一步排查发现真正的影响面是 K3s 集群里 `free-arm-vm` 节点上的 Kong pod 反复被重建。

集群实证（`sudo -n k3s kubectl`，CP 节点 `vm-0-2-debian`）：

```
kong-ingress-controller-kong-9g5r6   2/2   Running   0   2d   10.42.2.106   free-arm-vm
```

事件记录：

```
default  Warning  NodeNotReady  pod/kong-ingress-controller-kong-9g5r6  Node is not ready
default  Normal   NodeNotReady  node/free-arm-vm  Node free-arm-vm status is now: NodeNotReady
default  Normal   NodeNotReady  node/free-arm-vm  NodeNotReady  37m (x556 over 8d)
```

关键点：

- **8 天 556 次 `NodeNotReady`**，平均每小时约 3 次节点翻转；
- 受影响的不只 Kong，同节点的 redis / higress / fluent-bit / astra / oauth2-proxy / svclb 等 pod 全部被标记；
- 每次翻转，svclb 都会把 ARM 的 `100.105.130.0` 从 Kong LoadBalancer 的 EXTERNAL-IP 列表里踢出再重新加回：

```
UpdatedLoadBalancer  [100.104.150.19 100.105.130.0 100.77.64.95] -> [100.104.150.19 100.77.64.95]
UpdatedLoadBalancer  [100.104.150.19 100.77.64.95] -> [100.104.150.19 100.105.130.0 100.77.64.95]
```

结论：Kong 本身没问题，它是节点心跳超时的受害者。真正的问题是**整个 `free-arm-vm` 节点在反复失联**。

---

## 2. 根因定位

### 2.1 kubelet 心跳（Lease）超时

登录 `free-arm-vm` 查 k3s-agent 日志，满屏都是 Lease 更新失败：

```
E1123 02:05:06 controller.go:251 "Failed to update lease"
  err="Put \"https://127.0.0.1:6444/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/free-arm-vm?timeout=10s\":
       net/http: request canceled (Client.Timeout exceeded while awaiting headers)"

E1123 02:05:36 controller.go:171 "failed to update lease using latest lease, fallback to ensure lease"
  err="failed 5 attempts to update lease"

E1123 02:07:31 kubelet_node_status.go:465 "Unable to update node status"
  err="update node status exceeds retry count"
```

因果链：

```
kubelet 连不上 apiserver (6444 本地代理 → 100.77.64.95:6443)
  → Lease 更新 10s 超时，重传 5 次全失败
  → node-controller 判定节点失联，标记 NodeNotReady
  → 该节点所有 pod 被标记，svclb 摘除 LB IP
  → Kong 被驱逐重建
```

kubelet 心跳的应用层其实是 **HTTPS/TCP 的 Lease PUT**（`127.0.0.1:6444` 是 k3s-agent 的本地 apiserver 代理），但它连的目标地址决定了外层走什么隧道——这是下一节的关键。

### 2.2 心跳为什么要走 Tailscale 的 UDP 隧道

在 `free-arm-vm` 上查配置与路由：

```
# /etc/systemd/system/k3s-agent.service.env
K3S_TOKEN='K10665...36b2'
K3S_URL='https://100.77.64.95:6443'       ← 心跳目标是 Tailscale IP
K3S_OPTS="--flannel-iface tailscale0"

# 路由
$ ip route get 100.77.64.95
100.77.64.95 dev tailscale0 table 52 src 100.105.130.0 ...
```

`tailscale0` 是 WireGuard 的 tun 设备（`ip -d link` 确认），WireGuard 底层是 **UDP 封装**。所以那个 HTTPS/TCP 心跳包，出了机器就被裹进 UDP 外壳，在跨境链路上表现为 UDP 流量，正是被封锁针对的对象。

> 准确表述：心跳本身是 TCP/HTTPS，只因为目标是 Tailscale IP 才被套进 WireGuard 的 UDP 外壳。

---

## 3. 链路质量实测数据

所有数据均为实机对打测试，非推测。测试时间点：2026-11-23。

### 3.1 三组路径对比（决定性证据）

| 路径 | 协议 | 丢包率 | RTT |
|---|---|---|---|
| 公网 IP 直连（无 Tailscale） | ICMP | **1~3%** | 232ms |
| 公网 IP 直连 | UDP（裸打流） | **100%（双向 0 收到）** | — |
| Tailscale 隧道（P2P 直连） | ICMP 小包 | **31~48%** | 136ms 基线，抖动剧烈 |
| Tailscale 隧道 | TCP（真实 SSH） | 0%（5/5 握手成功） | — |
| Tailscale DERP(sin) 中转 | ICMP | **2%** | 370ms |

测试方法：

- ICMP：`ping -c 100 -i 0.2 -q <target>`，两端对打，双向对称复现 31~32%；
- UDP：Python socket 双向各发 200 包，`RECV=0 LOST=200`；脚本本机回环自测 50/50 正常，排除脚本 bug；
- DERP：在 CP 节点用 `sudo ip route add blackhole 129.221.153.223` 临时阻断直连，逼 Tailscale 降级 DERP(sin) 后测得，测完已删除路由恢复原状。

### 3.2 MTU 扫测（排除分片因素）

从 CP 到 ARM，带 DF 标志逐档测试：

```
payload=56   丢包 30%
payload=512  丢包 37.5%
payload=1000 丢包 32.5%
payload=1200 丢包 22.5%
payload=1300 丢包 100%（隧道 MTU 上限，正常）
```

小包照样丢 30%，说明**不是 MTU 黑洞、不是分片问题**，是链路本身对 UDP 的随机丢弃。

### 3.3 链路诊断

- `tailscale ping` 两端均显示走**直连 P2P**（`via 129.221.153.223:41641`，RTT ~136ms），不是 DERP 过载；
- RTT 极小包时极稳定（mdev 0.2~0.3ms），不是拥塞排队，是干净的随机丢包；
- 公网 traceroute：广州电信出口 `202.97.93.81 → 203.215.237.154` 后 RTT 从 4ms 跳到 120ms（国际段），末尾多跳超时。

### 3.4 根因结论

**这条跨境链路对 UDP 有针对性丢弃/ QoS 歧视**：裸 ICMP 只丢 1~3%（链路本身质量好），裸 UDP 双向 0 收到。Tailscale 的 WireGuard 全靠 UDP，隧道继承 31~48% 丢包，导致：

- kubelet Lease（裹在 UDP 里的 HTTPS）10s 超时 → NodeNotReady → Kong 重建；
- TCP 服务靠重传兜底，隧道里 SSH 5/5 握手成功，所以表面看"还能用"，但心跳这种对丢包敏感的小流量扛不住。

---

## 4. 解决方案

按推荐度排序。所有方案均未实施，供决策。

### 方案 D（推荐）：心跳改走公网 IP，从根上绕开 UDP 隧道

**思路**：两台节点都有公网 IP、公网 TCP 实测稳定、apiserver 证书已覆盖公网 IP，心跳没必要绕 Tailscale 的 UDP 隧道。

**前置验证（已完成，只读）**：

- ARM → `43.139.214.231:6443`：8/8 全 OPEN；
- 证书 SAN 含 `IP Address:43.139.214.231`，TLS 校验直接过；
- `ip route get 43.139.214.231` → `dev enp0s6`（物理网卡），裸 TCP。

**改动（一行）**：ARM 节点 `/etc/systemd/system/k3s-agent.service.env`

```
- K3S_URL='https://100.77.64.95:6443'
+ K3S_URL='https://43.139.214.231:6443'
```

token 与 `--flannel-iface tailscale0` 均不动。生效：

```bash
sudo systemctl daemon-reload && sudo systemctl restart k3s-agent
```

**验证**：k3s-agent 日志中 `Failed to update lease` 消失；`kubectl describe node free-arm-vm` 的 `NodeNotReady x556` 计数停止增长；观察 24 小时 Kong 不再被驱逐。

**局限**：只治心跳，flannel 的 `--flannel-iface tailscale0` 未动，pod-to-pod 的 VXLAN（UDP 8472）跨节点流量依然吃丢包。但对当前痛点（NodeNotReady → Kong 重建）直达病根。

**回滚**：把那行改回 `100.77.64.95`，restart 即可，秒级可逆。

**风险**：重启 agent 期间节点心跳短暂中断（放维护窗口做）；腾讯云固定公网 IP 若变更需同步改这行。

### Option 2（新增）：借道阿里云 VM 中转，以 TCP 绕开 UDP 封锁

**思路**：既然病根是腾讯云出口对 UDP 的针对性丢弃，而腾讯云 ↔ 阿里云的**国内 TCP 链路质量极好**，那就让腾讯云的流量先走 TCP 到阿里云，再由阿里云转发到 OCI ARM。用一段可控的 TCP 链路，替换掉那段不可控的跨境 UDP。

**前置实测（2026-11-23，三段链路全部实机对打，非推测）**：

| 路径 | 协议 | 丢包率 | RTT | 判定 |
|---|---|---|---|---|
| 腾讯云 → 阿里云公网 | ICMP | **0%**（40/40） | 1.97ms | 国内骨干，完美 |
| 腾讯云 → 阿里云公网 | TCP:22（10x） | **0%（10/10）** | ~2ms | 完美 |
| 腾讯云 → 阿里云 Tailscale | ICMP | 0% 但抖动剧烈 | 107ms（mdev 285ms） | 被迫走 DERP 的罪证 |
| 阿里云 → OCI ARM 公网 | ICMP | **5%**（38/40） | 198ms | 可用 |
| 阿里云 → OCI ARM Tailscale | ICMP | 5%（38/40） | 198~231ms | `direct 129.221.153.223:41641` |

关键发现（再次坐实 3.4 的根因结论）：

- 腾讯云这台机器 **ICMP/TCP 完全正常**——到阿里云 2ms、零丢包、TCP 10/10；
- 但**只要套进 UDP（Tailscale WireGuard）就被迫绕 DERP**，RTT 从 2ms 飙到 107ms、抖动 mdev 285ms；
- 这证明之前腾讯云 → ARM 的 31~48% 丢包不是链路烂，是 UDP 被针对性封锁。

**中转路径与收益**：

```
腾讯云 →(TCP, 2ms, 0% 丢包)→ 阿里云 →(直连/TCP, 198ms, 5% 丢包)→ OCI ARM
```

总延迟约 200ms（与绕 DERP 相当），但**丢包从 31~48% 降到 5%**，且 TCP 段靠重传可完全吸收。核心价值：把不可控的 UDP 封锁，换成两段可控链路。

**三种落地方式（按复杂度排）**：

| 方式 | 做法 | 适用 | 代价 |
|---|---|---|---|
| **O2-A｜阿里云自托管 DERP**（最优雅） | 阿里云跑 `derper`，腾讯云节点自动选它为最近 DERP（2ms），Tailscale 流量自动走 TCP 443 中继 | 全流量覆盖、零客户端改动 | 需域名 + TLS 证书 |
| **O2-B｜socat 端口转发**（最简单） | 阿里云 `socat` 转发单个 TCP 服务（如 apiserver 6443） | 只解决单个端口 | 十分钟搞定，不覆盖 flannel |
| **O2-C｜frp 内网穿透**（最成熟） | 阿里云 `frps` + ARM `frpc` | 需转发多个服务 | 多一个组件 |

**落地示例（O2-B，以心跳 apiserver 6443 为例）**：

```bash
# 阿里云（8.148.149.80）上：把 6443 转发到腾讯云 CP
# 需先确认阿里云安全列表放行所选端口
socat TCP-LISTEN:6443,fork,reuseaddr TCP:43.139.214.231:6443 &

# ARM 节点 k3s-agent.service.env：心跳目标从 Tailscale IP 改为阿里云中转 IP
- K3S_URL='https://100.77.64.95:6443'
+ K3S_URL='https://8.148.149.80:6443'

sudo systemctl daemon-reload && sudo systemctl restart k3s-agent
```

**与现有方案的关系**：Option 2 治的是「跨云整体链路」，粒度比方案 D（只治心跳那一行）更大；若采用 O2-A 全流量 DERP，可同时覆盖心跳和 flannel 的 VXLAN（UDP 8472），弥补方案 D「只治心跳」的局限。可与方案 D 叠加，也可作为其替代。

**局限**：

- 阿里云这台（`aliyun-master`，2C2G）同时是 ArgoCD 控制面 / K3s CP，加中转负载需评估余量；
- O2-B 只转单个端口，pod-to-pod 的 flannel 流量仍吃 UDP 丢包；要全覆盖得用 O2-A/O2-C；
- 多一跳中转，阿里云成为单点；其公网 IP 变更需同步改客户端目标。

**回滚**：O2-B 把 `K3S_URL` 改回原值并 restart、停掉 socat 即可，秒级可逆；O2-A/O2-C 停对应服务即可。

**状态**：已实测链路可行性，三种落地方式均未实施，供决策。

### 方案 A1：强制 Tailscale 走 DERP 中转（已实测有效，兜底）

**思路**：阻断 ARM ↔ CP 的 UDP 直连，逼 Tailscale 降级 DERP（TCP 443 封装），绕开 UDP 封锁。实测丢包 31% → 2%。

**落地**：CP 节点（`cp1-vm` 无 iptables/nft，有 `ip` 命令）用 systemd oneshot 持久黑洞路由：

```ini
# /etc/systemd/system/force-derp-arm.service
[Unit]
Description=Force Tailscale DERP relay for free-arm-vm
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=-/sbin/ip route del blackhole 129.221.153.223
ExecStart=/sbin/ip route add blackhole 129.221.153.223
ExecStop=-/sbin/ip route del blackhole 129.221.153.223

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now force-derp-arm.service
```

**验证**：`tailscale ping -c 4 100.105.130.0` 显示 `via DERP(sin)`；ICMP 丢包降到 ~2%。回滚：`sudo systemctl disable --now force-derp-arm.service`。

**代价**：RTT 136ms → 370ms（延迟翻倍，对心跳小流量无感）；吃 Tailscale 免费 DERP 带宽配额。

### 方案 B：放宽 K3s 心跳容忍（治标，不建议单用）

CP 的 `/etc/rancher/k3s/config.yaml` 里**已配置** `node-monitor-grace-period=120s`，但 8 天仍 556 次 NotReady —— 说明单纯放宽容忍扛不住连续 lease 超时。可作为 DERP/公网方案的兜底，不单独使用。

### 方案 C：换线路 / 换区（已排除）

- **C1 节点间改走公网 IP**：flannel 的 VXLAN 后端本身是 UDP（8472），改公网 IP 只治好心跳（TCP），pod 间通信照样被封；host-gw 要求同二层，跨公网用不了。理论上走不通。
- **C2 换 OCI 区域 / 买专线**：新加坡已是离大陆最优区域（已排除换区）；专线/精品线路成本高，与实验集群定位不匹配；有状态 Redis 数据钉在 free-arm-vm，迁移有丢数据风险。

---

## 5. 建议与实施顺序

1. **先上方案 D**（心跳走公网 TCP，根治 NotReady，改动只有一行 env）；
   - 若要**一并解决 flannel 的 UDP 丢包**，评估 Option 2（O2-A 自托管 DERP 全流量中转），它覆盖面比方案 D 大；
2. 观察 24 小时，`NodeNotReady` 计数停止增长、Kong 不再重建即告成功；
3. 若后续发现**跨节点 pod 间通信**也卡顿（Kong → 跨节点 backend 流量），再叠加方案 A1（DERP）治 flannel 那条 UDP —— 两者不冲突，各治一段；
4. 方案 B 已有部分配置，作为兜底保留。

未经明确授权，上述方案均未实施。

---

## 附录：关键命令速查

```bash
# 查看 Kong pod 与节点状态
sudo -n k3s kubectl get pods -n kong-system -o wide        # CP 节点
sudo -n k3s kubectl get events --all-namespaces --sort-by=.lastTimestamp | grep -iE "notready|free-arm"

# 查看节点翻转计数
sudo -n k3s kubectl describe node free-arm-vm | grep -i "NodeNotReady"

# 查看 ARM 节点 lease 失败日志
sudo -n journalctl -u k3s-agent --since "24 hours ago" | grep -iE "lease|timeout|deadline"

# Tailscale 路径判定
tailscale ping -c 4 100.105.130.0   # via <ip>:41641 = 直连；via DERP(sin) = 中转

# 丢包测试
ping -c 100 -i 0.2 -q <target>
```
