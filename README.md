Kubernetes 集群自动化部署（Ansible + Kubeadm + CRI-O + Calico）。

## 技术栈

- **容器运行时**: CRI-O（Systemd 集成，Cgroup v2 原生支持）
- **网络插件**: Calico v3.31（iptables 模式，兼容 RHEL 8/9 内核）
- **编排工具**: Kubeadm + Ansible（声明式基础设施）
- **合规**: 通过 `ansible-lint` production profile 审查

## 快速开始

```bash
# 1. 安装依赖
ansible-galaxy collection install -r requirements.yml
pip install passlib  # 用于账户管理密码哈希

# 2. 配置集群节点
cp inventory/hosts.ini.example inventory/hosts.ini
# 编辑 hosts.ini 填入你的节点 IP/主机名

# 3. 部署
chmod +x quick-start.sh
./quick-start.sh
```

## 架构

-  Day 0: 节点初始化（禁用 Swap、SELinux、防火墙、内核模块)
-  Day 0: 依赖安装（CRI-O、Kubeadm、Kubelet）
-  Day 0: 控制平面初始化 + Calico CNI
-  Day 0: 工作节点加入集群

## 已知局限

- 单 Master 拓扑（HA 需引入 kube-vip）
- 2核4G 虚拟化环境下启动较慢（属正常现象）
- 使用 iptables 而非 eBPF（为兼容旧内核）

## License
[MIT](LICENSE) © 2026 [wamudus]
