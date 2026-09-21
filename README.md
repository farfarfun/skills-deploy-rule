# skills-deploy-rule

用于编写管理本地后台服务的 Shell 脚本规范，覆盖 `gum` 交互、进程与端口检查、PID 文件、日志和 `start`/`stop`/`restart` 等命令。

## 安装

将 Skill 复制到 Codex 的 skills 目录：

```bash
git clone https://github.com/farfarfun/skills-deploy-rule.git
mkdir -p ~/.codex/skills
cp -R skills-deploy-rule ~/.codex/skills/
```

## 最小示例

下面的命令演示如何把一个本地 HTTP 服务放到后台，并保存 PID 和日志：

```bash
mkdir -p ~/.server/demo/logs
nohup python -m http.server 8000 >~/.server/demo/logs/stdout.log 2>&1 &
echo $! > ~/.server/demo/pid
curl http://127.0.0.1:8000/
kill "$(cat ~/.server/demo/pid)"
```

具体服务脚本应将 PID 和日志保存到 `~/.server/<服务名>/`，并在停止前同时确认 PID 与监听端口。

---

## 关于 farfarfun

[farfarfun](https://github.com/farfarfun) 是一个专注于实用工具库的开源组织，
涵盖云存储、数据处理、AI、多媒体与开发工具链等方向。

- 🏠 组织主页：<https://github.com/farfarfun>
- 📧 联系：farfarfun@qq.com

本项目基于 [MIT](LICENSE) 协议开源。
