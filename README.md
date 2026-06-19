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

## 更新指南

运行中的服务使用 `ghcr.io/s2vzbh5s2t-crypto/xboard:latest` 镜像，更新时拉取新镜像并重建容器：

```bash
docker compose pull && docker compose up -d
```

容器启动时 entrypoint 会自动执行 `php artisan xboard:update`（数据库迁移、缓存刷新等），无需手动操作。站点配置与数据均在容器之外（MySQL、`./.env`、`./plugins`、`./storage`、`redis-data` 卷），重建容器不会丢失。

## 提交规范

本仓库只使用 `rebase` 同步上游，自用改动固定为上游之上的 3 个分类提交。这一节讲的是每次改动如何落进历史，与前面的部署运维无关。

需要完整的 git 历史：`--depth 1` 的浅克隆只适合部署机器，维护机器上若缺历史，先执行 `git fetch --unshallow`。

### 三个分类

| 类别 | 前缀 | 涉及路径 |
|---|---|---|
| 代码改动 | `fix:` / `feat:` | `app/**`、`routes/**`、`database/**` |
| 构建环境 | `build:` | `Dockerfile`、`compose.*.yaml`、`.github/**`、`update.sh`、`init.sh` |
| 文本/文档 | `docs:` | `README.md`、`AGENTS.md`、`docs/**`、`*.md`、`.gitignore` |

每类在历史上只占一个提交，这个提交是同类的**容器**：新增同类改动时不另开新提交，而是用 fixup 折进已有的那个。因此分类提交的内容会被反复改写，标题始终保留类别前缀。

### 固定历史结构

3 个分类提交固定在历史最顶部，它们之下的 `HEAD~3` 即上游最新提交：

| 位置 | 类别 |
|---|---|
| `HEAD` | 代码改动 |
| `HEAD~1` | 构建环境 |
| `HEAD~2` | 文本/文档 |
| `HEAD~3` | 上游基线 |

定位只看位置，不看哈希——每次归并或同步后哈希都会变。

### 折入改动

先一次性把锚点记下来，**这一步必须在使用任何相对引用之前完成**：

```bash
BASE=$(git rev-parse HEAD~3)    # 上游基线，也是归并时的 rebase 起点
CODE=$(git rev-parse HEAD)
BUILD=$(git rev-parse HEAD~1)
DOCS=$(git rev-parse HEAD~2)
```

原因：只要创建了第一个 fixup 提交，`HEAD` 及其相对位置就整体上移一位，此时再写 `HEAD~1`、`HEAD~2` 会折到错误的类别上。预先取好的哈希则与执行顺序无关。

按改动性质分别折入，跨类别的改动就分多次提交：

```bash
git add app/Protocols/ClashMeta.php
git commit --fixup=$CODE

git add README.md
git commit --fixup=$DOCS
```

生成的标题形如 `fixup! fix: remove forced direct subscription rule`，只是给 autosquash 的标记，不会留在最终历史里。

### 归并

```bash
OLD=$(git rev-parse HEAD)                                  # 归并前状态，仅用于校验
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash $BASE
git log --oneline -4                                       # 顶部应回到 3 个分类提交
git diff $OLD HEAD                                         # 应为空：只重写历史，内容未变
```

`--autosquash` 把每个 fixup 提交自动并回对应分类提交。`GIT_SEQUENCE_EDITOR=true` 让 rebase 不打开编辑器，直接确认 git 已排好的整理清单，所以整条命令可以无人值守执行。

### 同步上游

首次配置上游远端：

```bash
git remote add upstream https://github.com/cedar2025/Xboard.git
git remote -v
```

日常同步前确认工作区干净，有本地修改就先按上面的方式 fixup 掉：

```bash
git status
git fetch upstream
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash upstream/master
```

这里的 `--autosquash` 同时做两件事：把尚未归并的 fixup 收进分类提交，再把 3 个分类提交重放到新的上游之上。没有配置 upstream 远端时，用前面记下的 `$BASE` 作起点同样可行。

出现冲突时逐个解决：

```bash
git status
git add <冲突文件>
git rebase --continue
```

放弃本次同步：

```bash
git rebase --abort
```

### 推送

```bash
git push --force-with-lease origin master
```

折入和同步都会重写历史，只能强推。不要用裸 `--force`：`--force-with-lease` 会在远端存在本地未见过的提交时拒绝推送，避免覆盖远端的新提交。

## 维护原则

- 优先保留上游结构，只在必要位置做自用修改。
- 修改前先确认 `git status`，避免混入无关文件。
- 自用改动严格归入三类，保持每类一个提交，便于 rebase 时定位冲突。
- 定期同步上游，避免一次性跨越太多提交导致冲突扩大。
