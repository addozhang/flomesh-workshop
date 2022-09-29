# 001 环境搭建

## 下载安装

### 下载二进制包

<details>
  <summary>GitHub Release 页面下载</summary>


从 Pipy 的 [release 页面](https://github.com/flomesh-io/pipy/releases) 下载。

```shell
wget https://github.com/flomesh-io/pipy/releases/download/0.50.0-25/pipy-0.50.0-25-generic_linux-x86_64.tar.gz
tar -zxf pipy-0.50.0-25-generic_linux-x86_64.tar.gz
cp ./usr/local/bin/pipy /usr/local/bin/pipy
```
</details>

### 从源码构建

<details>
  <summary>从源码构建</summary>

NodeJS

```shell
sudo apt update
curl -sL https://deb.nodesource.com/setup_14.x | sudo bash -
sudo apt -y install nodejs
```

CMake

```shell
sudo apt install -y cmake clang
```

下载源码

```shell
sudo apt install -y git
git clone https://github.com/flomesh-io/pipy.git
```

构建

```shell
cd pipy
#-g 构建内置的 admin console
./build.sh -g
```

在 `build.sh` 脚本中有更多参数说明，比如 `./build.sh -g` 将构建带有管理界面的 Pipy。

</details>

### 容器

```shell
docker run -p 8080:8080 flomesh/pipy:latest
```

## Admin Console 的使用

```shell
pipy
```

浏览器中打开 `http://localhost:6060`

## 工作模式

```shell
pipy tutorial/01-hello/hello.js
pipy http://localhost:6060/repo/tutorial/01-hello/
```
