### install
在 **Linux 服务器** 上配置 **Squid** 代理服务器的完整步骤如下：

---

## **1. 安装 Squid**
### **Ubuntu/Debian**
```bash
sudo apt update
sudo apt install squid -y
```

### **CentOS/RHEL**
```bash
sudo yum install epel-release -y  # CentOS 7/8 需要 EPEL 源
sudo yum install squid -y
```

### **启动并设置开机自启**
```bash
sudo systemctl start squid
sudo systemctl enable squid
```

---

## **2. 配置 Squid（基础代理）**
### **编辑配置文件**
```bash
sudo nano /etc/squid/squid.conf
```

### **常用配置选项**
#### **(1) 修改监听端口（默认 `3128`）**
```ini
http_port 3128
```
可改为其他端口（如 `8080`）：
```ini
http_port 8080
```

#### **(2) 允许特定 IP 访问代理**
```ini
acl allowed_ips src 192.168.1.0/24  # 允许局域网段
acl allowed_ips src 123.45.67.89    # 允许单个 IP
http_access allow allowed_ips
http_access deny all                # 拒绝其他所有 IP
```

#### **(3) 启用身份验证（用户名/密码）**
1. 安装 `htpasswd` 工具：
   ```bash
   sudo apt install apache2-utils -y  # Ubuntu/Debian
   sudo yum install httpd-tools -y    # CentOS/RHEL
   ```
2. 创建密码文件：
   ```bash
   sudo htpasswd -c /etc/squid/passwords proxy_user  # 设置用户名和密码
   ```
3. 在 `squid.conf` 中添加：
   ```ini
   auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwords
   auth_param basic realm proxy
   acl authenticated proxy_auth REQUIRED
   http_access allow authenticated
   ```

#### **(4) 限制访问网站**
```ini
acl blocked_sites dstdomain .facebook.com .twitter.com
http_access deny blocked_sites
```

---

## **3. 重启 Squid 生效**
```bash
sudo systemctl restart squid
```

---

## **4. 防火墙放行 Squid 端口**
### **UFW (Ubuntu)**
```bash
sudo ufw allow 3128/tcp
```

### **Firewalld (CentOS)**
```bash
sudo firewall-cmd --permanent --add-port=3128/tcp
sudo firewall-cmd --reload
```

### **iptables (通用)**
```bash
sudo iptables -A INPUT -p tcp --dport 3128 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4  # 永久保存规则
```

---

## **5. 客户端使用代理**
### **Linux/Mac/Windows**
- **命令行临时使用**：
  ```bash
  export http_proxy="http://用户名:密码@服务器IP:3128"
  export https_proxy="http://用户名:密码@服务器IP:3128"
  ```
- **浏览器/系统设置**：
  - 手动填入代理 `服务器IP:3128` 和认证信息。

---

## **6. 高级配置**
### **(1) 透明代理（网关模式）**
```ini
http_port 3128 transparent
```
需配合 iptables 重定向流量：
```bash
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128
```

### **(2) 缓存优化**
```ini
cache_dir ufs /var/spool/squid 5000 16 256  # 缓存大小 5GB
maximum_object_size 256 MB                  # 最大缓存文件
refresh_pattern . 1440 20% 10080           # 缓存刷新策略
```

### **(3) 日志管理**
```ini
access_log /var/log/squid/access.log
cache_log /var/log/squid/cache.log
```

---

## **7. 常见问题**
### **Q1: 代理连接失败？**
- 检查 Squid 是否运行：
  ```bash
  sudo systemctl status squid
  ```
- 检查防火墙是否放行端口：
  ```bash
  sudo netstat -tulnp | grep squid
  ```

### **Q2: 如何查看访问日志？**
```bash
sudo tail -f /var/log/squid/access.log
```

### **Q3: 如何完全卸载 Squid？**
```bash
sudo apt purge squid -y          # Ubuntu/Debian
sudo yum remove squid -y         # CentOS/RHEL
sudo rm -rf /etc/squid /var/spool/squid
```

---

## **总结**
| 功能               | 配置示例                          |
|--------------------|----------------------------------|
| **修改端口**       | `http_port 8080`                |
| **IP 白名单**      | `acl allowed_ips src 1.2.3.4`   |
| **用户名密码认证** | `htpasswd` + `basic_ncsa_auth`  |
| **屏蔽网站**       | `acl blocked_sites dstdomain .facebook.com` |
| **透明代理**       | `http_port 3128 transparent`    |

按需调整配置后，Squid 可满足 **企业级代理、内容过滤、加速缓存** 等需求。

----

```bash
FATAL: Authentication helper program /usr/lib/squid/basic_ncsa_auth: (2) No such file or directory
```
这个错误表明 **Squid 找不到认证辅助程序** `basic_ncsa_auth`。以下是解决方法：

---

### **1. 确认认证工具是否安装**
Squid 的认证模块通常需要额外安装：
#### Ubuntu/Debian
```bash
sudo apt install squid-common
```
#### CentOS/RHEL
```bash
sudo yum install squid
```

---

### **2. 查找正确的路径**
运行以下命令查找 `basic_ncsa_auth` 的实际路径：
```bash
sudo find / -name basic_ncsa_auth 2>/dev/null
```
常见路径：
- `/usr/lib/squid/basic_ncsa_auth` (Ubuntu/Debian)
- `/usr/lib64/squid/basic_ncsa_auth` (CentOS/RHEL)

---

### **3. 修改 Squid 配置**
编辑配置文件：
```bash
sudo nano /etc/squid/squid.conf
```
将路径改为实际路径（例如 CentOS）：
```ini
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwords
```

---

### **4. 重启 Squid**
```bash
sudo systemctl restart squid
```

---

### **5. 如果仍然缺失**
#### 方案1：编译安装辅助程序
```bash
sudo apt install squid-common --reinstall
```
#### 方案2：改用其他认证方式
```ini
auth_param basic program /usr/lib/squid/basic_getpwnam_auth
```

---

### **验证步骤**
1. 检查认证文件权限：
   ```bash
   sudo chmod 644 /etc/squid/passwords
   ```
2. 测试认证：
   ```bash
   /usr/lib/squid/basic_ncsa_auth /etc/squid/passwords
   ```
   输入用户名密码看是否返回 `OK`。

---

### **完整示例配置**
```ini
auth_param basic program /usr/lib64/squid/basic_ncsa_auth /etc/squid/passwords
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
```

---

### **总结**
| 问题原因 | 解决方法 |
|----------|----------|
| 路径错误 | 使用 `find` 查找正确路径 |
| 未安装组件 | 安装 `squid-common` 或重装 Squid |
| 权限问题 | 检查 `/etc/squid/passwords` 权限 |

按步骤操作后，认证功能应恢复正常。
