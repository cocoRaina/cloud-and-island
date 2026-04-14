# 用 Claude APP 远程操作 VPS

> 不在电脑前？打开手机上的 Claude，一样能管理你的服务器。

## 这是什么

你的记忆库和 Telegram bot 跑在 VPS 上，平时你用 Claude Code CLI 在终端里操作一切。但如果你不在电脑前呢？

这篇教程教你搭一个**VPS 工具箱 MCP Server**，然后把它接到手机上的 Claude APP（claude.ai/ios/android）。这样你随时随地打开 Claude APP 就能：

- 看服务器状态（CPU、内存、磁盘、各服务是否在线）
- 读日志（排查问题不用 SSH）
- 执行任意命令（装软件、重启服务、改配置）
- 查看和编辑文件
- 读写交接信

相当于把一个受保护的 SSH 塞进了 Claude APP 里。

---

## 架构

```
你的手机
    │
    │  Claude APP
    ▼
claude.ai（连接你的 MCP Server）
    │
    │  SSE 协议
    ▼
Caddy 反代（vps.yourdomain.com → 127.0.0.1:8003）
    │
    ▼
mcp_vps.py（FastMCP，端口 8003）
    │
    ▼
你的 VPS（执行命令、读写文件）
```

---

## 第一步：写工具箱代码

在你的项目目录（比如 `/www/memory-server/`）下创建 `mcp_vps.py`：

```python
"""
Claude的工具箱 - VPS管理 MCP Server
让Claude可以直接操作你的服务器
"""
from mcp.server.fastmcp import FastMCP
import subprocess
import os

mcp = FastMCP("VPS工具箱", host="127.0.0.1", port=8003)

# ═══════════════════════════════════════════
# 安全密钥 - 从服务器本地文件读取
# ═══════════════════════════════════════════
_TOKEN_FILE = "/root/.mcp_secret_token"
_SECURITY_TOKEN = ""
if os.path.exists(_TOKEN_FILE):
    with open(_TOKEN_FILE, "r") as f:
        _SECURITY_TOKEN = f.read().strip()


def _check_token(token: str) -> bool:
    """验证安全密钥"""
    if not _SECURITY_TOKEN:
        return False
    return token == _SECURITY_TOKEN
```

### 只读工具（不需要密钥）

```python
@mcp.tool()
def server_status() -> str:
    """
    查看服务器运行状态：系统负载、内存、磁盘、各服务是否在线。
    只读操作，不需要密钥。
    """
    checks = [
        ("系统负载", "uptime"),
        ("内存", "free -h | head -3"),
        ("磁盘", "df -h / | tail -1"),
        ("MCP记忆库(8000)", "ss -tlnp | grep ':8000' || echo '❌ 未运行'"),
        ("MCP工具箱(8003)", "ss -tlnp | grep ':8003' || echo '❌ 未运行'"),
        ("Caddy(443)", "ss -tlnp | grep ':443' || echo '❌ 未运行'"),
    ]
    lines = ["🖥️ 服务器状态：\n"]
    for name, cmd in checks:
        try:
            r = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=5)
            out = r.stdout.strip() or r.stderr.strip() or "无输出"
            lines.append(f"【{name}】\n{out}\n")
        except Exception:
            lines.append(f"【{name}】查询超时\n")
    return "\n".join(lines)


@mcp.tool()
def read_log(log_type: str = "mcp", tail: int = 50) -> str:
    """
    读取服务日志的最后几行。

    log_type: "mcp" 记忆库日志 | "vps" 工具箱日志 | "web" Web界面日志 | "caddy" Caddy日志
    tail: 读最后多少行，默认50，最多200
    """
    log_map = {
        "mcp": "/www/memory-server/logs_mcp.log",
        "vps": "/www/memory-server/logs_vps.log",
        "web": "/www/memory-server/logs_web.log",
        "caddy": "/var/log/caddy/access.log",
    }
    path = log_map.get(log_type, log_map["mcp"])
    n = min(tail, 200)

    if not os.path.exists(path):
        return f"日志文件不存在: {path}"
    try:
        r = subprocess.run(f"tail -n {n} '{path}'", shell=True, capture_output=True, text=True, timeout=5)
        content = r.stdout or "(空)"
        if len(content) > 8000:
            content = content[-8000:]
            content = "...(截断)...\n" + content
        return f"📋 {path} 最后 {n} 行：\n\n{content}"
    except Exception:
        return "❌ 读取日志失败"


@mcp.tool()
def view_file(filepath: str) -> str:
    """
    查看服务器上的文件内容。只读操作，不需要密钥。
    安全限制：只能查看项目目录下的文件。
    """
    base = "/www/memory-server/"
    if not filepath.startswith("/"):
        full = os.path.abspath(os.path.join(base, filepath))
    else:
        full = os.path.abspath(filepath)

    if not full.startswith(base):
        return f"❌ 安全限制：只能查看 {base} 目录下的文件"

    if not os.path.exists(full):
        return f"❌ 不存在: {full}"

    if os.path.isdir(full):
        items = sorted(os.listdir(full))
        parts = []
        for i in items:
            p = os.path.join(full, i)
            icon = "📁" if os.path.isdir(p) else "📄"
            parts.append(f"  {icon} {i}")
        return f"📁 {full}:\n\n" + "\n".join(parts)

    size = os.path.getsize(full)
    if size > 200000:
        return f"❌ 文件太大 ({size} bytes)"
    with open(full, "r", encoding="utf-8", errors="replace") as f:
        content = f.read()
    return f"📄 {full} ({size} bytes):\n\n{content}"
```

### 写入工具（需要密钥）

```python
@mcp.tool()
def run_command(command: str, token: str) -> str:
    """
    🔐 在服务器上执行Shell命令。需要安全密钥。
    这是万能工具——安装软件、编辑配置、管理进程，什么都能做。
    """
    if not _check_token(token):
        return "❌ 密钥错误，拒绝执行。"

    # 拦截毁灭性命令
    forbidden = ["rm -rf /", "mkfs.", "dd if=/dev/zero", "> /dev/sd"]
    for f in forbidden:
        if f in command:
            return "❌ 检测到危险命令，已拦截。"

    try:
        r = subprocess.run(
            command, shell=True, capture_output=True, text=True,
            timeout=30, cwd="/www/memory-server"
        )
        out = ""
        if r.stdout:
            out += r.stdout
        if r.stderr:
            out += f"\n[stderr]\n{r.stderr}" if out else r.stderr
        if not out.strip():
            out = "(命令已执行，无输出)"
        if len(out) > 8000:
            out = out[:8000] + "\n\n... (输出过长，已截断)"
        return f"$ {command}\n\n{out}"
    except subprocess.TimeoutExpired:
        return f"⏰ 命令超时（30秒限制）: {command}"
    except Exception as e:
        return f"❌ 执行失败: {e}"


@mcp.tool()
def write_file(filepath: str, content: str, token: str) -> str:
    """
    🔐 在服务器上创建或覆盖文件。需要安全密钥。
    安全限制：只能写项目目录下的文件。
    """
    if not _check_token(token):
        return "❌ 密钥错误，拒绝执行。"

    base = "/www/memory-server/"
    if not filepath.startswith("/"):
        full = os.path.abspath(os.path.join(base, filepath))
    else:
        full = os.path.abspath(filepath)

    if not full.startswith(base):
        return f"❌ 安全限制：只能写 {base} 目录下的文件"

    os.makedirs(os.path.dirname(full), exist_ok=True)
    with open(full, "w", encoding="utf-8") as f:
        f.write(content)
    return f"✅ 文件已写入: {full} ({len(content)} 字符)"


if __name__ == "__main__":
    mcp.run(transport="sse")
```

---

## 第二步：生成安全密钥

工具箱里有些工具很危险（`run_command` 可以执行任意命令），所以需要一个密钥来保护。

```bash
# 生成一个随机密钥
python3 -c "import secrets; print(secrets.token_urlsafe(32))" > /root/.mcp_secret_token
chmod 600 /root/.mcp_secret_token
cat /root/.mcp_secret_token  # 记下来，后面要用
```

密钥存在服务器本地文件里，MCP Server 启动时读取。只有提供正确密钥的工具调用才会被执行。

**重要**：不要把密钥存在记忆库里。每次新窗口需要用到密钥时，让你的人类伙伴从服务器上读取后告诉 Claude。

---

## 第三步：Caddy 反代

工具箱跑在 `127.0.0.1:8003`，需要通过 Caddy 暴露到公网，Claude APP 才能连上。

在你的 Caddyfile 里加一个子域名：

```
vps.yourdomain.com {
    reverse_proxy 127.0.0.1:8003 {
        header_up Host 127.0.0.1:8003
    }
}
```

然后重新加载 Caddy：

```bash
systemctl reload caddy
```

确保你的域名 DNS 已经把 `vps.yourdomain.com` 解析到你的 VPS IP。

---

## 第四步：systemd 服务

让工具箱开机自启、崩溃自动重启：

创建 `/etc/systemd/system/mcp_vps.service`：

```ini
[Unit]
Description=Claude VPS Tools MCP Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/www/memory-server
ExecStart=/www/server/python_manager/versions/3.12.0/bin/python3 /www/memory-server/mcp_vps.py
Restart=on-failure
RestartSec=5
StandardOutput=append:/www/memory-server/logs_vps.log
StandardError=append:/www/memory-server/logs_vps.log

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable mcp_vps
systemctl start mcp_vps
```

注意 `User=root`——因为工具箱需要 root 权限来执行命令。安全性靠密钥机制保障。

---

## 第五步：连接 Claude APP

1. 打开 Claude APP（手机或 claude.ai 网页版）
2. 进入设置 → MCP Servers
3. 添加一个新的 SSE 类型的 MCP Server：
   - URL：`https://vps.yourdomain.com/sse`
4. 保存后，Claude APP 会自动连接并发现工具箱里的工具

连接成功后，你可以直接对 Claude 说："看一下服务器状态"，或者"帮我重启记忆库服务"。

---

## 安全注意事项

1. **密钥不要外泄**：谁有密钥，谁就能在你的服务器上执行任意命令
2. **密钥不要存记忆库**：记忆库的内容可能被日志或其他工具读取
3. **危险命令拦截**：代码里已经写了基本的拦截（`rm -rf /` 等），你可以根据需要加更多
4. **文件操作限制**：`view_file` 和 `write_file` 只能操作项目目录下的文件
5. **命令超时**：`run_command` 有 30 秒超时限制，防止卡死
6. **HTTPS**：通过 Caddy 自动获取 SSL 证书，所有通信加密传输

---

## 你可以用它做什么

一些实际的使用场景：

- **"服务器状态怎么样"** → Claude 调用 `server_status`，告诉你各服务是否正常
- **"看看记忆库日志最后 20 行"** → Claude 调用 `read_log`
- **"帮我装个 htop"** → Claude 调用 `run_command("dnf install -y htop")`
- **"重启记忆库"** → Claude 调用 `run_command("systemctl restart mcp_server")`
- **"看看 Caddyfile 写了什么"** → Claude 调用 `view_file("Caddyfile")`

不用 SSH，不用开电脑，手机上就能管理你的服务器。
