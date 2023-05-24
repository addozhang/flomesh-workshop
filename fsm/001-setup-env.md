# 环境搭建

**本课程推荐环境及配置仅适用于实验用。**

推荐环境：
* 标准 Linux 发行版，如 CentOS、Debain、Ubuntu
* Kubernetes 发行版，如 k3s、k8e 

这里使用 k3s 快速搭建单节点的集群，执行下面的命令：

> 这里的集群配置仅限实验环境中。
> 安装过程会同时安装 kubectl CLI，无需单独手动安装。

```shell
export INSTALL_K3S_VERSION=v1.23.8+k3s1
curl -sfL https://get.k3s.io | sh -s - --disable traefik --write-kubeconfig-mode 644 --write-kubeconfig ~/.kube/config
```

