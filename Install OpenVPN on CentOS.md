# OpenVPN server installation on CentOS stream 9

## 一、OpenVPN 证书制作
工具：`easy-rsa`
```bash
git clone https://github.com/OpenVPN/easy-rsa.git
# 或者
wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.1.7/EasyRSA-3.1.7.tgz
# 或者
yum install epel-release
yum install easy-rsa
```
使用说明在`README.quickstart.md`文件中
```bash
# 第一步修改vars文件，然后初始化pki
./easyrsa init-pki
# 生成ca证书, [nopass]为可选参数
./easyrsa build-ca [nopass]
# 生成dh交换密钥
./easyrsa gen-dh
# 生成客户端证书
./easyrsa build-client-full client [nopass]
# 生成服务端证书
./easyrsa build-server-full server [nopass]
```
在`pki`目录下可以找到所有证书和密钥

## 二、配置OpenVPN服务端
1. 安装
```bash
yum install epel-release
yum install openvpn
```
2. 配置openvpn
```bash
# 把配置文件模板复制过来
cp /usr/share/doc/openvpn/sample/sample-config-files/server.conf /etc/openvpn/server
# 新建keys目录，存放证书
cd /etc/openvpn/server && mkdir keys
cp ca.crt /etc/openvpn/server/keys/
cp dh.pem /etc/openvpn/server/keys/
cp issued/server.crt /etc/openvpn/server/keys/
cp private/server.key /etc/openvpn/server/keys/
```
修改`server.conf`文件下列地方：
```
ca ca.crt
cert server.crt
key server.key

dh dh2048.pem

;push "route 192.168.10.0 255.255.255.0"
;push "route 192.168.20.0 255.255.255.0"

tls-auth ta.key 0

cipher AES-256-CBC
```
3. 启用路由转发功能
```bash
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p
```
4. 建立`ta.key`文件（拒绝服务攻击证书文件）
```bash
openvpn --genkey tls-auth ta.key
```
5. 启动服务
```bash
openvpn --daemon --config server.conf --askpass
```

## 三、配置OpenVPN客户端
```bash
# 把客户端配置文件复制到某个目录，比如client
cp /usr/share/doc/openvpn/sample/sample-config-files/client.conf /root/client/
# 把相关证书复制到同一目录
cp ca.crt /root/client/
cp client.crt /root/client/
cp client.key /root/client/keys/
cp ta.key /root/client/
```
修改`client.conf`文件下列位置：
```
remote my-server-1 1194

cipher AES-256-CBC
```
```bash
# 客户端配置文件稍后用于Windows下，故需要改后缀名
mv client.conf client.ovpn

# 打包后发到windows上
zip client.zip client/*
```

## 四、Windows客户端上测试
## 五、测试内网连通性
会发现无法ping通内网其他主机，原因是内网主机没有回来的路由， 解决方法有2个， 一个是单独给内网主机设置静态路由（网关）， 第二个是推荐做法，在openvpn服务器上配置iptables的nat功能。 

静态路由：
```bash
ip route add 10.8.0.0/24 via 192.168.1.3 dev nm-bridge
```
iptables的nat功能：
```bash
iptables -t nat --list
iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -j MASQUERADE
```

## 六、PAM认证（optional）
1. 新增openvpn的pam profile文件`/etc/pam.d/openvpn`:
```bash
[root@localhost pam.d]# cat /etc/pam.d/openvpn 
auth	required	pam_unix.so shadow nodelay
account	required	pam_unix.so
```
2. 在openvpn服务器配置文件`server.conf`新增:
```
# enable pam auth
plugin /usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so openvpn
```
3. 在客户端配置文件`client.ovpn`新增：
```
auth-user-pass
```
4. 创建pam认证用户
```bash
useradd -g openvpn -s /bin/false testuser
passwd testuser
```
