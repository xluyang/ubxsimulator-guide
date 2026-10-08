# Bug 归档：pygpsclient 连接后立刻断开（Not connected）

本目录保存「pygpsclient 内置模拟器连接后立刻断开」问题的完整资料。

## 文件清单

| 文件 | 内容 |
|------|------|
| `issue.md` | 可提交到上游 GitHub 的 issue 草稿（英文） |
| `analysis.md` | 完整排查与分析过程（中文） |
| `fix-ubxsimulator-timeout.patch` | 一行修复补丁（unified diff 格式） |

## 一句话结论

- **责任方**：pyubxutils 工具本身的 bug
- **用户用法**：正确（`pygpsclient -U ubxsimulator` 是官方支持用法）
- **根因**：`ubxsimulator.py` 中 `_lastread` 被初始化为 `datetime.fromordinal(1)`（公元 1 年），
  导致首次读取、缓冲区为空时立即抛 `TimeoutError`
- **上游状态**：`main` 分支仍未修复（截至 2026-10-08）

## 应用补丁

```bash
cd ~/.local/lib/python3.14/site-packages
patch -p1 < ~/ubxsimulator/bugreport/fix-ubxsimulator-timeout.patch
```

（本机当前已手动修复，无需重复应用；升级 pyubxutils 后需重新应用。）

## 相关文档

- 使用指南：`../README.md`
- 上游仓库：https://github.com/semuconsulting/pyubxutils
