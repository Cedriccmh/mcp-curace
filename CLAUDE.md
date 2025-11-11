# CLAUDE.md

这个文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 项目概述

这是 **Acemcp** 的 Node.js/TypeScript 实现，一个基于 MCP (Model Context Protocol) 的代码库索引和语义搜索服务器。它为 AI 助手（如 Claude、GPT）提供代码库的语义搜索能力，支持增量索引、跨平台路径处理（包括 WSL），以及 Web 管理界面。

详情见 [README.md](README.md)

**关键特点**：
- 与 Python 版本完全兼容，共享配置文件和数据格式
- 使用单例模式管理配置和日志，确保全局一致性
- 支持复杂的跨平台路径规范化，特别是 WSL 环境
- 通过 stdio 传输实现 MCP 协议，避免污染标准输入输出

## 常用开发命令

```bash
# 安装依赖
npm install

# 编译 TypeScript 到 dist/
npm run build

# 开发模式（支持热重载）
npm run dev

# 启动 MCP 服务器（stdio 模式）
npm start

# 启动带 Web 管理界面（端口 8080）
npm start -- --web-port 8080

# 使用自定义 API 配置
npm start -- --base-url https://api.example.com --token your-token

# 运行测试服务器
npm test
```

## 核心架构设计

### 1. 单例模式的全局状态管理

项目使用单例模式管理两个关键组件：

**Config 单例** ([config.ts](src/config.ts))：
- 管理所有配置参数（API 地址、Token、批处理大小等）
- 从 `~/.acemcp/settings.toml` 加载配置
- 命令行参数优先级高于配置文件
- 支持 `reload()` 热重载配置（Web API 更新配置后使用）
- 首次运行时自动生成默认配置文件

**Logger 单例** ([logger.ts](src/logger.ts))：
- 统一的日志输出，支持三个目标：
  - 文件日志（`~/.acemcp/log/acemcp.log`）：DEBUG 级别及以上，支持轮转（5MB/文件，保留 10 个）
  - 控制台输出（stderr）：INFO 级别及以上，**注意：输出到 stderr 而非 stdout，避免污染 MCP stdio 通信**
  - WebSocket 广播：INFO 级别及以上，实时推送到 Web 界面
- 支持动态添加/移除 WebSocket 广播处理器

**为什么使用单例**：
- 配置在整个应用生命周期中必须保持一致
- 日志系统需要全局可访问，避免传递依赖
- 避免多实例导致的文件锁冲突或配置不一致

### 2. 增量索引机制

索引管理器 ([index/manager.ts](src/index/manager.ts)) 实现了智能的增量索引系统：

**工作流程**：
```
1. 收集文件
   ↓ 遍历项目目录（遵循 .gitignore 和 excludePatterns）
2. 文件分割
   ↓ 大文件按 maxLinesPerBlob（默认 800 行）分割成多个 chunk
3. 计算哈希
   ↓ 对每个 blob 计算 SHA-256（路径 + 内容）
4. 增量比对
   ↓ 与 projects.json 中已存在的 blob 比对
5. 批量上传
   ↓ 只上传新增/修改的 blob（分批次，支持重试）
6. 更新元数据
   ↓ 保存新的 blob 列表到 projects.json
```

**关键设计**：
- **SHA-256 哈希计算**：`hash(文件路径 + 文件内容)` 用作去重标识
- **换行符保留**：分割文件时保留换行符（`\n`、`\r\n`、`\r`），确保与 Python 版本的哈希值完全一致
- **批量上传**：默认每批 10 个 blob，失败时使用指数退避重试，单个批次失败不阻塞后续批次
- **chunk 路径格式**：大文件分割后路径为 `path/to/file.ts#chunk2of5`

### 3. 跨平台路径规范化

路径处理是项目的核心挑战之一，[utils/pathUtils.ts](src/utils/pathUtils.ts) 提供了统一的路径规范化方案：

**支持的路径格式**：
| 输入格式 | 转换后 | 场景 |
|---------|--------|------|
| `C:\Users\username\project` | `C:/Users/username/project` | Windows 本地路径 |
| `/home/user/project` | `/home/user/project` | Unix/Linux 路径 |
| `/mnt/c/Users/username/project` | `C:/Users/username/project` (Windows 环境下) | WSL 访问 Windows 文件 |
| `\\wsl$\Ubuntu\home\user\project` | `/home/user/project` | Windows 访问 WSL 文件 |

**规范化规则**：
1. 所有路径统一使用正斜杠 `/`
2. 移除末尾斜杠（除了根目录 `/` 和 Windows 盘符根 `C:/`）
3. 自动识别并转换 UNC 路径（`\\wsl$\...`）
4. 在 Windows 环境下自动转换 `/mnt/x/` 格式为 `X:/` 格式

**重要提醒**：
- 所有涉及项目路径的代码必须使用 `normalizeProjectPath()` 函数
- 路径存储在 `projects.json` 中时已经是规范化后的格式
- 进行路径比对时必须先规范化再比较

### 4. MCP 协议集成架构

[index.ts](src/index.ts) 实现了 MCP 服务器的核心逻辑：

**通信方式**：
- 使用 `StdioServerTransport` 通过标准输入/输出与 MCP 客户端通信
- **关键约束**：stdout 仅用于 MCP 协议消息，所有日志输出到 stderr 或文件

**工具注册流程**：
```typescript
// 1. 注册工具列表（ListToolsRequestSchema）
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [{ name: 'search_context', description: '...', inputSchema: {...} }]
}));

// 2. 处理工具调用（CallToolRequestSchema）
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments } = request.params;
  if (name === 'search_context') {
    return await searchContextTool(arguments);
  }
});
```

**search_context 工具的执行流程** ([tools/searchContext.ts](src/tools/searchContext.ts))：
1. 验证参数（`project_root_path` 和 `query`）
2. 规范化项目路径
3. 创建 IndexManager 实例
4. 调用 `searchContext()`：
   - 自动执行增量索引
   - 加载 blob 列表
   - 调用搜索 API（`/agents/codebase-retrieval`）
   - 返回格式化的代码片段

### 5. Web 管理界面架构

[web/app.ts](src/web/app.ts) 提供了独立的 Web 管理功能：

**双服务器模式**：
- MCP 服务器（stdio）：主要功能，始终运行
- Web 服务器（HTTP + WebSocket）：可选功能，启动失败不影响 MCP 核心

**优雅降级机制**：
```typescript
// 端口占用时不阻止 MCP 服务器启动
httpServer.on('error', (error) => {
  if (error.code === 'EADDRINUSE') {
    logger.warning(`Port ${port} already in use. Web interface will not start.`);
    resolve(); // 继续启动 MCP 服务器
  }
});
```

**RESTful API 端点**：
- `/api/config` - GET/POST 配置管理（支持热重载）
- `/api/status` - 获取服务器状态和项目数量
- `/api/projects` - 列出所有已索引项目
- `/api/projects/check` - 检查项目是否已索引
- `/api/projects/reindex` - 重新索引指定项目
- `/api/projects/delete` - 删除项目索引
- `/api/tools/execute` - 调试执行 MCP 工具

**WebSocket 实时日志**：
- 路径：`/ws/logs`
- LogBroadcaster ([web/logBroadcaster.ts](src/web/logBroadcaster.ts)) 管理所有 WebSocket 连接
- Logger 自动将 INFO 级别及以上的日志广播到所有连接的客户端

### 6. 多编码支持系统

`readFileWithEncoding()` 函数 ([index/manager.ts](src/index/manager.ts:42-82)) 实现了智能编码检测：

**回退策略**：
```
UTF-8 → GBK → GB2312 → Latin-1 → UTF-8（强制）
```

**验证机制**：
- 检测解码后的替换字符（U+FFFD）数量
- 短文件（< 100 字符）：超过 5 个替换字符则认为编码错误
- 长文件：超过 5% 的替换字符则认为编码错误

**日志策略**：
- 仅在使用非 UTF-8 编码时记录 debug 日志，减少日志噪音
- 所有编码都失败时记录 warning 日志

## 配置系统

### 配置文件结构

**位置**：`~/.acemcp/settings.toml`

**关键配置项**：
```toml
BASE_URL = "https://api.example.com"  # 索引服务 API 地址
TOKEN = "your-token-here"              # 认证令牌
BATCH_SIZE = 10                        # 批量上传数量（1-50）
MAX_LINES_PER_BLOB = 800               # 文件分割阈值
TEXT_EXTENSIONS = [".py", ".js", ...]  # 支持的文件类型
EXCLUDE_PATTERNS = ["node_modules", ...] # 排除模式
```

**数据存储**：`~/.acemcp/data/projects.json`
```json
{
  "C:/Users/username/project": ["blob_hash1", "blob_hash2", ...],
  "/home/user/another-project": ["blob_hash3", "blob_hash4", ...]
}
```

### 配置优先级

命令行参数 > 配置文件 > 默认值

```bash
# 命令行参数会覆盖配置文件
npm start -- --base-url https://custom-api.com --token custom-token
```

## 与 Python 版本的兼容性

此项目与 [Acemcp-Python](https://github.com/yeuxuan/Ace-Mcp-Python) 完全兼容：

**共享资源**：
- `~/.acemcp/settings.toml` - 配置文件
- `~/.acemcp/data/projects.json` - 项目元数据
- API 接口格式（`/batch-upload` 和 `/agents/codebase-retrieval`）

**兼容性保证**：
- 相同的 SHA-256 哈希算法（路径 + 内容，保留换行符）
- 相同的文件分割逻辑（按行数分割，保留换行符）
- 相同的路径规范化规则

**切换版本**：
可以随时在 Node.js 和 Python 版本之间切换，无需迁移数据。

## TypeScript 和 ESM 注意事项

### 导入规范

**必须使用 `.js` 扩展名**（即使源文件是 `.ts`）：
```typescript
// ✅ 正确
import { getConfig } from './config.js';
import { IndexManager } from './index/manager.js';

// ❌ 错误
import { getConfig } from './config';
```

**原因**：TypeScript 编译到 ESM 时，导入路径不会被重写，必须使用实际的 JavaScript 文件扩展名。

### 编译配置

- **目标**：ES2022
- **模块系统**：ES Modules (ESM)
- **输出目录**：`dist/`
- **Source Maps**：已启用，便于调试

**编译流程**：
```bash
npm run build
# 1. tsc 编译 TypeScript
# 2. 复制 web/templates/ 到 dist/web/templates/
```

## 开发最佳实践

### 添加新的 MCP 工具

1. 在 `src/tools/` 创建新文件（如 `myTool.ts`）：
```typescript
export async function myTool(args: { param1: string }): Promise<{ type: 'text'; text: string }> {
  try {
    // 参数验证
    if (!args.param1) {
      return { type: 'text', text: 'Error: param1 is required' };
    }

    // 业务逻辑
    const result = await doSomething(args.param1);

    return { type: 'text', text: result };
  } catch (error: any) {
    logger.exception('Error in myTool', error);
    return { type: 'text', text: `Error: ${error.message}` };
  }
}
```

2. 在 `src/index.ts` 注册工具：
```typescript
// 在 ListToolsRequestSchema 处理器中添加
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    // ... 现有工具
    {
      name: 'my_tool',
      description: '工具描述',
      inputSchema: {
        type: 'object',
        properties: {
          param1: { type: 'string', description: '参数说明' }
        },
        required: ['param1']
      }
    }
  ]
}));

// 在 CallToolRequestSchema 处理器中添加
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name === 'my_tool') {
    const result = await myTool(request.params.arguments as any);
    return { content: [{ type: 'text', text: result.text }] };
  }
  // ...
});
```

### 路径处理规范

**永远使用规范化函数**：
```typescript
import { normalizeProjectPath } from './utils/pathUtils.js';

// 接收用户输入的路径
const userPath = args.project_root_path;

// 立即规范化
try {
  const normalizedPath = normalizeProjectPath(userPath);
  // 使用 normalizedPath 进行后续操作
} catch (error) {
  // 处理无效路径
}
```

**测试不同路径格式**：
- Windows: `C:\Users\username\project`
- Unix: `/home/user/project`
- WSL (Windows→WSL): `\\wsl$\Ubuntu\home\user\project`
- WSL (WSL→Windows): `/mnt/c/Users/username/project`

### 错误处理模式

**API 请求使用重试机制**：
```typescript
// IndexManager 已实现 retryRequest() 方法
const result = await this.retryRequest(
  async () => await this.httpClient.post(url, data),
  3,        // 最大重试次数
  1000      // 初始延迟（指数退避）
);
```

**日志记录异常**：
```typescript
try {
  // 危险操作
} catch (error: any) {
  logger.exception('操作描述', error); // 记录完整堆栈
  return { type: 'text', text: `Error: ${error.message}` };
}
```

### Web API 开发

**添加新端点**：
1. 在 `createApp()` 中添加路由
2. 使用 `getConfig()` 获取配置
3. 使用 `logger` 记录操作
4. 返回统一的 JSON 格式（成功/错误）

**错误响应格式**：
```typescript
res.status(400).json({ error: '错误描述' });
```

**成功响应格式**：
```typescript
res.json({ success: true, message: '操作成功', data: {...} });
```

## 故障排查要点

### 日志文件位置
```
~/.acemcp/log/acemcp.log        # 当前日志
~/.acemcp/log/acemcp.log.1      # 轮转日志
```

### 常见问题

**路径不存在**：
- 检查路径格式是否正确（绝对路径）
- 检查 WSL 路径转换是否正确
- 查看日志中的 `Path check failed` 消息

**API 连接失败**：
- 验证 `BASE_URL` 是否包含协议前缀（自动添加 `https://`）
- 检查 `TOKEN` 是否配置正确
- 查看日志中的重试记录

**编译错误**：
- 确保导入路径包含 `.js` 扩展名
- 运行 `npm run build` 后检查 `dist/` 目录
- 检查 `dist/web/templates/` 是否存在（需要手动复制）

**Web 界面无法访问**：
- 检查端口是否被占用（默认 8080）
- 查看日志中的 `EADDRINUSE` 错误
- 使用不同端口：`npm start -- --web-port 3000`

### 调试技巧

**查看详细日志**：
```bash
tail -f ~/.acemcp/log/acemcp.log
```

**测试工具调用**：
使用 Web 界面的"工具调试"功能，或直接调用 `/api/tools/execute`：
```bash
curl -X POST http://localhost:8080/api/tools/execute \
  -H "Content-Type: application/json" \
  -d '{
    "tool_name": "search_context",
    "arguments": {
      "project_root_path": "C:/your/project",
      "query": "your search query"
    }
  }'
```

**检查项目索引状态**：
```bash
cat ~/.acemcp/data/projects.json | jq
```

## 项目维护

### 发布新版本

1. 更新 `package.json` 中的版本号
2. 更新 `src/index.ts` 中的版本号（Server 初始化）
3. 编译并测试：`npm run build && npm test`
4. 发布到 NPM：`npm publish`

### 文件结构说明

**关键文件**：
- `src/index.ts` - MCP 服务器入口和协议实现
- `src/config.ts` - 配置管理单例
- `src/logger.ts` - 日志系统单例
- `src/index/manager.ts` - 索引和搜索核心逻辑（最复杂的文件，约 680 行）
- `src/utils/pathUtils.ts` - 跨平台路径规范化
- `src/tools/searchContext.ts` - search_context 工具实现
- `src/web/app.ts` - Web 管理界面和 API

**配置和数据**：
- `~/.acemcp/settings.toml` - 用户配置
- `~/.acemcp/data/projects.json` - 项目索引元数据
- `~/.acemcp/log/acemcp.log` - 日志文件

### 依赖说明

**核心依赖**：
- `@modelcontextprotocol/sdk` - MCP 协议实现
- `axios` - HTTP 客户端（API 请求）
- `express` + `ws` - Web 服务器和 WebSocket
- `@iarna/toml` - TOML 配置文件解析
- `ignore` - .gitignore 模式匹配
- `iconv-lite` - 多编码支持

**开发依赖**：
- `typescript` - TypeScript 编译器
- `tsx` - TypeScript 执行器（开发模式）
- `@types/*` - 类型定义

## 架构决策记录

### 为什么使用 stdio 而非 HTTP？
MCP 协议设计为使用 stdio 传输，确保与 MCP 客户端（如 Claude Desktop）的兼容性。Web 界面是独立的可选功能。

### 为什么日志输出到 stderr？
stdout 被 MCP 协议占用，任何输出到 stdout 的内容都会被解释为 MCP 消息，导致协议错误。因此所有日志必须输出到 stderr 或文件。

### 为什么保留换行符？
为了与 Python 版本的 SHA-256 哈希保持一致。Python 的 `splitlines(keepends=True)` 保留换行符，Node.js 版本必须模拟这一行为。

### 为什么使用批量上传？
单个文件上传效率低，批量上传（默认 10 个/批）大幅提升性能。同时支持失败重试和部分成功，提高容错性。

### 为什么 Web 服务器失败不阻塞 MCP？
MCP 核心功能是代码搜索，Web 界面只是辅助工具。即使 Web 启动失败（如端口占用），MCP 服务器仍应正常工作。
