# RAG 应用 — GitHub Actions + Railway 自动化部署

本项目是一个基于 FastAPI 的检索增强生成（RAG）应用，包含前端 Web UI（Firebase 登录），并通过 GitHub Actions + Railway 实现 CI/CD：构建 Docker、运行测试、自动部署。

**技术栈**
- 后端：`FastAPI`、`LangChain`、`FAISS`、`OpenAI`
- 前端：`HTML`、`CSS`、`JavaScript`、`Firebase Web SDK`、`Firestore`
- DevOps：`Docker`、`GitHub Actions`、`Railway`

**项目概览**
- 提供 `/ui` 前端界面，支持邮箱密码登录；登录后可在聊天窗口调用后端。
- 提供 `/chat` 聊天接口，使用 LangChain + OpenAI 模型并结合本地 `FAISS` 检索。
- 挂载静态资源 `/static/*`，提供页面与脚本；通过 `/firebase-config` 下发前端所需配置。
- 提供最小化测试覆盖：健康检查、聊天接口、UI 与配置路由。

**目录结构**
- `app.py`：FastAPI 应用与路由（`/`、`/ui`、`/firebase-config`、`/chat`）。
- `static/`：前端页面与静态资源（`index.html`、`app.js`、`styles.css`）。
- `faiss_index/`：向量检索索引（通过 `ingest.py` 生成）。
- `ingest.py`：从 `data.txt` 生成 `FAISS` 索引。
- `tests/test_app.py`：端到端接口测试。
- `.github/workflows/main.yml`：CI/CD 工作流。
- `Dockerfile`、`requirements.txt`：容器与依赖定义。

**环境变量**
- 后端（运行与索引构建）
  - `OPENAI_API_KEY`
- 前端配置（由后端通过 `/firebase-config` 返回）
  - `FIREBASE_API_KEY`
  - `FIREBASE_AUTH_DOMAIN`
  - `FIREBASE_PROJECT_ID`
  - `FIREBASE_APP_ID`
  - `FIREBASE_MESSAGING_SENDER_ID`

**本地开发**
- 安装与运行
  - `python3 -m venv .venv && source .venv/bin/activate`
  - `pip install -r requirements.txt`
  - 可选：设置 `OPENAI_API_KEY` 并运行 `python ingest.py` 以更新向量索引。
  - 启动服务：`uvicorn app:app --reload --host 0.0.0.0 --port 8080`
  - 访问界面：`http://localhost:8080/ui`
- 测试
  - `pip install pytest httpx && pytest -q`

**API**
- `GET /`（`app.py:67-69`）：健康检查，返回 `{"message": ..., "version": ...}`。
- `GET /ui`（`app.py:71-73`）：返回前端页面 `static/index.html`。
- `GET /firebase-config`（`app.py:75-83`）：返回前端初始化所需的 Firebase 配置。
- `POST /chat`（`app.py:85-94`）：请求体 `{"question": "..."}`，返回 `{"answer": "..."}` 或 `{"error": "..."}`。
- 静态资源：`/static/*`（`app.py:96`）。

**聊天记录持久化（Firestore）**
- 启用 Cloud Firestore：在 Firebase 控制台创建项目与 Web 应用，进入 `Firestore Database` 开启数据库（生产模式）。
- 安全规则示例：仅允许用户读写自己的聊天记录。
  ```
  rules_version = '2';
  service cloud.firestore {
    match /databases/{database}/documents {
      match /users/{uid}/chats/{doc} {
        allow read, write: if request.auth != null && request.auth.uid == uid;
      }
    }
  }
  ```
- 写入位置：`users/{uid}/chats`；字段：`question`、`answer`、`error`、`createdAt`。
- 游客模式（`/ui?guest=1`）不写入。
- 登录后前端自动加载历史消息并按时间排序展示，默认最多 100 条。

**CI/CD（GitHub Actions → Railway）**
- 工作流：`.github/workflows/main.yml:1-37`，触发 `push` 到 `main`
  - Checkout 代码：`actions/checkout@v4`（`.github/workflows/main.yml:12`）
  - 安装 Python 与依赖：`actions/setup-python@v5` + `pip install`（`.github/workflows/main.yml:15-24`）
  - 运行测试：`pytest -q`（`.github/workflows/main.yml:26-27`）
  - 构建镜像（健康校验）：`docker build -t rag-app:ci .`（`.github/workflows/main.yml:29-30`）
  - 部署到 Railway：`bervProject/railway-deploy@main`（`.github/workflows/main.yml:32-37`）
- 仓库 Secrets
  - `RAILWAY_TOKEN`：项目令牌（Railway 项目 Settings → Tokens）
  - `RAILWAY_SERVICE_ID`：服务 ID

**部署与使用**
- 在 Railway 项目中设置环境变量：`OPENAI_API_KEY` 与所有 `FIREBASE_*`。
- 推送到 `main` 后，GitHub Actions 将自动构建、测试并部署到 Railway。
- 部署完成后访问 `https://<railway-domain>/ui` 登录并开始聊天。

**文件位置参考**
- 路由与静态挂载：`app.py:67`、`app.py:71-73`、`app.py:75-83`、`app.py:85-94`、`app.py:96`
- 工作流：`.github/workflows/main.yml:1-37`
- 前端：`static/index.html`、`static/app.js`、`static/styles.css`
- 测试：`tests/test_app.py`

**参考链接**
- Railway 文章：`https://blog.railway.com/p/github-actions`
- 部署 Action：`https://github.com/bervProject/railway-deploy`

随便改点啥
