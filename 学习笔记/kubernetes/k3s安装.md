# MAC OSX下安装k3s

## 安装multipass

这是一个ubuntu系统的虚拟化管理工具，在Mac下使用hyperkit作为虚拟化驱动。

```bash
brew cask install multipass
```



## 使用multipass 创建虚拟机

```bash
# master节点
multipass launch --name k3s-master --cpus 1 --mem 1024M --disk 5G

# 3台node节点
multipass launch --name k3s-node1 --cpus 2 --mem 2048M --disk 10G
multipass launch --name k3s-node2 --cpus 2 --mem 2048M --disk 10G
multipass launch --name k3s-node3 --cpus 2 --mem 2048M --disk 10G

```

## 安装k3s

创建集群

```bash
# master 节点安装k3s
multipass exec k3s-master -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" sh -"
# 查看 token,并记录下来，用于其他节点加入此集群
multipass exec k3s-master sudo cat /var/lib/rancher/k3s/server/node-token
```

加入集群

```
multipass exec k3s-node1 -- /bin/bash -c "curl -sfL https://get.k3s.io | K3S_TOKEN=${K3S_TOKEN} K3S_URL=${K3S_NODEIP_MASTER} sh -"
```



