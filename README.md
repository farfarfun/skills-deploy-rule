# skills-deploy-rule

用于编写管理本地后台服务的 Shell 脚本规范，覆盖 `scripts/setup.sh` 统一入口、`dev`/`prod` 环境、进程与端口检查、PID 文件、日志和服务生命周期命令。

## 安装

将 Skill 复制到 Codex 的 skills 目录：

```bash
git clone https://github.com/farfarfun/skills-deploy-rule.git
mkdir -p ~/.codex/skills
cp -R skills-deploy-rule/skills/* ~/.codex/skills/
```

## 安装验证

安装后确认 Codex 可以发现该 Skill：

```bash
test -f ~/.codex/skills/authoring-background-service-scripts/SKILL.md
```

本仓库只发布 Skill 文档，不包含 `scripts/setup.sh` 或示例服务。将 Skill 用于目标项目后，按其要求实现并验证 `scripts/setup.sh <action> <env>`；例如 `bash -n scripts/setup.sh`、`scripts/setup.sh run dev`、`scripts/setup.sh start dev`、`scripts/setup.sh status` 和 `scripts/setup.sh stop dev`。`run dev` 在前台运行开发服务（验证后用 `Ctrl-C` 退出），其余命令验证后台生命周期。

---

## 关于 farfarfun

[farfarfun](https://github.com/farfarfun) 是一个专注于实用工具库的开源组织，
涵盖云存储、数据处理、AI、多媒体与开发工具链等方向。

- 🏠 组织主页：<https://github.com/farfarfun>
- 📧 联系：farfarfun@qq.com

本项目基于 [MIT](LICENSE) 协议开源。
