# PC端commit 命令模板：
```bash
# 1. 首先拉取最新更改（避免冲突）
git pull origin main --rebase

# 2. 检查状态，查看修改的文件
git status

# 3. 添加所有更改到暂存区
git add .

# 4. 提交更改
git commit -m "PC: [简要说明修改内容] - $(date '+%Y-%m-%d %H:%M')"

```

# 如果PC端IDEA 出现 因为开代理 ，代理正常，但链接不上github的情况，执行以下命令：
```bash
# 如果你在浏览器中使用 VPN 或代理工具访问 GitHub，
# 也需要让 Git 使用相同的代理。打开终端（PowerShell 或 CMD）
# 并运行以下命令（将 7890 替换为你的实际代理端口，Clash 通常为 7890，
# 其他代理通常为 1080）
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890
```

