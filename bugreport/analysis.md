# 分析过程：pygpsclient 连接后立刻断开（Not connected）

## 1. 现象

- 使用方式 A（内置模拟器）：`pygpsclient -U ubxsimulator`
- 点击「USB/UART」连接后，左下角短暂显示绿色已连接，很快变回 `Not connected`
- 状态栏可能出现 `inactivity timeout` 红字提示

## 2. 排查步骤

### 2.1 排除「用法错误」

- 确认 pygpsclient 对 `ubxsimulator` 端口名的支持是官方内置功能：
  `stream_handler.py` 中 `if HASUBXUTILS and settings["serial_settings"].port == UBXSIMULATOR: conntype = CONNECTED_SIMULATOR`
- 确认 `pyubxutils` 可正常导入、`UBXSimulator` 已导出
- 结论：用法无误，问题在连接建立后的数据流阶段

### 2.1.1 GitHub README 佐证（用法与官方文档一致）

**PyGPSClient README**（Instructions → Settings panel，第 3、4 条）：

> 3. To connect to a GNSS receiver via USB or UART port, select the device
>    from the listbox, set the appropriate serial connection parameters and
>    click [USB icon].
>
> 4. A custom user-defined serial port can also be passed ... as a command
>    line argument `--userport`. A special userport value of "ubxsimulator"
>    invokes the experimental `pyubxutils.UBXSimulator` utility to emulate a
>    GNSS NMEA/UBX serial stream.

即：`pygpsclient -U ubxsimulator` → 选中 → 点 USB 连接按钮，与本次使用方式完全一致。

**pyubxutils README**（ubxsimulator utility 章节）：

> Provides a simple simulation of a GNSS serial stream by generating
> synthetic UBX or NMEA messages based on parameters defined in a json
> configuration file.

即：模拟器本就应输出 NMEA/UBX 报文；本机配置 `~/ubxsimulator.json` 也符合其预期。

→ 结论：**用户用法与官方文档完全吻合，问题不在用法。**

### 2.2 排除「配置文件 / 数据生成」问题

- 命令行 `ubxsimulator --verbosity 3` 正常，能输出 NAV-PVT / NAV-DOP / NAV-SAT / GNGGA / GNRMC 等
- 进程内复现 pygpsclient 的读取路径（`GNSSReader` + `UBXSimulator()` 默认配置）正常，能持续读到消息
- 结论：数据生产与解析本身无问题

### 2.3 定位到「连接后立刻断开」的触发点

追踪 pygpsclient 的断开链路：

```
stream_handler._read_thread()
  └─ except TimeoutError:
       stopevent.set()
       master.event_generate(settings["timeout_event"])
            └─ app.on_gnss_timeout()
                 └─ conn_status = DISCONNECTED  →  "Not connected"
```

即：读取线程抛了 `TimeoutError`。

### 2.4 找到 `TimeoutError` 的抛出来源

`TimeoutError` 来自 `pyubxutils/ubxsimulator.py` 的 `read()`：

```python
def read(self, num: int = 1) -> bytes:
    while len(self._buffer) < num:
        sleep(self._interval / 20000)
        if datetime.now() > self._lastread + timedelta(seconds=self._timeout):
            raise TimeoutError
    ...
```

而 `_lastread` 在 `__init__` 里被初始化为：

```python
self._lastread = datetime.fromordinal(1)   # 公元 1 年
```

### 2.5 验证根因

写最小复现脚本，构造「缓冲区为空 + 首次 read」的场景：

```python
sim = UBXSimulator()   # 不调用 start()，缓冲区保持为空
sim.read(1)            # 立即抛 TimeoutError
```

结果：**立即**抛出 `TimeoutError`（而非等待 3 秒）。

原因是 `datetime.now() > 公元1年 + 3秒` 恒为真。

## 3. 触发条件（启动竞态）

模拟器 `start()` 的时序：

1. 启动消息生产线程（`_msgfactory`），约需时间产出第一批消息
2. 主线程 `sleep(interval/1000)`（1 秒）
3. 启动主循环线程（`_mainloop`），把生产队列搬到读取缓冲区
4. `start()` 返回，pygpsclient 的读取线程开始 `read()`

若第 4 步的首次 `read()` 发生在第 3 步缓冲区就绪**之前**（线程调度竞态），
就会命中上面的 bug，立即超时断开。

## 4. 结论

| 项 | 结论 |
|----|------|
| 责任方 | **pyubxutils 工具本身**（bug） |
| 用户用法 | 正确，非使用问题 |
| 触发条件 | 启动竞态，与用户操作无关 |
| 上游状态 | `main` 分支第 112 行仍为 `datetime.fromordinal(1)`，未修复 |

## 5. 修复

将初始化值改为当前时间：

```python
self._lastread = datetime.now()
```

使首次读取在缓冲区为空时等待完整的 `timeout`（默认 3 秒），而不是立即超时。

验证结果：

- 空缓冲区首次 `read()` 从「立即抛」变为「等待 3.0 秒」
- 模拟器功能回归正常
