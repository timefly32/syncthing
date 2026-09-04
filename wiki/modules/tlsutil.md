# TLS 工具模块

## 概述

`lib/tlsutil/` 提供 TLS 证书和配置工具，约 300 行 Go 代码。

## 职责

- **证书生成**：自签名证书生成
- **TLS 配置**：Syncthing 特定的 TLS 配置
- **证书加载**：从磁盘加载证书

## 关键文件

| 文件 | 行数 | 职责 |
| --- | ---: | --- |
| `tlsutil.go` | ~200 | 证书生成和加载 |
| `secure.go` | ~100 | TLS 配置 |

## 证书生成

```go
func NewCertificate(id protocol.DeviceID, name string) (tls.Certificate, error)
```

1. 生成 3072 位 RSA 密钥（或 ECDSA）
2. 创建自签名证书
   - CommonName = 设备 ID 字符串
   - SAN = 设备 ID 字符串
   - 有效期 10 年
3. 返回 `tls.Certificate`

## 证书存储

- 路径：`~/.local/state/syncthing/cert.pem` 和 `key.pem`
- 首次启动自动生成
- 后续启动加载

## TLS 配置

```go
func SecureDefault() *tls.Config
```

Syncthing 的 TLS 配置：

- `InsecureSkipVerify=true`：不验证证书链（使用设备 ID 自验证）
- `MinVersion=TLS 1.2`
- `CipherSuites`：优先使用强加密套件
- `NextProtos`：`["bep/1.0"]`（ALPN 协议协商）

## 设备 ID 自验证

Syncthing 不使用 PKI/CA，而是：

1. TLS 配置 `InsecureSkipVerify=true`
2. 从对端证书计算设备 ID
3. 与已知设备 ID 比对
4. 仅信任用户明确添加的设备

## 证书字段

| 字段 | 值 |
| --- | --- |
| `CommonName` | 设备 ID 字符串 |
| `Subject Alternate Names` | 设备 ID 字符串 |
| `NotBefore` | 创建时间 |
| `NotAfter` | 创建时间 + 10 年 |

## 测试覆盖

`tlsutil_test.go`：

- 证书生成测试
- 证书加载测试
- TLS 配置测试

## 设计权衡

### 自签名 vs CA 签名

**选择**：自签名 + 设备 ID 自验证。
**优势**：无需 CA，去中心化。
**劣势**：需要带外交换设备 ID。

### RSA vs ECDSA

**选择**：默认 RSA 3072。
**优势**：广泛兼容。
**劣势**：密钥较大。

### TLS 1.2 vs 1.3

**选择**：最低 TLS 1.2。
**优势**：兼容性。
**劣势**：不强制使用 TLS 1.3。
