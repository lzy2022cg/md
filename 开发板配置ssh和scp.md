# 开发板配置 SSH 和 SCP

## 第一阶段：在 WSL 上交叉编译 dropbear

### 第一步：下载源码
```bash
cd ~
wget https://matt.ucc.asn.au/dropbear/releases/dropbear-2022.83.tar.bz2
tar xjf dropbear-2022.83.tar.bz2
cd dropbear-2022.83
```

### 第二步：交叉编译（静态链接，避免库依赖问题）
```bash
./configure --host=arm-linux \
    --disable-zlib \
    --disable-wtmp \
    --disable-lastlog \
    CC=arm-linux-gcc

make PROGRAMS="dropbear dropbearkey scp" LDFLAGS="-static"
```

编译完成后当前目录会有 `dropbear`、`dropbearkey`、`scp` 三个二进制文件。

---

## 第二阶段：传输到开发板

### 第一步：WSL 开启 HTTP 服务（在 dropbear 编译目录下）
```bash
python3 -m http.server 8080
```
> 注意：Windows 需要关闭防火墙，或放通 8080 端口

### 第二步：开发板下载并安装
```bash
wget http://192.168.137.1:8080/dropbear -O /usr/sbin/dropbear
wget http://192.168.137.1:8080/dropbearkey -O /usr/bin/dropbearkey
wget http://192.168.137.1:8080/scp -O /usr/bin/scp

chmod +x /usr/sbin/dropbear /usr/bin/dropbearkey /usr/bin/scp
```

### 第三步：生成主机密钥
```bash
mkdir -p /etc/dropbear
dropbearkey -t rsa -f /etc/dropbear/dropbear_rsa_host_key
dropbearkey -t dss -f /etc/dropbear/dropbear_dss_host_key
```

### 第四步：启动 SSH 服务
```bash
dropbear -B    # -B 表示允许空密码登录
```

---

## 日常使用

> **核心原则：谁被连接，谁就是服务器，服务器必须先开启 SSH 服务**
>
> - 开发板作为服务器：`dropbear -B`
> - WSL 作为服务器：`sudo service ssh start`（首次需要安装：`sudo apt install openssh-server -y`）

---

### WSL → 开发板（GEC6818可用）
开发板是服务器，开启SSH服务  dropbear -B
```bash
# 传单个文件
scp main.c root@192.168.137.100:/IOT

# 传文件夹
scp -r ./文件夹名 root@192.168.137.100:/IOT
```

### 开发板 → WSL
WSL2是服务器，开启SSH服务  service ssh start
```bash
# 传单个文件
scp /IOT/test.c root@192.168.137.1:~/

# 传文件夹
scp -r /IOT/文件夹名 root@192.168.137.1:~/
```

### WSL 从开发板取文件（GEC6818可用）
开发板是服务器，开启SSH服务  dropbear -B
```bash
# 取单个文件
scp root@192.168.137.100:/IOT/test.c ~/

# 取文件夹
scp -r root@192.168.137.100:/IOT/文件夹名 ~/
```



当更换了开发板时把旧的密钥记录删掉就行。

bash

```bash
ssh-keygen -f "/root/.ssh/known_hosts" -R "192.168.137.10"
```
