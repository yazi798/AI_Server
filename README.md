# AI 电商智能客服系统

基于 **Spring AI + RAG** 的电商 AI 客服系统。后端使用 Spring Boot 3 + Spring AI Alibaba（通义千问）接入大模型，并通过 Milvus 向量库对商品文档做检索增强生成（RAG）；前端为 Vue 3 单页应用，提供**商户管理端**与**用户咨询端**两套界面，聊天通过 WebSocket 实时双向通信。

## 技术栈

| 层次 | 技术 |
| --- | --- |
| 后端框架 | Spring Boot 3.3.13 / Java 17 |
| AI 能力 | Spring AI 1.0.3、Spring AI Alibaba 1.0.0.4、DashScope（`qwen-plus` 对话、`text-embedding-v4` 向量，2048 维） |
| 向量检索 | Milvus（`spring-ai-starter-vector-store-milvus`），索引 `ivf_flat`，度量 `cosine` |
| 持久化 | MySQL 8 + MyBatis-Plus 3.5.9 |
| 缓存/会话 | Redis（会话、角色、多轮记忆均设 10 分钟过期） |
| 实时通信 | Spring WebSocket |
| 工具库 | Lombok、Hutool |
| 前端 | Vue 3.5、Vite 7、TypeScript 5.9、Vue Router 4、Axios、marked |

## 目录结构

```
.
├── server/                 # Spring Boot 后端
│   ├── src/main/java/baize/code/java/
│   │   ├── ai/             # AIService：大模型调用与 RAG 编排
│   │   ├── config/         # AI / 向量库 / WebSocket / MyBatis 配置
│   │   ├── controller/     # REST 接口（用户、商户、商品、会话、角色、日志）
│   │   ├── entity/ mapper/ service/   # 领域模型与数据访问层
│   │   ├── websocket/      # 端点与消息模型（ChatMessage）
│   │   └── utils/          # KeyUtils 等
│   ├── src/main/resources/application.yaml
│   └── pom.xml
├── chat-client/            # Vue 3 前端
│   └── src/
│       ├── api/            # 后端接口封装
│       ├── router/         # 路由（商户端 / 用户端）
│       ├── utils/          # WebSocket 与本地存储封装
│       └── views/
│           ├── ct/         # 商户端：登录、首页、商品管理、AI 设置、客服会话
│           └── user/       # 用户端：登录、首页、商品列表、智能咨询
├── sql/                    # 数据库初始化脚本
├── test/                   # RAG 知识库样例资料（商品文档、提示词）
└── openapi.json            # 接口描述
```

## 核心功能

- **RAG 知识库问答**：商品 Markdown / PDF 文档经 `spring-ai-markdown-document-reader`、`spring-ai-pdf-document-reader` 解析后向量化入库，检索相似度阈值 `0.3`、召回 `5` 条，作为上下文交给大模型生成回答。
- **多轮会话与记忆**：会话上下文与记忆存于 Redis，按 `session`/`role`/`memory` 前缀区分，10 分钟过期，记忆窗口保留最近 10 轮。
- **WebSocket 实时聊天**：`WebSocketEndpoint`、`CommercialTenantEndpoint`、`UserServiceEndpoint` 分别承载通用、商户端与用户端连接。
- **商户（多租户）管理**：`CommercialTenant` 维度管理商品、AI 参数、会话日志；`GoodsDocument` 维护知识库文档。
- **敏感词过滤**：`SensitiveWords` + `SensitiveWordsService` 对输出做风控。
- **会话日志**：`SessionLog` 记录完整对话用于回溯与质检。

## 本地运行

### 1. 前置依赖

- JDK 17、Maven（或直接使用仓库内 `server/mvnw`）
- Node.js 18+
- MySQL 8、Redis、Milvus（默认 `localhost:19530`）

### 2. 初始化数据库

```bash
mysql -u root -p < sql/ai_customer_service.sql
```

### 3. 配置

`server/src/main/resources/application.yaml` 中的连接信息请按本机环境调整。**大模型密钥从环境变量读取，不要写入配置文件：**

```bash
# Windows PowerShell
$env:DASHSCOPE_KEY="sk-你的密钥"
# Linux / macOS
export DASHSCOPE_KEY="sk-你的密钥"
```

默认端口约定：Redis `6380`、Milvus `19530`、MySQL `3306`（数据库 `ai_customer_service`）。

### 4. 启动后端

```bash
cd server
./mvnw spring-boot:run          # Windows: mvnw.cmd spring-boot:run
```

### 5. 启动前端

```bash
cd chat-client
npm install
npm run dev
```

## 说明

- `test/` 目录中的商品资料与提示词仅用于搭建 RAG 知识库和调试，可按需替换为真实商品数据。
- 仓库已忽略 `node_modules/`、`dist/`、`target/` 等构建产物，请勿将密钥、`.env` 或数据库真实密码提交到仓库。
