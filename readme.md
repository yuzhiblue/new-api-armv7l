# New API 在 ARMv7l 平台部署指南

## 1. 简介

本文档介绍如何在 ARMv7l 架构的设备（如海纳思机顶盒 hi3798mv100）上部署 New API。New API 是一个统一的 AI API 网关和管理后台。

**适用设备：**
- 海纳思机顶盒（hi3798mv100/hi3798mv200 等）
- 树莓派 1/2/Zero（ARMv6/ARMv7）
- 其他 ARMv7l 架构的嵌入式设备

---

## 2. 环境要求

### 硬件要求
- CPU：ARMv7l 架构（32位）
- 内存：≥ 512MB（推荐 1GB 以上）
- 存储：≥ 100MB 可用空间

### 软件要求
- 操作系统：Linux（Ubuntu/Debian/CentOS 等）
- 依赖：SQLite3（可选，用于管理配置）

---

## 3. 获取二进制文件

### 方式一：直接下载预编译版本

如果你不想自己编译，可以直接下载适配 ARMv7l 的预编译二进制文件：

```bash
# 下载地址（示例）
wget https://example.com/new-api-armv7l -O new-api
chmod +x new-api
```

> **注意：** 请根据实际情况替换为真实的下载地址。

### 方式二：从源码交叉编译

如果你的设备是 ARMv7l 架构，但想自己编译，可以按照以下步骤在 macOS/Linux 主机上交叉编译：

#### 步骤 1：安装 Go 1.25+

```bash
# 下载并安装 Go 1.25.1（或更高版本）
wget https://go.dev/dl/go1.25.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.1.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

#### 步骤 2：获取源码

```bash
git clone https://github.com/QuantumNous/new-api.git
cd new-api
```

#### 步骤 3：修改默认主题（可选）

如果你的设备不支持 classic 主题的前端资源，可以修改默认主题为 `default`：

编辑 `setting/system_setting/theme.go`，将：

```go
var themeSettings = ThemeSettings{
    Frontend: "classic",
}
```

改为：

```go
var themeSettings = ThemeSettings{
    Frontend: "default",
}
```

#### 步骤 4：交叉编译

```bash
# 设置交叉编译参数
export CGO_ENABLED=0
export GOOS=linux
export GOARCH=arm
export GOARM=7

# 使用国内代理加速下载
export GOPROXY=https://goproxy.cn,direct

# 编译
go mod tidy
go build -a -ldflags='-s -w' -o new-api-armv7l .
```

编译完成后，会生成 `new-api-armv7l` 二进制文件。

---

## 4. 部署步骤

### 步骤 1：上传二进制文件到设备

使用 `scp` 或其他工具将编译好的二进制文件上传到目标设备：

```bash
scp new-api-armv7l root@192.168.8.201:/root/new-api
```

### 步骤 2：登录设备并配置

```bash
# 登录设备
ssh root@192.168.8.201

# 解压（如果文件是压缩的）
gunzip new-api-armv7l.gz

# 添加执行权限
chmod +x new-api
```

### 步骤 3：运行 New API

```bash
# 直接运行（前台）
./new-api --port 3000

# 后台运行
nohup ./new-api --port 3000 > /tmp/new-api.log 2>&1 &
```

### 步骤 4：检查运行状态

```bash
# 查看进程
ps | grep new-api

# 查看日志
tail -f /tmp/new-api.log

# 测试访问
curl http://127.0.0.1:3000/
```

---

## 5. 配置说明

### 环境变量

New API 支持通过环境变量配置：

```bash
# 监听端口（默认 3000）
export PORT=3000

# 数据库路径（默认 ./one-api.db）
export SQL_DSN=./one-api.db

# 是否调试模式
export DEBUG=true
```

### 配置文件

New API 使用 SQLite 数据库存储配置。首次运行会自动创建 `one-api.db` 文件。

如果需要修改主题等配置，可以通过 API 或直接在数据库中修改：

```bash
# 安装 sqlite3
apt-get install -y sqlite3

# 修改主题
sqlite3 one-api.db "UPDATE options SET value='default' WHERE key='theme';"
```

---

## 6. 运行和测试

### 访问 Web 界面

打开浏览器，访问：

```
http://设备IP:3000/
```

例如：`http://192.168.8.201:3000/`

### 检查 API 状态

```bash
curl http://127.0.0.1:3000/api/status
```

正常输出示例：

```json
{
  "success": true,
  "data": {
    "theme": "default",
    "version": "v0.0.0",
    "setup": false
  }
}
```

---

## 7. 常见问题

### Q1：编译时出现 `go 1.25.1` 版本错误？

**A：** 这是 new-api 的 `go.mod` 文件声明了不正确的 Go 版本。解决方法：

1. 安装真实的 Go 1.25+ 版本

### Q2：运行时显示 `Classic theme not available`？

**A：** 这是因为编译时没有包含 classic 主题的前端资源。解决方法：

1. 修改 `setting/system_setting/theme.go`，将默认主题改为 `default`
2. 重新编译并部署

### Q3：设备内存不足，无法运行？

**A：** New API 对内存要求不高，但如果你设备内存较小（< 512MB），可以尝试：

1. 关闭其他不必要的服务
2. 增加交换分区（swap）
3. 使用更轻量的主题（default 主题比 classic 更轻量）

### Q4：如何设置开机自启动？

**A：** 可以创建一个 systemd 服务（如果设备支持）：

```bash
# 创建服务文件
cat > /etc/systemd/system/new-api.service <<EOF
[Unit]
Description=New API Service
After=network.target

[Service]
Type=simple
ExecStart=/root/new-api --port 3000
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 启用并启动服务
systemctl enable new-api
systemctl start new-api
```

如果设备不支持 systemd，可以将启动命令添加到 `/etc/rc.local`：

```bash
echo "nohup /root/new-api --port 3000 > /tmp/new-api.log 2>&1 &" >> /etc/rc.local
chmod +x /etc/rc.local
```

---

## 8. 性能优化

### 启用 Gzip 压缩

在 Nginx 或 Caddy 等反向代理中启用 Gzip 压缩，可以减少传输数据量。

### 使用 SQLite 内存模式

如果你的设备内存充足，可以将数据库设置为内存模式（重启后数据会丢失）：

```bash
export SQL_DSN=:memory:
```

### 限制日志大小

定期清理日志文件，避免占用过多存储空间：

```bash
# 清空日志
echo "" > /tmp/new-api.log

# 或者使用 logrotate 自动轮转日志
```

---

## 9. 安全建议

1. **不要暴露到公网**：New API 默认没有身份验证，建议只在局域网内使用
2. **设置防火墙规则**：限制访问端口，只允许信任的 IP 访问
3. **定期备份数据库**：`one-api.db` 文件包含了所有配置和密钥，建议定期备份

---

## 10. 参考资源

- New API 官方文档：https://docs.newapi.pro
- New API GitHub 仓库：https://github.com/QuantumNous/new-api
- 海纳思官网：https://www.ecoo.top

---

**文档版本：** v1.0  
**最后更新：** 2026-05-19  
**适用 New API 版本：** v0.0.0+
