# VPS 搭建 Xray Reality 工作流程（v2）

## 给 CC 的说明
这是咪咪的新 VPS（CstoneCloud 美国家宽），需要先验证 IP 质量，再搭建 Xray Reality 代理。请按步骤执行，每一步完成后确认再继续。

---

## 第一步：连接 VPS

```bash
ssh root@<VPS_IP>
# 密码：（咪咪提供）
```

确认系统：
```bash
cat /etc/os-release
```
应该显示 Debian 12。

---

## 第二步：验证 IP 质量（搭建之前必做）

### 检查 IP 基本信息
```bash
curl https://ipinfo.io
```
确认：
- `country` 是 `US`
- `org` 里显示的是 ISP 名称（不是 hosting/datacenter）

### 检查是否为住宅 IP
```bash
curl "https://ip.ping0.cc" 2>/dev/null | head -50
```

### 测速
```bash
apt update && apt install -y speedtest-cli
speedtest --simple
```
确认下载和上传速度与标称的 100Mbps 带宽大致匹配。

### 检测结果判断
- ✅ 类型显示 ISP/Residential → 继续搭建
- ❌ 类型显示 Hosting/Datacenter → 停止，联系客服换 IP 或退款

**把检测结果截图发给咪咪确认后再继续。**

---

## 第三步：系统更新 & 基础工具

```bash
apt update && apt upgrade -y
apt install -y curl wget socat ufw
```

---

## 第四步：开启 BBR 加速

```bash
echo "net.core.default_qdisc=fq" >> /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" >> /etc/sysctl.conf
sysctl -p
```

验证：
```bash
sysctl net.ipv4.tcp_congestion_control
# 应该显示 bbr
```

---

## 第五步：安装 Xray

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

验证：
```bash
xray version
```

---

## 第六步：生成密钥和参数

### 生成 X25519 密钥对
```bash
xray x25519
```
**记下 Private key 和 Public key。**

### 生成 UUID
```bash
xray uuid
```
**记下 UUID。**

### 生成 shortId
```bash
openssl rand -hex 8
```
**记下 shortId。**

---

## 第七步：配置 Xray 服务端

```bash
cat > /usr/local/etc/xray/config.json << 'EOF'
{
  "log": {
    "loglevel": "warning"
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "dest": "www.microsoft.com:443",
          "serverNames": [
            "www.microsoft.com"
          ],
          "privateKey": "<PRIVATE_KEY>",
          "shortIds": [
            "<SHORT_ID>"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
EOF
```

**CC 自行替换 `<UUID>`、`<PRIVATE_KEY>`、`<SHORT_ID>` 为第六步生成的值。**

---

## 第八步：配置防火墙

```bash
ufw allow 22/tcp
ufw allow 443/tcp
ufw --force enable
ufw status
```

---

## 第九步：启动 Xray

```bash
systemctl restart xray
systemctl enable xray
systemctl status xray
```

确认状态是 `active (running)`。

---

## 第十步：生成客户端配置

### 整理连接信息

```
协议：VLESS
地址：<VPS_IP>
端口：443
UUID：<UUID>
流控：xtls-rprx-vision
传输协议：tcp
安全：reality
SNI：www.microsoft.com
指纹：chrome
Public Key：<PUBLIC_KEY>
shortId：<SHORT_ID>
```

---

### Clash Meta（mihomo）配置

在咪咪的 Clash 配置文件中添加以下 proxy 节点：

```yaml
proxies:
  - name: "US-Reality"
    type: vless
    server: <VPS_IP>
    port: 443
    uuid: <UUID>
    network: tcp
    udp: true
    tls: true
    flow: xtls-rprx-vision
    servername: www.microsoft.com
    reality-opts:
      public-key: <PUBLIC_KEY>
      short-id: <SHORT_ID>
    client-fingerprint: chrome
```

把这段加到咪咪现有的 Clash 配置文件的 proxies 部分，然后在 proxy-groups 里引用 `US-Reality` 即可。

### 安卓备选：v2rayNG / NekoBox

如果 Clash Meta 有问题，安卓上装 v2rayNG 或 NekoBox：
1. 添加服务器 → VLESS
2. 填入上面的连接信息
3. 连接测试

---

## 第十一步：验证连接

连上代理后测试：
1. 浏览器打开 https://ip.sb → 确认显示 VPS 的 IP
2. 打开 https://claude.ai → 确认能正常访问
3. 打开 https://www.google.com → 确认能正常搜索
4. 打开 https://ipinfo.io → 再次确认 IP 类型是 ISP

---

## 安全加固（可选，搭好能用之后再做）

```bash
# 修改 SSH 端口
sed -i 's/#Port 22/Port 29622/' /etc/ssh/sshd_config
ufw allow 29622/tcp
systemctl restart sshd
# 之后 SSH 连接用 ssh -p 29622 root@<VPS_IP>
```

---

## 注意事项
- 伪装域名（www.microsoft.com）可换成 www.apple.com、www.amazon.com 等大站
- 如果连不上，先 `systemctl status xray` 检查有没有报错
- 如果 IP 被墙，联系 CstoneCloud 客服换 IP（24小时内未封可退款）
- Clash 必须是 Meta（mihomo）内核，原版 Clash 不支持 VLESS Reality
