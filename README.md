# skills-deploy-rule

用于编写管理本地后台服务的 Shell 脚本规范，覆盖 `scripts/setup.sh` 统一入口、`dev`/`prod` 环境、进程与端口检查、PID 文件、日志和服务生命周期命令。

## 安装

将 Skill 复制到 Codex 的 skills 目录：

```bash
git clone https://github.com/farfarfun/skills-deploy-rule.git
mkdir -p ~/.codex/skills
cp -R skills-deploy-rule ~/.codex/skills/
```

## 最小示例

符合规范的项目通过统一入口管理服务。以下命令先检查脚本语法，再分别验证前台开发模式和后台启停生命周期：

```bash
bash -n scripts/setup.sh
scripts/setup.sh run dev
scripts/setup.sh start dev
scripts/setup.sh status
scripts/setup.sh stop dev
```

`run dev` 在前台运行开发服务（验证后用 `Ctrl-C` 退出），其余命令验证后台生命周期。脚本应将 PID、日志和锁文件保存到仓库内的 `.run/`，拒绝重复启动，识别陈旧 PID，并在启动和停止后校验进程及端口；验证失败必须返回非零状态。`start prod` 只能运行已经安装的正式包，不能临时从源码构建或改用开发服务。

---

## 关于 farfarfun

[farfarfun](https://github.com/farfarfun) 是一个专注于实用工具库的开源组织，
涵盖云存储、数据处理、AI、多媒体与开发工具链等方向。

- 🏠 组织主页：<https://github.com/farfarfun>
- 📧 联系：farfarfun@qq.com

本项目基于 [MIT](LICENSE) 协议开源。
