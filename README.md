
# 使用 Ansible 自动化批量安装 GPU 显卡驱动（NVIDIA Driver + CUDA）完整教程

适用于：Ubuntu 22.04 / 多台 GPU 服务器 / 自动化批量部署场景

在 GPU 服务器集群中，人工逐台部署 NVIDIA 驱动与 CUDA 不仅耗时，还容易出现版本不一致的问题。
本文将演示如何通过 **Ansible** 实现 GPU 驱动的自动化批量安装，使你可以：

* 一次命令，批量安装 GPU 驱动
* 统一版本管理（Driver + CUDA）
* 支持集群并行执行
* 可指定单节点或多节点
* 无需人工输入密码（SSH 免密登录）

---

# 基于 Ubuntu/Ubuntu (WSL)/aptitude 的系统安装 Ansible

```bash
sudo apt -y update && sudo apt -y install ansible python3-pip
```

# 一、准备工作：构建 SSH 免密访问

为了让 Ansible 能自动登录每台 GPU 服务器，必须先配置免密登录。

### 1. 在控制节点生成 SSH 密钥

```bash
ssh-keygen -t rsa -b 4096 -C "192.168.1.3"
```

保持默认回车即可，生成：

* 私钥：`~/.ssh/id_rsa`
* 公钥：`~/.ssh/id_rsa.pub`

查看公钥内容（如需要）：

```bash
cat ~/.ssh/id_rsa.pub
```

---

### 2. 将公钥复制到 GPU 服务器

例如 GPU 节点 IP：`172.16.1.3`

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub root@172.16.1.3
```

成功后即可无密码登录：

```bash
ssh root@172.16.1.3
```

按相同方式向其它 GPU 节点复制即可。

---

# 二、准备 Ansible Inventory（主机清单）

```yaml
all:
  hosts:
    ubuntu-gpu-5090:
      ansible_host: 172.16.1.3
      ansible_user: root
    ubuntu-gpu-4090:
      ansible_host: 172.16.1.4
      ansible_user: root
```

你可以根据 GPU 节点数量继续扩展，并使用组（group）管理：

```yaml
gpu-nodes:
  hosts:
    ubuntu-gpu-5090:
    ubuntu-gpu-4090:
    ubuntu-gpu-3090:
```

---

# 三、准备 Ansible Playbook：自动安装 NVIDIA Driver + CUDA

* 移除旧版驱动
* 自动禁用 nouveau
* 安装指定版本 NVIDIA 驱动
* 安装指定版本 CUDA
* 自动重启
* 检查 `nvidia-smi` 状态

示例结构（与你的文件一致即可）：

```yaml
---
- hosts: all
  become: yes
  vars:
    ubuntu_major: "{{ ubuntu_major }}"
    ubuntu_minor: "{{ ubuntu_minor }}"
    cuda_version: "{{ cuda_version }}"
    nvidia_version: "{{ nvidia_version }}"
  tasks:
    - name: Install required packages
      apt:
        name: ['build-essential','gcc','make','wget']
        state: present
        update_cache: yes

    - name: Install Nvidia driver
      apt:
        name: "nvidia-driver-{{ nvidia_version }}"
        state: present

    - name: Install CUDA Toolkit
      apt:
        name: "cuda-{{ cuda_version }}"
        state: present

    - name: Reboot if required
      reboot:
        test_command: nvidia-smi
```

你上传的文件可直接使用。

---

# 四、执行批量安装（Ansible Playbook）

安装命令有两种方式：

---

## **方式一：指定单节点安装（推荐测试时使用）**

例如仅安装到 `ubuntu-gpu-5090` 这一台：

```bash
ansible-playbook -i inventory.yml upgrade-nvidia.yml --limit ubuntu-gpu-5090 \
  -e "ubuntu_major=22 ubuntu_minor=04 cuda_version=12-9 nvidia_version=575"
```

说明：

* `--limit ubuntu-gpu-5090`：只对这台机器执行
* `cuda_version=12-9`：CUDA 12.9 ：版本可自行修改
* `nvidia_version=575`：NVIDIA Driver 575

非常适合测试单 Node。

---

## **方式二：批量安装到整个集群（生产环境使用）**

直接执行：

```bash
ansible-playbook -i inventory.yml upgrade-nvidia.yml \
  -e "ubuntu_major=22 ubuntu_minor=04 cuda_version=12-9 nvidia_version=575"
```

不加 `--limit` 时，将自动安装到 inventory 中的所有 GPU 服务器。

如果你在 inventory 中定义了一个 group，比如：

```yaml
gpu-nodes:
  hosts:
    ubuntu-gpu-5090:
    ubuntu-gpu-4090:
```

则可以：

```bash
ansible-playbook -i inventory.yml upgrade-nvidia.yml --limit gpu-nodes \
  -e "ubuntu_major=22 ubuntu_minor=04 cuda_version=12-9 nvidia_version=575"
```

---

# 五、安装完成后验证效果

Ansible 已包含重启（如果需要），安装完成后，你可以登录任意节点：

```bash
ssh root@172.16.1.3
nvidia-smi
```

输出类似：

```
Fri Dec 12 15:30:21 2025
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 575.xx                 Driver Version: 575.xx   CUDA Version: 12.9         |
+---------------------------------------------------------------------------------------+
| GPU Name             Persistence-M| Bus-Id      Disp.A | Volatile Uncorr. ECC         |
| 0   NVIDIA RTX 5090        Off    | 00000000:01:00.0 Off |                  0         |
+---------------------------------------------------------------------------------------+
```

表示驱动安装成功。

---

# 六、常见操作说明

### ■ 批量重试执行

安装过程中遇到失败，可重新执行同样指令，Ansible 会自动跳过已完成任务。

### ■ 新增 GPU 节点

只需在 `inventory.yml` 中加入服务器信息即可。

### ■ 更换驱动版本

只需更换版本参数，例如：

```bash
-e "nvidia_version=550"
```

无需修改 Playbook。

---

# 七、完整流程总结

1. 生成 SSH 密钥并复制公钥到所有 GPU 服务器
2. 准备 Ansible inventory
3. 使用 `upgrade-nvidia.yml` Playbook
4. 指定执行范围（单节点 / 多节点）
5. 一条命令完成 Driver + CUDA 安装
6. `nvidia-smi` 验证结果

---
