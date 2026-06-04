# cc-grow

## 🔧 Windows 安装 tmux

```powershell
scoop install psmux
```

`psmux` 是 tmux 的 Windows 移植版，安装后通过 `tmux` 命令直接使用：

```bash
tmux new -s myproject   # 创建新会话
tmux attach             # 附着到已存在的会话
tmux ls                 # 列出所有会话
```

> **备选：WSL 安装**（纯 Linux 版，需 WSL 环境）
> ```bash
> sudo apt update && sudo apt install tmux
> ```

## 项目环境

- **平台**: Windows
- **Shell**: PowerShell 7+
- **默认分支**: `main`
