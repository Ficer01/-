# LinkMed Mac 本地运行交接说明

这份文档用于在新的 Mac 设备上交给 Codex 使用，让它帮助你把 LinkMed 项目本地跑起来，并接上近期做过的文件交付相关后端改动。

## 给 Mac 端 Codex 的第一句话

可以把下面这段直接发给 Mac 上的 Codex：

```text
我需要在这台 Mac 上把 LinkMed 项目本地跑起来。请先不要乱改代码，先检查环境、仓库、Docker、MySQL、前后端启动配置。目标是能本地启动前端和后端，并能测试 Codex 文件交付链路。

远端仓库是 git@115.190.44.90:agent/LinkMed.git。

需要重点关注后端 artifact 文件交付相关分支：
codex/artifact-pdf-delivery-handoff-20260731

这个分支包含之前 artifact worker、文件卡片交付、token/统计 worker 整合，以及最近补的 PDF fallback 渲染增强。请先只检查和启动，不要随意重构；如果需要修改，先说明原因和影响。
```

## 当前目标

在 Mac 上完成这些事情：

1. 拉取 LinkMed 远端代码。
2. 能启动 MySQL。
3. 能启动后端服务。
4. 能启动前端页面。
5. 能创建 Codex 会话容器。
6. 能测试生成 Word、PDF、Excel、PPT 等文件并看到下载卡片。
7. 如果后端正式分支还没合入 artifact 改动，可从交接分支 cherry-pick 或 merge。

## 推荐分支策略

先不要直接在正式分支上改。

建议 Mac 端 Codex 这样处理：

```bash
git clone git@115.190.44.90:agent/LinkMed.git
cd LinkMed
git fetch origin
git checkout develop
git pull
git checkout -b codex/mac-local-run-setup
```

如果要查看文件交付增强分支：

```bash
git fetch origin codex/artifact-pdf-delivery-handoff-20260731
git checkout codex/artifact-pdf-delivery-handoff-20260731
```

如果只想把文件交付相关改动带到你自己的测试分支：

```bash
git checkout codex/mac-local-run-setup
git cherry-pick a5f688862
git cherry-pick bdc467272
```

说明：

- `a5f688862` 是 artifact worker、文件交付、token/统计 worker 等主改动。
- `bdc467272` 是 PDF fallback 渲染增强。
- 如果目标分支已经包含 `a5f688862`，只需要考虑 `bdc467272`。

## Mac 基础环境检查

让 Mac 端 Codex 先跑这些检查：

```bash
git --version
java -version
mvn -version
node -v
npm -v
docker --version
docker compose version
```

建议版本：

- Java：21
- Maven：3.9.x
- Node：按前端项目要求，通常 18 或 20
- Docker Desktop：已启动

如果缺 Java 21，可以用 Homebrew：

```bash
brew install openjdk@21
```

如果缺 Maven：

```bash
brew install maven
```

如果缺 Node，建议用 nvm 或 Homebrew 安装。

## Docker/MySQL

本地一般需要 MySQL 8。

可以新建一个本地测试 MySQL 容器，避免污染其他设备已有数据库：

```bash
docker run -d \
  --name linkmed-mysql \
  -e MYSQL_ALLOW_EMPTY_PASSWORD=yes \
  -p 3306:3306 \
  mysql:8.0
```

如果容器已经存在：

```bash
docker start linkmed-mysql
```

创建本地测试数据库，建议明确字符集和排序规则，避免 Flyway 或中文字段出问题：

```bash
docker exec -it linkmed-mysql mysql -uroot -e "CREATE DATABASE IF NOT EXISTS link_med_artifact_test CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

检查：

```bash
docker exec -it linkmed-mysql mysql -uroot -e "SHOW DATABASES;"
```

## Codex 运行镜像

之前本地使用过的 Codex 运行镜像名称需要固定为：

```text
codex-src-python-venv:latest
```

Mac 上需要确认：

```bash
docker images | grep codex-src-python-venv
```

如果没有这个镜像，有两种方式：

1. 从别人导出的 tar 包导入：

```bash
docker load -i codex-src-python-venv.tar
```

2. 如果仓库里有 Dockerfile，再按项目实际 Dockerfile 构建：

```bash
docker build -t codex-src-python-venv:latest .
```

注意：先不要随便重构镜像。尤其是 Playwright、Chromium、Python 统计依赖、PDF/Word/PPT 生成依赖都可能影响体积和可用性。

## 后端启动

进入后端目录后启动。

示例：

```bash
mvn spring-boot:run \
  -Dspring-boot.run.arguments="--server.port=8181 --management.server.port=8481 --app.server.base-url=http://localhost:8181 --codex.codexHomeRoot=data-artifact/codex-tenants"
```

如果数据库名使用 `link_med_artifact_test`，需要确认配置文件或启动参数里 datasource 指向它。

如果启动失败，先看：

- MySQL 容器是否运行。
- 数据库是否存在。
- Flyway 是否因为历史迁移顺序失败。
- 端口是否被占用。
- Docker Desktop 是否已经启动。

常用检查：

```bash
lsof -i :8181
lsof -i :8481
lsof -i :3306
docker ps
```

## 前端启动

进入前端目录后：

```bash
npm install
npm run dev
```

前端一般通过 Vite 代理访问后端。如果前端报：

```text
http proxy error
ECONNREFUSED
```

优先检查后端是否真的在对应端口启动成功。

## 文件交付链路重点

近期后端改动的核心是减少模型对文件交付结果的随机影响。

目前重点逻辑：

1. 模型/skill 仍可能生成内容或源文件。
2. 后端监控 output 目录。
3. 生成到 output 的目标文件会进入快照和文件卡片链路。
4. 如果模型没有稳定生成目标文件，artifact worker 会尝试从本轮已有内容源生成目标文件。
5. PDF fallback 已增强：后端会用固定 PDFBox 流程解析 Markdown/文本/Docx 源，生成可打开、有基础排版的 PDF。
6. 本地运行产物 `backend/data-artifact/` 已加入忽略规则，不应提交。

## 文件交付测试建议

启动后可以逐个测试：

```text
生成一个 Word 报告，主题是腹膜透析患者心血管事件预测模型。
```

```text
把上面的内容生成 PDF。
```

```text
生成一个 Excel 表格，整理 2026 年临床药学领域最新文献。
```

```text
生成一个 PPT，主题是 SGLT2 抑制剂在心衰患者中的应用。
```

观察重点：

- 最终是否出现文件卡片。
- Docker 容器内是否真的有目标文件。
- 后端日志是否出现 output monitor 检测到文件。
- `turn/completed` 里的 `items: []` 不一定代表没有卡片，主要看后端 generated-files 和前端卡片。

## 后端测试命令

文件交付相关测试：

```bash
mvn "-Dtest=PdfArtifactWorkerTest,DocxArtifactWorkerTest,SpreadsheetArtifactWorkerTest,ExistingArtifactWorkerTest,ArtifactCandidateSelectorTest,CodexOutputMonitorTest" test
```

当前已在 Windows 本地验证过：

```text
Tests run: 19, Failures: 0, Errors: 0, Skipped: 0
```

Mac 上合入或启动前建议重新跑一遍。

## 给 Mac 端 Codex 的注意事项

请让 Mac 端 Codex 遵守：

1. 先检查，不要一上来改代码。
2. 不要删除已有数据库或 Docker volume。
3. 不要随便重构镜像。
4. 不要把运行目录、生成文件、数据库数据提交到 Git。
5. 如果需要改代码，先说明改动点和回退方式。
6. 文件交付问题优先查后端 output、snapshot、generated-files、artifact worker。
7. PDF 问题优先查 `PdfArtifactRenderer` 和 `PdfArtifactWorker`。

## 交接分支

远端分支：

```text
codex/artifact-pdf-delivery-handoff-20260731
```

关键提交：

```text
a5f688862 Improve codex artifact delivery workers
bdc467272 Improve PDF artifact fallback rendering
```

同事可以选择：

- merge 整个分支。
- cherry-pick 两个提交。
- 如果已有前一个提交，只 cherry-pick 后一个 PDF 提交。

