# ☸️ Kubernetes 集群自动化部署（Ansible + Kubeadm + CRI-O + Calico）

> 一套面向国内网络环境的 Kubernetes 裸机/虚拟机 **Day 0 自动化交付方案**。从节点初始化到工作节点加入集群，全程声明式编排，无需手动登录节点执行命令。

---

## 📋 目录

- [项目概述](#-项目概述)
- [技术选型](#-技术选型)
- [架构与部署流程](#-架构与部署流程)
- [目录结构](#-目录结构)
- [快速开始](#-快速开始)
- [核心配置解析](#-核心配置解析)
- [已知局限与演进方向](#-已知局限与演进方向)

---

## 🎯 项目概述

本项目解决的是 **"如何在裸机或虚拟化环境中，以可重复、可审计的方式交付一个生产就绪的 Kubernetes 集群"**。

在 homelab 和生产环境 POC 中，手动逐节点执行 `kubeadm init`、`sysctl` 调优、CNI 安装等操作不仅效率低下，更容易因人为疏忽导致环境不一致。本项目通过 Ansible Playbook 将整个集群交付过程编码化，实现：

| 能力 | 说明 |
|---|---|
| **节点标准化** | 自动完成主机名、Swap 禁用、SELinux、防火墙、内核模块、sysctl 网络调优等基础设施准备工作 |
| **运行时交付** | 自动安装并配置 CRI-O 容器运行时，附带国内镜像加速与自定义 pause 镜像 |
| **控制平面初始化** | 通过 Kubeadm 拉起单 Master 集群，自动分发 `admin.conf` 到指定用户 |
| **网络插件部署** | 以 Tigera Operator 声明式部署 Calico CNI，支持 VXLAN 封装 |
| **工作节点加入** | 自动创建 join token，并驱动 Slave 节点加入集群，带 Calico 就绪检测 |
| **可观测性基线** | 通过 systemd 部署 Promtail（日志采集）与 Node Exporter（节点指标），二进制由节点运行时从官方 Release 自动下载，无需预置到仓库 |

**设计要点：**
- 🏗️ **声明式基础设施**：所有节点配置通过 Ansible `template` / `lineinfile` / `sysctl` 模块管理，变更可追踪、可回滚
- 🔒 **版本锁定**：K8s、CRI-O、Podman 版本在 `group_vars` 中集中定义，避免"在我机器上能跑"
- 🌐 **国内网络适配**：K8s 核心镜像（pause 等）走华为云 SWR 镜像代理，kubeadm / CRI-O 软件源通过模板动态渲染
- ✅ **合规审查**：通过 `ansible-lint` production profile 检查，遵循 Ansible 最佳实践

---

## 🛠️ 技术选型

### 容器运行时：CRI-O 而非 Containerd/Docker

| 维度 | CRI-O | Containerd | Docker |
|---|---|---|---|
| **架构设计目标** | 专为 K8s CRI 设计，极简守护进程 | 通用容器运行时，兼管 Docker 生态 | 完整的容器平台，功能冗余 |
| **Systemd 集成** | ✅ 原生 Cgroup v2 支持，与 systemd 深度集成 | ⚠️ 需额外配置 | ❌ 需 docker-ce |
| **安全 attack surface** | ✅ 最小化，无多余守护进程 | 中等 | 较大 |
| **Podman 生态一致性** | ✅ 与 Podman 共享 containers/image、containers/storage 等库 | 独立实现 | 独立实现 |

**选型理由**：在 RHEL/Rocky Linux 系发行版中，CRI-O 与操作系统的设计哲学最为契合（Systemd 集成、Cgroup v2 原生、无冗余守护进程）。对于以 K8s 为唯一容器编排平台的场景，CRI-O 是更纯粹的选择。

### 网络插件：Calico Operator 模式

本项目采用 **Tigera Operator v1.40.7** 管理 Calico v3.31，而非传统的直接 `kubectl apply` Calico manifest。

**Operator 模式的优势：**
- 生命周期管理：Calico 组件（node、typha、controller 等）的升级、配置变更加载由 Operator 自动协调
- 声明式配置：通过 `Installation` CRD 定义网络参数（CIDR、VXLAN、BGP 模式等），符合 K8s 原生范式
- 可观测性扩展：本项目已预置 `Goldmane`（流聚合器）和 `Whisker`（可观测性 UI）CRD，为后续流量可视化预留接口

### 编排工具：Ansible

选择 Ansible 而非 Shell 脚本或 Terraform 的原因：
- **幂等性**：`dnf`、`systemd`、`sysctl`、`lineinfile` 等模块天然幂等，重复执行不会破坏已有环境
- **无 Agent**：仅需 SSH + Python，不依赖目标节点预装任何代理
- **变量与模板分离**：镜像地址、版本号、网络参数全部抽离到 `group_vars`，同一套 Playbook 可适配多环境

---

## 🏗️ 架构与部署流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Ansible Control Node                           │
│  ┌─────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐  │
│  │ inventory/  │  │ playbooks/site  │  │   template/ + deploy/       │  │
│  │ hosts.ini   │──│ .yml (编排入口)  │──│  repo / crio / calico ...   │  │
│  └─────────────┘  └─────────────────┘  └─────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │ SSH
           ┌────────────────────────┼────────────────────────┐
           ▼                        ▼                        ▼
   ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
   │   k8s-node-1  │       │   k8s-node-2  │       │   k8s-node-3  │
   │   [master]    │       │   [slave]     │       │   [slave]     │
   └───────────────┘       └───────────────┘       └───────────────┘
```

### Day 0 部署流水线（site.yml）

| 阶段 | Playbook | 作用域 | 关键操作 |
|---|---|---|---|
| **1. 环境初始化** | `Environment_initialization.yml` | all | 主机名、防火墙、SELinux、Swap 彻底禁用、`br_netfilter` 模块、内核网络参数调优 |
| **2. 依赖安装** | `Cluster_dependces_setup.yml` | k8s_nodes | 渲染 yum repo 模板、安装 `kubelet/kubeadm/kubectl/cri-o`、配置国内镜像 registry 与 pause 镜像、启动 CRI-O |
| **3. 账户管理** | `Manage-accounts.yml` | all | 创建特权用户并配置 sudo（可选） |
| **4. 集群初始化** | `Initialization-cluster.yml` | master | `kubeadm init`（指定 CRI socket 与 Pod CIDR）、创建 `.kube/config` |
| **5. CNI 部署** | `Apply_calico_cni.yml` | master | 依次应用 Calico CRDs → Tigera Operator → Installation CR |
| **6. 节点加入** | `Join_cluster.yml` | slave | Master 上生成 join token → Slave 节点执行 `kubeadm join`（带 CRI socket 与节点名） |

### Day 1 扩展：可观测性 Agent 部署

集群就绪后，可独立执行 `deploy_observability_agents.yml` 为所有节点注入节点级可观测能力：

| 阶段 | Playbook | 作用域 | 关键操作 |
|---|---|---|---|
| **可观测性 Agent** | `deploy_observability_agents.yml` | k8s_nodes | 运行时从 GitHub Release 自动下载 Promtail / Node Exporter，解压后安装到 `/usr/local/bin/`，渲染 systemd unit 与配置，启动并启用服务 |

> **设计意图**：可观测性 Agent 作为 Day 1 扩展，与 Day 0 集群交付解耦。二进制不预置在仓库中，避免 Git 大文件限制与架构耦合，节点通过 `get_url` 按需拉取指定版本。 |

---

## 📁 目录结构

```
.
├── README.md                           # 项目文档（本文件）
├── LICENSE                             # MIT License
├── ansible.cfg                         # Ansible 运行时配置
├── requirements.yml                    # Ansible Collection 依赖
├── quick_start.sh                      # 一键部署入口脚本
│
├── inventory/
│   └── hosts.ini.example               # 主机清单模板（复制后编辑）
│
├── playbooks/
│   ├── site.yml                        # 总编排入口，按顺序 import 各阶段 Playbook
│   ├── Environment_initialization.yml  # Day 0：节点初始化（防火墙/Swap/SELinux/内核）
│   ├── Cluster_dependces_setup.yml     # Day 0：CRI-O + K8s 组件安装与配置
│   ├── Initialization-cluster.yml      # Day 0：Kubeadm 初始化控制平面
│   ├── Apply_calico_cni.yml            # Day 0：Calico Operator 与网络配置
│   ├── Join_cluster.yml                # Day 0：工作节点加入集群
│   ├── Manage-accounts.yml             # 账户与权限管理（可选）
│   ├── deploy_observability_agents.yml   # Day 1：节点级可观测性 Agent 部署（Promtail + Node Exporter）
│   └── group_vars/
│       ├── k8s_nodes.yml               # K8s 版本、包列表、sysctl、模块等全局变量
│       └── user_information.yml        # 用户账户信息
│
├── template/
│   ├── repos/
│   │   ├── kubernetes.repo.j2          # K8s 官方 yum 仓库模板
│   │   └── cri-o.repo.j2               # CRI-O 官方 yum 仓库模板
│   ├── crio/
│   │   └── 20-pause.conf.j2            # CRI-O pause 镜像配置
│   ├── containers/
│   │   └── 99-registries.conf.j2       # 容器镜像仓库镜像/重定向配置
│   └── sysctl/
│       └── k8s.conf.j2                 # 内核参数调优配置
│
│   └── observability_agents/
│       ├── promtail/
│       │   ├── promtail-config.yaml.j2 # Promtail 配置模板（Loki 地址、日志路径、hostname 标签）
│       │   └── promtail.service.j2     # Promtail systemd unit 模板
│       └── node-exporter/
│           └── node-exporter.service.j2 # Node Exporter systemd unit 模板
│
└── deploy/
    └── calico/
        ├── operator-crds.yaml          # Calico Operator CRD 定义
        ├── tigera-operator.yaml        # Tigera Operator 部署清单
        ├── custom-resources-iptables.yaml  # Calico Installation / APIServer / Goldmane / Whisker CR
        └── custom-resources-bpf.yaml   # Calico eBPF 模式配置（预留）
```

---

## 🚀 快速开始

### 1. 前置要求

- **控制节点**：安装 Ansible >= 2.15，可 SSH 免密登录所有目标节点
- **目标节点**：RHEL 8/9 或兼容衍生版（Rocky Linux、AlmaLinux），建议 2C4G 及以上
- **网络**：目标节点可访问互联网（用于下载软件包与镜像）

### 2. 安装依赖

```bash
ansible-galaxy collection install -r requirements.yml
pip install passlib  # 若启用 Manage-accounts.yml 中的密码哈希功能
```

### 3. 配置集群节点

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
# 编辑 hosts.ini，填入你的节点 IP、主机名与 SSH 用户
```

示例：
```ini
[k8s_nodes]
k8s-node-1 ansible_host=192.168.1.11 ansible_user=root
k8s-node-2 ansible_host=192.168.1.12 ansible_user=rocky
k8s-node-3 ansible_host=192.168.1.13 ansible_user=rocky

[master]
k8s-node-1

[slave]
k8s-node-2
k8s-node-3
```

### 4. 一键部署

```bash
chmod +x quick_start.sh
./quick_start.sh
```

或手动执行：
```bash
ansible-playbook playbooks/site.yml -i inventory/hosts.ini
```

### 5. 验证集群

在 Master 节点执行：
```bash
kubectl get nodes -o wide
kubectl get pods -n calico-system
```

### 6. 部署可观测性 Agent（可选，Day 1）

若你已有 Loki / Prometheus 服务端，可为所有节点注入采集 Agent：

```bash
# 使用默认 Loki 地址（localhost:3100），适用于 Loki 与本机同节点部署场景
ansible-playbook playbooks/deploy_observability_agents.yml -i inventory/hosts.ini

# 或显式指定外部 Loki 地址
ansible-playbook playbooks/deploy_observability_agents.yml -i inventory/hosts.ini \
  -e promtail_loki_url=http://172.30.30.135:3100/loki/api/v1/push
```

> 注意：Playbook 执行时会自动从 GitHub Release 下载对应版本的 Promtail 与 Node Exporter 二进制，目标节点需具备外网访问能力。

**验证 Agent 状态：**
```bash
# 节点上检查服务
systemctl status promtail node-exporter

# 验证 Node Exporter 指标端点
curl http://localhost:9100/metrics

# 验证 Promtail 运行状态
curl http://localhost:9080/ready
```

---

## ⚙️ 核心配置解析

### 国内镜像加速（双重适配）

本项目对国内网络环境做了**软件包 + 容器镜像**的双重适配：

**1. Yum 软件源模板化（`template/repos/`）**
- K8s 与 CRI-O 的 `.repo` 文件通过 Jinja2 模板渲染到目标节点，避免手动维护多节点源配置

**2. 容器镜像 Registry 镜像（`template/containers/99-registries.conf.j2`）**
```toml
[[registry]]
prefix = "registry.k8s.io"
location = "swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io"
    [[registry.mirror]]
    location = "swr.cn-north-4.myhuaweicloud.com/ddn-k8s/registry.k8s.io"
```
- 仅影响 **CRI-O 拉取的镜像**（如 pause 沙箱）。kubeadm 直接拉取的镜像需在初始化时额外指定 `--image-repository` 进一步适配（可根据需要扩展）。

**3. Pause 镜像定制（`template/crio/20-pause.conf.j2`）**
```toml
[crio.image]
pause_image = "registry.k8s.io/pause:3.10.1"
```
- 与 `group_vars/k8s_nodes.yml` 中的 `pause_version` 联动，确保沙箱镜像版本与集群组件预期一致。

### 内核与网络调优

```yaml
# playbooks/group_vars/k8s_nodes.yml
k8s_sysctls:
  - { name: net.bridge.bridge-nf-call-iptables, value: "1" }
  - { name: net.bridge.bridge-nf-call-ip6tables, value: "1" }
  - { name: net.ipv4.ip_forward, value: "1" }
  - { name: net.ipv4.conf.all.forwarding, value: "1" }
```

- 网桥流量进入 iptables/netfilter，确保 K8s Service 网络、NetworkPolicy 能正常工作
- 开启 IPv4 路由转发，为 Pod 跨节点通信与 NAT 提供基础

### 可观测性 Agent 部署

本项目提供 **systemd 级别的裸机可观测性基线**，而非 DaemonSet 形态的容器化部署。选型理由：

- **生命周期独立**：Agent 的启停、升级不受 K8s 调度影响，即使 Kubelet 或容器运行时故障，节点级指标与系统日志依然持续采集
- **资源可见性真实**：Node Exporter 直接读取宿主机 `/proc` / `/sys`，避免容器隔离带来的统计偏差
- **二进制按需获取**：Ansible `get_url` 直接从 GitHub Release 拉取指定版本二进制，不预置在仓库中，避免 Git 大文件限制与跨架构分发问题

#### Node Exporter

```ini
# template/observability_agents/node-exporter/node-exporter.service.j2
ExecStart=/usr/local/bin/node_exporter
```

- 默认暴露 `9100` 端口，提供标准 Prometheus 节点指标
- 由 systemd 守护，崩溃后自动重启

#### Promtail

```yaml
# template/observability_agents/promtail/promtail-config.yaml.j2
clients:
  - url: {{ promtail_loki_url | default('http://localhost:3100/loki/api/v1/push') }}
    external_labels:
      hostname: {{ inventory_hostname }}

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/{messages,secure,cron,dnf.log}
```

- **Loki 地址模板化**：通过 `promtail_loki_url` 变量注入目标 Loki 地址，支持部署时动态指定
- **Hostname 标签自动注入**：利用 Ansible `inventory_hostname` 为每条日志附加节点标识
- **采集范围**：默认覆盖系统日志（`messages`、`secure`、`cron`、`dnf.log`）

### Swap 彻底禁用

K8s 要求节点禁用 Swap。本项目不仅注释 `/etc/fstab`，还通过 `blkid` + `swapoff UUID=` 确保**当前已激活的交换分区也被关闭**，最后以 `swapon -s` 做守门验证：

```yaml
- name: Verify swap is actually off
  ansible.builtin.command: swapon -s
  register: swap_check
  changed_when: false
  failed_when: swap_check.stdout != ""   # 若仍有输出，直接失败停止
```

### Calico 网络参数

```yaml
# deploy/calico/custom-resources-iptables.yaml
calicoNetwork:
  linuxDataplane: Iptables
  ipPools:
    - cidr: 10.244.0.0/16          # 与 kubeadm --pod-network-cidr 保持一致
      encapsulation: VXLAN         # 兼容绝大多数虚拟化/云环境二层网络
      blockSize: 26                # 每个节点分配 /26（64 个 IP），避免 IP 碎片
```

- **iptables 模式**：兼容 RHEL 8/9 默认内核（无需 eBPF 所需的高版本内核与特定编译选项）
- **VXLAN 封装**：适用于底层网络不支持 BGP 或 MAC 地址洪泛受限的虚拟化/云环境

---

## 🔮 已知局限与演进方向

| 局限 | 说明 | 演进方向 |
|---|---|---|
| **单 Master 拓扑** | 当前仅支持单控制平面节点，存在单点故障风险 | 引入 [kube-vip](https://kube-vip.io/) 或外部 LB + 多 Master 部署 |
| **iptables 数据面** | 相比 eBPF，iptables 在大规模规则下性能存在瓶颈 | 待内核升级后，可切换至 `custom-resources-bpf.yaml` 启用 eBPF 数据面 |
| **资源敏感** | 2C4G 虚拟化环境下组件启动较慢（CRI-O、Calico、Goldmane 等） | 属正常现象，生产环境建议 4C8G 以上 |
| **裸告警缺失** | 本项目已覆盖节点级采集 Agent（Promtail + Node Exporter），但服务端（Loki / Prometheus）与告警规则需外部部署 | 可与我个人的 **[node-observability-baseline](https://github.com/wamudus/node-observability-baseline)** 项目衔接，补齐服务端与可视化层 |

---

## 🔗 关联项目

- **[node-observability-baseline](https://github.com/wamudus/node-observability-baseline)**：基于 Docker Compose 的 Prometheus + Loki + Grafana + Alertmanager + 钉钉告警可观测性基线。可作为本集群节点级监控与日志聚合的配套方案。

---

## 📄 License

[MIT](LICENSE) © 2026 [wamudus]
