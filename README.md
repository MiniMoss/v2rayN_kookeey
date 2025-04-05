# v2rayN链式代理配置流程（kookeey）
假设你想在 v2rayN（Windows 上的 v2ray 客户端）中配置链式代理，并使用“kookeey 代理 IP”作为前置代理。如果你说的“kookeey”是一个特定的代理服务提供商（例如提供 SOCKS5 或 HTTP 代理 IP），以下是如何实现的具体步骤。

---

### 前提条件

- **v2rayN 已安装**：确保使用最新版 v2rayN（可从 GitHub 下载，例如 v6.x 版本）。
- **kookeey 代理 IP**：假设你已从 kookeey 获取代理信息，例如：
    - 类型：SOCKS5
    - 地址：`192.168.1.100`（示例 IP）
    - 端口：`1080`
    - 用户名/密码（可选）：`user` / `pass`
- **后置代理**：假设你有一个目标代理服务器，例如 VMess：
    - 地址：`v2ray.example.com`
    - 端口：`443`
    - ID：`你的UUID`
- **目标**：通过 kookeey 代理 IP（前置）连接到 VMess 服务器（后置），形成链式代理。

---

### 设置步骤

### 1. **添加 kookeey 代理 IP（前置代理）**

- 打开 v2rayN，点击主界面右下角的“服务器”按钮。
- 点击“添加服务器” -> 选择“手动输入 SOCKS”。
- 输入 kookeey 代理信息：
    
    ```
    服务器地址: 192.168.1.100
    端口: 1080
    用户名: user（如果有）
    密码: pass（如果有）
    备注: kookeey-front（自定义名称）
    
    ```
    
- 保存后，记下备注名（例如 `kookeey-front`）。

### 2. **添加后置代理（VMess）**

- 点击“添加服务器” -> 选择“手动输入 VMess”。
- 输入后置代理信息：
    
    ```
    地址: v2ray.example.com
    端口: 443
    用户ID: 你的UUID
    加密方式: auto
    传输协议: tcp
    TLS: 启用（根据实际服务器配置）
    备注: back-proxy（自定义名称）
    
    ```
    
- 保存后，记下备注名（例如 `back-proxy`）。

### 3. **编辑配置文件实现链式代理**

v2rayN 使用 JSON 格式配置文件来实现链式代理。

- 在 v2rayN 主界面，点击“参数设置” -> “编辑当前配置文件” 或 “打开配置文件路径”。
- 用文本编辑器（如 Notepad++）打开 `config.json` 文件。
- 修改或添加以下内容（根据你的实际信息调整）：

```json
{
  "inbounds": [
    {
      "port": 10808, // 本地 SOCKS 代理端口
      "protocol": "socks",
      "settings": {
        "udp": true
      }
    },
    {
      "port": 10809, // 本地 HTTP 代理端口
      "protocol": "http"
    }
  ],
  "outbounds": [
    {
      "tag": "kookeey-front",
      "protocol": "socks",
      "settings": {
        "servers": [
          {
            "address": "192.168.1.100",
            "port": 1080,
            "users": [
              {
                "user": "user",
                "pass": "pass"
              }
            ] // 如果无认证则删除 users 部分
          }
        ]
      }
    },
    {
      "tag": "back-proxy",
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
            "address": "v2ray.example.com",
            "port": 443,
            "users": [
              {
                "id": "你的UUID",
                "alterId": 0,
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "serverName": "v2ray.example.com"
        }
      },
      "proxySettings": {
        "tag": "kookeey-front" // 关键字段，指定通过 kookeey 代理连接
      }
    },
    {
      "tag": "direct",
      "protocol": "freedom"
    }
  ],
  "routing": {
    "rules": [
      {
        "type": "field",
        "outboundTag": "back-proxy" // 所有流量走链式代理
      }
    ]
  }
}

```

- **关键点**：
    - `kookeey-front` 是 kookeey 代理 IP 的前置节点。
    - `back-proxy` 是后置 VMess 节点，通过 `proxySettings.tag` 指定其流量经过 `kookeey-front`。
    - `routing.rules` 确保所有流量走链式代理。

### 4. **保存并重启 v2rayN**

- 保存修改后的 `config.json` 文件。
- 返回 v2rayN，点击“重启服务”或关闭并重新打开 v2rayN，让配置生效。

### 5. **测试连接**

- 在 v2rayN 主界面，确认当前模式为“全局模式”（或手动选择 `back-proxy`）。
- 打开浏览器，设置代理为 `127.0.0.1:10808`（SOCKS5），访问 `ipinfo.io`，检查出口 IP 是否为后置代理（`v2ray.example.com`）的 IP。
- 如果配置正确，流量会先经过 kookeey 代理 IP，再到达 VMess 服务器。

---

### 注意事项

- **kookeey 代理类型**：如果 kookeey 提供的是 HTTP 代理而非 SOCKS5，将 `protocol` 改为 `http`，并确保端口和认证信息正确。
- **性能**：链式代理会增加延迟，确保 kookeey 代理 IP 速度快且稳定。
- **调试**：如果连接失败，查看 v2rayN 的“日志”窗口，排查错误（例如 kookeey IP 不可用或 VMess 配置错误）。
- **路由规则**（可选）：若只想部分流量走链式代理，可调整 `routing.rules`，例如：
    
    ```json
    "rules": [
      {
        "type": "field",
        "domain": ["google.com", "youtube.com"],
        "outboundTag": "back-proxy"
      },
      {
        "type": "field",
        "outboundTag": "direct"
      }
    ]
    
    ```
    
- **安全性**：确认 kookeey 代理服务可信，避免流量泄露。

---

### 示例场景

假设你的 kookeey 代理 IP 是 `192.168.1.100:1080`（SOCKS5），后置服务器是 `v2ray.example.com:443`（VMess），通过上述配置，流量会先经过 kookeey 代理，再通过 VMess 访问目标网站。这种设置适用于绕过本地限制或增强隐私。

