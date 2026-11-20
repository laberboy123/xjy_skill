# 上位机文本命令协议

让上位机同时被人和脚本驱动。目标：一个通道，两种用法——人点 GUI，脚本发 JSON。

## 设计原则

1. **一条通道，两种前端**。GUI 和脚本用同一套命令，不做两套。
2. **可发现**。`help` 返回命令表，`id` 返回固件版本与能力位。脚本不必硬编码版本差异。
3. **幂等优先**。重复发同一条命令不应产生不同副作用（`read` 幂等，`reboot` 明确标记）。
4. **有超时**。每条命令必须有最大响应时间，超时由上位机判失败，不能让测试挂死。
5. **错误可区分**。不要只返回 `false`，要返回错误码，脚本才能判"是设备坏了还是参数错了"。

## 传输与分帧

- **UART**：115200 8N1（或更高）。一行一条 JSON，以 `\n` 结束，UTF-8/ASCII。
- **TCP**：同一套，一条一行。
- 单行最大 4 KB，超出直接断开重连（防缓冲区被打爆）。

## 命令表

| 命令 | 参数 | 返回 | 说明 |
|---|---|---|---|
| `ping` | — | `{"ok":true}` | 存活检测，超时 200ms |
| `id` | — | `{"ok":true,"fw":"1.2.3","hw":"revB","sn":"...","caps":["i2c","spi","inject"]}` | 版本与能力 |
| `help` | — | `{"ok":true,"cmds":[...]}` | 命令表 |
| `read` | `addr`,`reg`,`len?` | `{"ok":true,"data":[117]}` | 读寄存器，data 为十进制字节数组 |
| `write` | `addr`,`reg`,`data` | `{"ok":true}` | 写寄存器 |
| `dump` | `addr`,`from?`,`to?` | `{"ok":true,"map":{"0x00":11,...}}` | 区间快照 |
| `stream` | `ch`,`ms`,`n?` | `{"ok":true,"n":100}` 后跟 n 条 `{"ch":"accel","t":1234,"v":[1,2,3]}` | 连续采样 |
| `stop` | `ch?` | `{"ok":true}` | 停止 stream |
| `selftest` | — | `{"ok":true,"items":[{"dev":"sht40","id":"0x68","pass":true,"err":0}]}` | 全器件自检 |
| `inject` | `target`,`mode`(`nack`/`timeout`/`badid`/`drop`) | `{"ok":true}` | 故障注入，仅测试固件可用 |
| `reboot` | — | `{"ok":true}` | 重启，**非幂等** |
| `log` | `level` | `{"ok":true}` | 设置日志级别 |

## 响应格式

成功：

```json
{"ok":true, ...}
```

失败：

```json
{"ok":false,"err":"ERR_TIMEOUT","msg":"i2c read timeout addr=0x68"}
```

错误码：

| 码 | 含义 | 脚本该怎么处理 |
|---|---|---|
| `ERR_PARAM` | 参数非法 | 测试脚本 bug，直接失败 |
| `ERR_BUSY` | 设备忙 | 退避重试 |
| `ERR_TIMEOUT` | 总线/器件超时 | 重试 2 次，仍失败判设备异常 |
| `ERR_NACK` | 从机无应答 | 判接线/器件问题 |
| `ERR_CRC` | 帧校验错 | 重发 |
| `ERR_NOTREADY` | 未初始化完成 | 等待后重试 |
| `ERR_UNSUPPORTED` | 该固件不支持（如 inject） | 跳过该用例并标注 |

## 示例

```
→ {"cmd":"ping"}
← {"ok":true}

→ {"cmd":"id"}
← {"ok":true,"fw":"1.2.3","caps":["i2c","spi","self-test"]}

→ {"cmd":"read","addr":"0x68","reg":"0x75"}
← {"ok":true,"data":[104]}

→ {"cmd":"inject","target":"i2c","mode":"nack"}
← {"ok":false,"err":"ERR_UNSUPPORTED","msg":"release build"}
```

## 固件侧最小实现要点（C）

```c
/* 1. 行缓冲 + 超时，避免半包卡死 */
/* 2. 命令表用静态数组，查不到直接 ERR_PARAM */
/* 3. 每条命令执行前检查就绪态，未就绪返回 ERR_NOTREADY */
/* 4. 输出用统一序列化函数，别到处 sprintf 拼 JSON（转义会出错） */
/* 5. inject 只在 DEBUG 构建里编译进命令表 */
```

## 上位机侧脚本示例（Python）

```python
import json, serial, time

class Dev:
    def __init__(self, port, baud=115200, timeout=2.0):
        self.s = serial.Serial(port, baud, timeout=timeout)

    def cmd(self, **kw):
        self.s.write((json.dumps(kw) + "\n").encode())
        line = self.s.readline()
        if not line:
            raise TimeoutError(f"no response: {kw}")
        r = json.loads(line)
        if not r.get("ok"):
            raise RuntimeError(f"{r.get('err')}: {r.get('msg')}")
        return r

    def ping(self):  return self.cmd(cmd="ping")["ok"]
    def read(self, addr, reg, n=1): return self.cmd(cmd="read", addr=hex(addr), reg=hex(reg), len=n)["data"]

def test_sensor_id():
    d = Dev("COM3")
    assert d.ping()
    data = d.read(0x68, 0x75)
    assert data == [0x68], f"unexpected ID: {data}"
```

## 反模式

| 反模式 | 后果 |
|---|---|
| 只有 GUI，命令藏在按钮回调里 | 无法脚本化、无法复现、无法进 CI |
| 返回 `ok/fail` 不带错误码 | 脚本无法区分"设备坏"和"参数错" |
| 明文多行输出、没有分帧 | 粘包/半包，脚本解析随机失败 |
| 命令无超时 | 测试挂死，比失败更糟 |
| 测试命令和生产命令混在一起且无版本标记 | 误触发（如 `inject` 进了量产固件） |
| 用中文/多行文本做机器接口 | 编码与解析成本高，用 JSON + 错误码 |
