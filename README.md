# Xboard Fork Optimized

基于 [cedar2025/Xboard](https://github.com/cedar2025/Xboard) 的自用 fork，只保留部署、构建和少量必要定制，尽量降低同步上游的维护成本。

## aaPanel + Docker 部署

### 1. 安装 aaPanel

```bash
curl -sSL https://www.aapanel.com/script/install_6.0_en.sh -o install_6.0_en.sh && \
bash install_6.0_en.sh aapanel
```

### 2. 安装运行环境

在 aaPanel 中安装 Nginx 和 MySQL 5.7。PHP 和 Redis 不是必须项，项目运行依赖由 Docker 容器提供。

### 3. 创建站点

在 aaPanel 中创建站点时，PHP 版本选择“纯静态”。

### 4. 克隆项目并初始化

进入 aaPanel 站点目录，清空默认文件后克隆本仓库：

```bash
cd /www/wwwroot/你的站点目录
rm -rf * .[!.]*
git clone -b master --depth 1 https://github.com/s2vzbh5s2t-crypto/Xboard.git ./
cp compose.host.sample.yaml compose.yaml
docker compose run -it --rm xboard php artisan xboard:install
docker compose up -d
```

### 5. 配置反向代理

在 aaPanel 站点反向代理中，将请求转发到 `127.0.0.1:7001`。

基础 Nginx 反向代理配置示例：

```nginx
location ^~ / {
    proxy_pass http://127.0.0.1:7001;
    proxy_http_version 1.1;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $http_connection;
    proxy_read_timeout 60s;
    proxy_buffering off;
    proxy_cache off;
}
```

## 常用命令

```bash
docker compose ps
docker compose logs -f xboard
docker compose restart
docker compose pull
docker compose up -d
```

## Fork 维护流程

本仓库只使用 `rebase` 同步上游，自用修改固定为 `upstream/master` 之上的 3 个分类提交。

首次配置上游仓库：

```bash
git remote add upstream https://github.com/cedar2025/Xboard.git
git remote -v
```

### 提交分类

自用改动按三类组织，每类在 `upstream/master` 之上保持恰好一个提交：

| 类别 | 前缀 | 涉及路径 |
|---|---|---|
| 代码改动 | `fix:` / `feat:` | `app/**`、`routes/**`、`database/**` |
| 文本/文档 | `docs:` | `README.md`、`AGENTS.md`、`docs/**`、`*.md`、`.gitignore` |
| 构建环境 | `build:` | `Dockerfile`、`compose.*.yaml`、`.github/**`、`update.sh`、`init.sh` |

3 个分类提交固定位于历史最顶部，按位置定位：

| 位置 | 类别 |
|---|---|
| `HEAD` | 代码改动 |
| `HEAD~1` | 构建环境 |
| `HEAD~2` | 文本/文档 |

新增自用改动时，用 `--fixup` 折入对应分类提交，避免产生额外提交：

```bash
git add <修改文件>
git commit --fixup=HEAD      # 代码改动
git commit --fixup=HEAD~1    # 构建环境
git commit --fixup=HEAD~2    # 文本/文档
```

### 同步上游

日常同步前先确保工作区干净；如有本地修改，先按上述分类 fixup 提交或临时 `stash`。

```bash
git status
git fetch upstream
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash upstream/master
```

`--autosquash` 会把 fixup 提交自动合并回对应分类提交，保持始终只有 3 个 fork 提交。

如有冲突，修复后继续：

```bash
git status
git add <冲突文件>
git rebase --continue
```

如需放弃本次同步：

```bash
git rebase --abort
```

同步完成后推送到当前 fork：

```bash
git push --force-with-lease origin master
```

不要使用裸 `--force`，避免覆盖远端已有的新提交。

## 维护原则

- 优先保留上游结构，只在必要位置做自用修改。
- 修改前先确认 `git status`，避免混入无关文件。
- 自用改动严格归入三类，保持每类一个提交，便于 rebase 时定位冲突。
- 定期同步上游，避免一次性跨越太多提交导致冲突扩大。
