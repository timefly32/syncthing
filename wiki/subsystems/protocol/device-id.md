# 设备 ID

## 概述

`lib/protocol/deviceid.go`（239 行）实现设备 ID 的计算、编码和验证。设备 ID 是 Syncthing 信任模型的基础。

## 设备 ID 计算

```go
func NewDeviceID(rawCert []byte) DeviceID {
    return DeviceID(sha256.Sum256(rawCert))  // 证书的 SHA-256
}
```

**不变量**：设备 ID = 设备 TLS 证书原始字节的 SHA-256，32 字节，确定性。

## 类型

```go
const (
    DeviceIDLength      = 32  // SHA-256
    ShortIDStringLength = 7
)

type DeviceID [32]byte
type ShortID  uint64

LocalDeviceID  = repeatedDeviceID(0xff)  // 本地设备
GlobalDeviceID = repeatedDeviceID(0xf8)  // 全局聚合
EmptyDeviceID  = DeviceID{}              // 全零
```

## 字符串编码

`String()` 流程（`deviceid.go:70-83`）：

1. base32 标准编码
2. 去除 `=` 填充
3. **`luhnify`**：每 13 字符组后插入 1 个 Luhn-32 校验位，52 字符 → 56 字符
4. **`chunkify`**：每 7 字符用 `-` 分隔

最终格式：`XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX-XXXXXXX`

### UnmarshalText

反向操作（`deviceid.go:123-153`）：

- 长度 56 → 新格式（带校验位），`unluhnify` 校验
- 长度 52 → 旧格式（无校验位）
- 其他长度 → 错误
- `untypeoify`：将易混淆数字 `0→O`、`1→I`、`8→B` 纠正

## Luhn-32 校验

`luhn.go`（50 行）使用 base32 字母表 `ABCDEFGHIJKLMNOPQRSTUVWXYZ234567`，交替因子 1/2，模 32 求校验位。

**注意**：并非标准 Luhn 算法，是 Syncthing 自定义变体。

## ShortID

`Short()` = 设备 ID 前 8 字节的大端 uint64。

`ShortID.String()` = base32 编码前 7 字符。

ShortID 用于版本向量中的 `Counter.ID`，减少传输开销。

## protobuf 集成

`ProtoSize`/`MarshalTo`/`Unmarshal`（`deviceid.go:155-176`）让 `DeviceID` 可直接作为 protobuf bytes 字段，固定 32 字节。

## 特殊设备 ID

| ID | 值 | 用途 |
| --- | --- | --- |
| `LocalDeviceID` | `0xff` 重复 32 次 | 本地设备（虚拟） |
| `GlobalDeviceID` | `0xf8` 重复 32 次 | 全局聚合（虚拟） |
| `EmptyDeviceID` | 全零 | 空/未设置 |

## 信任模型

Syncthing 不依赖 PKI/CA，而是通过设备 ID 自验证：

1. TLS 配置 `InsecureSkipVerify=true`（不验证证书链）
2. 从对端证书计算设备 ID
3. 与已知设备 ID 比对
4. 仅信任用户明确添加的设备

设备 ID 必须通过**带外渠道**交换（如手动输入、二维码）。

## 测试覆盖

`deviceid_test.go`（150 行）：

- `TestFormatDeviceID` / `TestValidateDeviceID` — 往返
- `TestMarshallingDeviceID` — proto 序列化
- `TestShortIDString` — ShortID
- `TestDeviceIDFromBytes` — 字节转换
- `BenchmarkLuhnify` / `BenchmarkUnluhnify` / `BenchmarkChunkify` / `BenchmarkUnchunkify` — 性能
