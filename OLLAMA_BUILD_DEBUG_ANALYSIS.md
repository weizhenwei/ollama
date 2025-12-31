# Ollama 项目构建、调试与执行流程分析

## 一、项目概述

Ollama 是一个用于运行大型语言模型的本地化工具，使用 Go 语言开发，集成了 llama.cpp 等 C/C++ 原生代码。项目采用 Cobra 框架构建 CLI 命令行界面，使用 Gin 框架提供 HTTP REST API 服务。

**项目仓库**: `d:\llm\ollama`

---

## 二、项目入口点分析

### 2.1 主入口文件

**文件路径**: `d:\llm\ollama\main.go`

```go
package main

import (
	"context"

	"github.com/spf13/cobra"

	"github.com/ollama/ollama/cmd"
)

func main() {
	cobra.CheckErr(cmd.NewCLI().ExecuteContext(context.Background()))
}
```

**执行流程**:
1. `main()` 函数是整个程序的入口点
2. 调用 `cmd.NewCLI()` 创建 Cobra 命令行应用
3. 使用 `ExecuteContext()` 执行命令，传入 `context.Background()`
4. `cobra.CheckErr()` 处理可能的错误

### 2.2 CLI 命令结构

**文件路径**: `d:\llm\ollama\cmd\cmd.go`

**核心函数**: `NewCLI()` (第 1682 行)

```go
func NewCLI() *cobra.Command {
	log.SetFlags(log.LstdFlags | log.Lshortfile)
	cobra.EnableCommandSorting = false

	if runtime.GOOS == "windows" && term.IsTerminal(int(os.Stdout.Fd())) {
		console.ConsoleFromFile(os.Stdin) //nolint:errcheck
	}

	rootCmd := &cobra.Command{
		Use:           "ollama",
		Short:         "Large language model runner",
		SilenceUsage:  true,
		SilenceErrors: true,
		CompletionOptions: cobra.CompletionOptions{
			DisableDefaultCmd: true,
		},
		Run: func(cmd *cobra.Command, args []string) {
			if version, _ := cmd.Flags().GetBool("version"); version {
				versionHandler(cmd, args)
				return
			}

			cmd.Print(cmd.UsageString())
		},
	}

	rootCmd.Flags().BoolP("version", "v", false, "Show version information")
	
	// ... 创建各种子命令
}
```

**支持的主要命令**:
- `serve` / `start` - 启动 Ollama 服务器
- `run` - 运行模型并进行交互
- `pull` - 从注册表拉取模型
- `push` - 推送模型到注册表
- `create` - 从 Modelfile 创建模型
- `show` - 显示模型信息
- `list` / `ls` - 列出本地模型
- `ps` - 列出正在运行的模型
- `cp` - 复制模型
- `rm` - 删除模型
- `signin` / `signout` - 登录/登出 ollama.com

---

## 三、项目编译方法

### 3.1 Windows 平台编译

#### 3.1.1 前置依赖

**必需工具**:
1. **Go 语言环境** - [下载地址](https://go.dev/doc/install)
2. **CMake** - [下载地址](https://cmake.org/download/)
3. **Visual Studio 2022** - 包含 Native Desktop Workload
4. **C/C++ 编译器** - TDM-GCC (amd64) 或 llvm-mingw (arm64)

**可选 GPU 支持**:
- **NVIDIA GPU**: [CUDA SDK](https://developer.nvidia.com/cuda-downloads?target_os=Windows&target_arch=x86_64&target_version=11&target_type=exe_network)
- **AMD GPU**: [ROCm](https://rocm.docs.amd.com/en/latest/) + [Ninja](https://github.com/ninja-build/ninja/releases)
- **Intel/AMD GPU**: [VULKAN SDK](https://vulkan.lunarg.com/sdk/home)

#### 3.1.2 编译步骤

**标准编译（CPU 或 CUDA）**:

```bash
# 1. 配置项目
cmake -B build

# 2. 编译项目（Release 模式）
cmake --build build --config Release

# 3. 运行服务器
go run . serve
```

**Vulkan 支持编译**:

```powershell
# PowerShell
$env:VULKAN_SDK="C:\VulkanSDK\<version>"
cmake -B build
cmake --build build --config Release
```

```cmd
# CMD
set VULKAN_SDK=C:\VulkanSDK\<version>
cmake -B build
cmake --build build --config Release
```

**ROCm 支持编译**:

```bash
cmake -B build -G Ninja -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++
cmake --build build --config Release
```

**Windows ARM 平台**:
```bash
# ARM 平台不支持加速库，直接使用 Go 编译
go run . serve
# 或
go build
```

#### 3.1.3 快速开发运行

```bash
# 直接运行（开发模式）
go run . serve

# 在另一个终端运行模型
go run . run llama3.2
```

**注意事项**:
- Ollama 包含使用 CGO 编译的原生代码
- 如果数据结构变化导致 CGO 不同步，可能会崩溃
- 强制完整重新编译: `go clean -cache`

### 3.2 CMake 构建配置

**文件路径**: `d:\llm\ollama\CMakeLists.txt`

**关键配置**:
```cmake
cmake_minimum_required(VERSION 3.21)
project(Ollama C CXX)

# 构建类型和共享库
set(CMAKE_BUILD_TYPE Release)
set(BUILD_SHARED_LIBS ON)

# C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# GGML 配置
set(GGML_BUILD ON)
set(GGML_SHARED ON)
set(GGML_CCACHE ON)
set(GGML_BACKEND_DL ON)
set(GGML_BACKEND_SHARED ON)

# CUDA 配置
set(GGML_CUDA_PEER_MAX_BATCH_SIZE 128)
set(GGML_CUDA_GRAPHS ON)
set(GGML_CUDA_FA ON)

# 输出目录
set(OLLAMA_BUILD_DIR ${CMAKE_BINARY_DIR}/lib/ollama)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${OLLAMA_BUILD_DIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${OLLAMA_BUILD_DIR})
```

**支持的后端**:
1. **CPU** - ggml-cpu（默认）
2. **CUDA** - ggml-cuda（NVIDIA GPU）
3. **HIP** - ggml-hip（AMD GPU）
4. **Vulkan** - ggml-vulkan（通用 GPU）

### 3.3 加速库检测路径

Ollama 在以下相对路径查找加速库:
- **Windows**: `./lib/ollama`
- **Linux**: `../lib/ollama`
- **macOS**: `.`
- **开发环境**: `build/lib/ollama`

---

## 四、单步调试配置

### 4.1 Visual Studio Code 调试配置

创建 `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Ollama Serve",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}",
            "args": ["serve"],
            "env": {
                "OLLAMA_DEBUG": "1",
                "OLLAMA_HOST": "127.0.0.1:11434"
            },
            "showLog": true,
            "trace": "verbose"
        },
        {
            "name": "Debug Ollama Run",
            "type": "go",
            "request": "launch",
            "mode": "debug",
            "program": "${workspaceFolder}",
            "args": ["run", "llama3.2"],
            "env": {
                "OLLAMA_HOST": "127.0.0.1:11434"
            }
        },
        {
            "name": "Attach to Ollama Process",
            "type": "go",
            "request": "attach",
            "mode": "local",
            "processId": "${command:pickProcess}"
        }
    ]
}
```

### 4.2 Delve 命令行调试

```bash
# 安装 Delve
go install github.com/go-delve/delve/cmd/dlv@latest

# 启动调试服务器
dlv debug . -- serve

# 在 Delve 中设置断点
(dlv) break cmd.RunServer
(dlv) break server.Serve
(dlv) break server.GenerateHandler

# 继续执行
(dlv) continue

# 查看变量
(dlv) print variableName

# 查看调用栈
(dlv) stack

# 单步执行
(dlv) next  # 下一行
(dlv) step  # 进入函数
(dlv) stepout  # 跳出函数
```

### 4.3 关键调试断点位置

**服务器启动流程**:
1. `main.go:12` - `main()` 函数入口
2. `cmd/cmd.go:1682` - `NewCLI()` 创建命令
3. `cmd/cmd.go:1567` - `RunServer()` 启动服务器
4. `server/routes.go:1552` - `Serve()` 初始化服务

**请求处理流程**:
1. `server/routes.go:1461` - `GenerateRoutes()` 路由注册
2. `server/routes.go:176` - `GenerateHandler()` 生成处理
3. `server/routes.go:1854` - `ChatHandler()` 聊天处理
4. `server/routes.go:640` - `EmbedHandler()` 嵌入处理

**模型管理流程**:
1. `server/routes.go:848` - `PullHandler()` 拉取模型
2. `server/routes.go:899` - `PushHandler()` 推送模型
3. `server/routes.go:1217` - `ListHandler()` 列出模型
4. `server/routes.go:982` - `DeleteHandler()` 删除模型

### 4.4 环境变量调试

```bash
# 启用调试日志
export OLLAMA_DEBUG=1

# 设置日志级别
export OLLAMA_LOG_LEVEL=debug

# 指定服务器地址
export OLLAMA_HOST=127.0.0.1:11434

# 指定模型存储路径
export OLLAMA_MODELS=/path/to/models

# 禁用历史记录
export OLLAMA_NOHISTORY=1

# 设置上下文长度
export OLLAMA_CONTEXT_LENGTH=4096

# 设置保持加载时间
export OLLAMA_KEEP_ALIVE=5m

# 设置最大加载模型数
export OLLAMA_MAX_LOADED_MODELS=1

# 设置最大队列数
export OLLAMA_MAX_QUEUE=512

# 设置并行数
export OLLAMA_NUM_PARALLEL=1
```

---

## 五、项目执行基本流程

### 5.1 服务器启动流程

```
main()
  └─> cmd.NewCLI()
       └─> cobra.Command.ExecuteContext()
            └─> RunServer()
                 ├─> initializeKeypair()  // 初始化密钥对
                 │    └─> 生成 ~/.ollama/id_ed25519
                 │
                 ├─> net.Listen("tcp", host)  // 监听端口
                 │
                 └─> server.Serve(ln)
                      ├─> slog.SetDefault()  // 设置日志
                      ├─> fixBlobs()  // 修复 blob
                      ├─> PruneLayers()  // 清理未使用的层
                      ├─> InitScheduler()  // 初始化调度器
                      ├─> GenerateRoutes()  // 生成路由
                      │    ├─> gin.Default()
                      │    ├─> cors.New()
                      │    ├─> allowedHostsMiddleware()
                      │    └─> 注册所有 API 路由
                      │         ├─> /api/generate
                      │         ├─> /api/chat
                      │         ├─> /api/embed
                      │         ├─> /api/pull
                      │         ├─> /api/push
                      │         ├─> /api/create
                      │         ├─> /api/show
                      │         ├─> /api/tags (list)
                      │         ├─> /api/ps
                      │         ├─> /api/delete
                      │         └─> /v1/* (OpenAI 兼容)
                      │
                      ├─> discover.GPUDevices()  // 检测 GPU
                      ├─> sched.Run()  // 启动调度器
                      └─> http.Server.Serve(ln)  // 启动 HTTP 服务
```

### 5.2 模型运行流程 (ollama run)

```
RunHandler()
  ├─> 解析命令行参数
  │    ├─> 模型名称
  │    ├─> 提示词
  │    ├─> 选项 (--format, --think, --keepalive 等)
  │    └─> 检测是否为交互模式
  │
  ├─> api.ClientFromEnvironment()  // 创建 API 客户端
  │
  ├─> client.Show()  // 获取模型信息
  │    └─> 如果模型不存在 -> PullHandler() 拉取模型
  │
  ├─> 检查模型能力
  │    ├─> CapabilityVision (多模态)
  │    ├─> CapabilityThinking (思考模式)
  │    └─> CapabilityEmbedding (嵌入模型)
  │
  ├─> 如果是嵌入模型
  │    └─> generateEmbedding()
  │
  ├─> 如果是交互模式
  │    ├─> loadOrUnloadModel()  // 加载模型
  │    └─> generateInteractive()  // 交互式生成
  │         └─> 循环读取用户输入
  │              ├─> readline 读取
  │              ├─> 处理特殊命令 (/set, /show, /bye 等)
  │              └─> client.Chat() 发送请求
  │
  └─> 如果是非交互模式
       └─> generate()  // 单次生成
            └─> client.Generate() 或 client.Chat()
```

### 5.3 API 请求处理流程 (以 /api/chat 为例)

```
HTTP POST /api/chat
  └─> ChatHandler()
       ├─> c.ShouldBindJSON(&req)  // 解析请求体
       │
       ├─> 验证请求参数
       │    ├─> 模型名称有效性
       │    ├─> top_logprobs 范围 (0-20)
       │    └─> 消息格式
       │
       ├─> GetModel(req.Model)  // 获取模型
       │    └─> 如果不存在 -> 404 错误
       │
       ├─> 检查是否为远程模型
       │    └─> 如果是 -> 转发到远程服务器
       │
       ├─> scheduleRunner()  // 调度运行器
       │    ├─> 验证模型能力
       │    ├─> 合并模型选项
       │    ├─> sched.GetRunner()  // 获取或创建运行器
       │    │    ├─> 检查已加载的运行器
       │    │    ├─> 如果未加载 -> load()
       │    │    │    ├─> discover.GPUDevices()  // 检测 GPU
       │    │    │    ├─> 计算内存需求
       │    │    │    ├─> llm.New()  // 创建 LLM 实例
       │    │    │    └─> runner.Start()  // 启动运行器进程
       │    │    └─> 更新过期时间
       │    └─> 返回 runner, model, options
       │
       ├─> 处理消息
       │    ├─> 合并系统消息
       │    ├─> 处理多模态内容 (图片)
       │    ├─> 应用模板
       │    └─> 过滤思考标签
       │
       ├─> 调用运行器
       │    └─> runner.Completion()
       │         ├─> 通过 RPC 与运行器进程通信
       │         ├─> 传递 prompt 和参数
       │         └─> 流式返回结果
       │
       └─> 返回响应
            ├─> 如果 stream=true -> streamResponse()
            │    └─> 逐行返回 JSON (NDJSON)
            └─> 如果 stream=false -> waitForStream()
                 └─> 等待完成后返回完整 JSON
```

### 5.4 模型加载流程

```
sched.GetRunner()
  └─> load()
       ├─> 检查模型文件
       │    └─> GetModel() -> 读取 manifest
       │
       ├─> 检测 GPU 设备
       │    └─> discover.GPUDevices()
       │         ├─> 检测 CUDA 设备
       │         ├─> 检测 ROCm 设备
       │         ├─> 检测 Metal 设备 (macOS)
       │         └─> 检测 CPU
       │
       ├─> 估算内存需求
       │    ├─> 模型参数大小
       │    ├─> 上下文长度
       │    ├─> 批次大小
       │    └─> KV 缓存大小
       │
       ├─> 选择运行设备
       │    ├─> 优先 GPU
       │    ├─> 计算层分配 (GPU/CPU)
       │    └─> 如果内存不足 -> 降级到 CPU
       │
       ├─> 创建 LLM 实例
       │    └─> llm.New()
       │         ├─> 选择后端库 (CUDA/ROCm/Metal/CPU)
       │         ├─> 加载模型文件
       │         └─> 初始化推理引擎
       │
       ├─> 启动运行器进程
       │    └─> runner.Start()
       │         ├─> 创建子进程
       │         ├─> 建立 RPC 通信
       │         └─> 加载模型到内存
       │
       └─> 注册到调度器
            ├─> 添加到 loaded map
            ├─> 设置过期时间
            └─> 启动监控 goroutine
```

### 5.5 调度器工作流程

```
Scheduler.Run()
  └─> 启动后台 goroutine
       ├─> 监控运行器状态
       │    └─> 每秒检查一次
       │         ├─> 检查过期时间
       │         ├─> 如果过期 -> unload()
       │         └─> 更新统计信息
       │
       ├─> 处理加载请求
       │    └─> 从 pendingReqCh 接收请求
       │         ├─> 检查是否已加载
       │         ├─> 检查内存是否足够
       │         ├─> 如果需要 -> 卸载其他模型
       │         └─> 调用 load() 加载模型
       │
       └─> 处理卸载请求
            └─> 从 unloadReqCh 接收请求
                 ├─> 停止运行器进程
                 ├─> 释放内存
                 └─> 从 loaded map 移除
```

### 5.6 数据流向图

```
用户请求
   │
   ├─> CLI 命令 (ollama run/chat)
   │    └─> API 客户端 -> HTTP 请求
   │
   └─> 直接 HTTP 请求 (curl/SDK)
        │
        ▼
   HTTP Server (Gin)
        │
        ├─> CORS 中间件
        ├─> 主机验证中间件
        │
        ▼
   路由处理器 (Handler)
        │
        ├─> 请求验证
        ├─> 模型查找
        │
        ▼
   调度器 (Scheduler)
        │
        ├─> 检查已加载模型
        ├─> 内存管理
        ├─> 设备选择
        │
        ▼
   运行器 (Runner)
        │
        ├─> RPC 通信
        ├─> 模型推理
        │
        ▼
   LLM 后端 (llama.cpp)
        │
        ├─> CUDA/ROCm/Metal/CPU
        ├─> 矩阵运算
        ├─> Token 生成
        │
        ▼
   流式响应
        │
        ├─> NDJSON 格式
        ├─> 逐 Token 返回
        │
        ▼
   用户接收
```

---

## 六、关键组件详解

### 6.1 命令行接口 (cmd/)

**主要文件**:
- `cmd.go` - 命令定义和处理器
- `interactive.go` - 交互式会话
- `start.go` / `start_windows.go` - 平台特定启动逻辑

**核心功能**:
1. **命令解析** - 使用 Cobra 框架
2. **参数验证** - 模型名称、选项等
3. **服务器心跳检查** - `checkServerHeartbeat()`
4. **进度显示** - 使用 `progress` 包

### 6.2 服务器 (server/)

**主要文件**:
- `routes.go` - HTTP 路由和处理器
- `sched.go` - 模型调度器
- `images.go` - 模型管理
- `download.go` - 模型下载
- `upload.go` - 模型上传

**核心组件**:

**Server 结构体**:
```go
type Server struct {
    addr    net.Addr      // 监听地址
    sched   *Scheduler    // 调度器
    lowVRAM bool          // 低显存模式
}
```

**Scheduler 结构体**:
```go
type Scheduler struct {
    loaded        map[string]*runnerRef  // 已加载的运行器
    pendingReqCh  chan *runnerRequest    // 待处理请求
    unloadReqCh   chan *runnerRef        // 卸载请求
    // ...
}
```

### 6.3 API 客户端 (api/)

**主要文件**:
- `client.go` - HTTP 客户端
- `types.go` - 请求/响应类型

**支持的 API**:
1. `/api/generate` - 文本生成
2. `/api/chat` - 聊天对话
3. `/api/embed` - 文本嵌入
4. `/api/pull` - 拉取模型
5. `/api/push` - 推送模型
6. `/api/create` - 创建模型
7. `/api/show` - 显示模型信息
8. `/api/tags` - 列出模型
9. `/api/ps` - 运行中的模型
10. `/api/delete` - 删除模型

**OpenAI 兼容 API**:
1. `/v1/chat/completions`
2. `/v1/completions`
3. `/v1/embeddings`
4. `/v1/models`

### 6.4 LLM 后端 (llm/)

**主要文件**:
- `llm.go` - LLM 接口定义
- `server.go` - 运行器服务器

**支持的后端**:
1. **llama.cpp** - 主要后端
2. **CUDA** - NVIDIA GPU 加速
3. **ROCm** - AMD GPU 加速
4. **Metal** - Apple GPU 加速
5. **CPU** - CPU 推理

### 6.5 模型转换 (convert/)

**功能**:
- 将 PyTorch/Safetensors 模型转换为 GGUF 格式
- 支持多种模型架构 (Llama, Mistral, Gemma 等)
- 量化支持 (Q4_K_M, Q5_K_M 等)

### 6.6 GPU 发现 (discover/)

**功能**:
- 自动检测可用 GPU
- 获取 GPU 内存信息
- 选择最佳运行设备

---

## 七、常见调试场景

### 7.1 模型加载失败

**可能原因**:
1. 模型文件损坏
2. 内存不足
3. GPU 驱动问题
4. 权限问题

**调试方法**:
```bash
# 启用详细日志
export OLLAMA_DEBUG=1
export OLLAMA_LOG_LEVEL=debug

# 检查模型文件
ollama show <model_name> --verbose

# 查看 GPU 信息
# 在日志中查找 "GPU devices" 相关信息

# 尝试强制使用 CPU
export OLLAMA_NUM_GPU=0
ollama run <model_name>
```

**断点位置**:
- `server/sched.go` - `load()` 函数
- `llm/llm.go` - `New()` 函数
- `discover/discover.go` - `GPUDevices()` 函数

### 7.2 推理速度慢

**可能原因**:
1. 使用 CPU 而非 GPU
2. 上下文长度过大
3. 批次大小不合适
4. 内存交换

**调试方法**:
```bash
# 查看运行中的模型
ollama ps

# 检查 GPU 使用情况
# NVIDIA: nvidia-smi
# AMD: rocm-smi

# 调整参数
ollama run <model> --num-ctx 2048 --num-batch 512
```

**断点位置**:
- `server/sched.go` - `GetRunner()` 函数
- `runner/runner.go` - `Completion()` 函数

### 7.3 内存溢出

**可能原因**:
1. 同时加载多个大模型
2. 上下文长度过大
3. 批次大小过大

**调试方法**:
```bash
# 限制最大加载模型数
export OLLAMA_MAX_LOADED_MODELS=1

# 减小上下文长度
export OLLAMA_CONTEXT_LENGTH=2048

# 启用低显存模式
# 自动检测，总显存 < 20GB 时启用

# 手动卸载模型
ollama stop <model_name>
```

**断点位置**:
- `server/sched.go` - `unloadAllRunners()` 函数
- `server/sched.go` - `expireRunner()` 函数

### 7.4 API 请求失败

**可能原因**:
1. 服务器未启动
2. 端口被占用
3. CORS 问题
4. 请求格式错误

**调试方法**:
```bash
# 检查服务器状态
curl http://localhost:11434/

# 检查版本
curl http://localhost:11434/api/version

# 测试生成
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Hello"
}'

# 查看日志
# 启用 OLLAMA_DEBUG=1
```

**断点位置**:
- `server/routes.go` - 各个 Handler 函数
- `server/routes.go` - `allowedHostsMiddleware()` 中间件

---

## 八、性能优化建议

### 8.1 硬件优化

1. **GPU 选择**
   - NVIDIA: 优先使用 CUDA
   - AMD: 使用 ROCm (Linux)
   - Intel/AMD: 使用 Vulkan

2. **内存配置**
   - 7B 模型: 至少 8GB RAM
   - 13B 模型: 至少 16GB RAM
   - 33B 模型: 至少 32GB RAM

3. **存储优化**
   - 使用 SSD 存储模型
   - 定期清理未使用的模型

### 8.2 软件优化

1. **上下文长度**
   ```bash
   export OLLAMA_CONTEXT_LENGTH=2048  # 默认 2048
   ```

2. **并行处理**
   ```bash
   export OLLAMA_NUM_PARALLEL=4  # 并行请求数
   ```

3. **保持加载时间**
   ```bash
   export OLLAMA_KEEP_ALIVE=5m  # 模型保持加载时间
   ```

4. **批次大小**
   ```bash
   # 在 Modelfile 中设置
   PARAMETER num_batch 512
   ```

5. **量化**
   ```bash
   # 使用量化模型减少内存占用
   ollama pull llama3.2:7b-q4_K_M
   ```

### 8.3 调度优化

1. **限制最大加载模型数**
   ```bash
   export OLLAMA_MAX_LOADED_MODELS=1
   ```

2. **禁用自动清理**
   ```bash
   export OLLAMA_NOPRUNE=1
   ```

3. **调整 GPU 开销**
   ```bash
   export OLLAMA_GPU_OVERHEAD=0  # 默认值因 GPU 而异
   ```

---

## 九、测试方法

### 9.1 单元测试

```bash
# 运行所有测试
go test ./...

# 运行特定包的测试
go test ./server
go test ./api

# 运行特定测试
go test -run TestGenerateHandler ./server

# 启用详细输出
go test -v ./...

# 启用竞态检测
go test -race ./...

# 生成覆盖率报告
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
```

### 9.2 集成测试

```bash
# 启动服务器
go run . serve &

# 等待服务器启动
sleep 2

# 测试 API
curl http://localhost:11434/api/version

# 拉取模型
ollama pull llama3.2:1b

# 测试生成
ollama run llama3.2:1b "Hello"

# 停止服务器
pkill ollama
```

### 9.3 性能测试

```bash
# 使用 benchmark
go test -bench=. ./server

# 使用 pprof 分析
go test -cpuprofile=cpu.prof -memprofile=mem.prof ./server
go tool pprof cpu.prof
go tool pprof mem.prof

# 压力测试
# 使用 Apache Bench
ab -n 100 -c 10 -p request.json -T application/json \
   http://localhost:11434/api/generate
```

---

## 十、常用环境变量完整列表

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `OLLAMA_HOST` | `127.0.0.1:11434` | 服务器监听地址 |
| `OLLAMA_MODELS` | `~/.ollama/models` | 模型存储路径 |
| `OLLAMA_DEBUG` | `false` | 启用调试日志 |
| `OLLAMA_LOG_LEVEL` | `info` | 日志级别 (debug/info/warn/error) |
| `OLLAMA_ORIGINS` | `*` | 允许的 CORS 来源 |
| `OLLAMA_KEEP_ALIVE` | `5m` | 模型保持加载时间 |
| `OLLAMA_MAX_LOADED_MODELS` | `0` (无限制) | 最大同时加载模型数 |
| `OLLAMA_MAX_QUEUE` | `512` | 最大请求队列长度 |
| `OLLAMA_NUM_PARALLEL` | `0` (自动) | 并行处理请求数 |
| `OLLAMA_CONTEXT_LENGTH` | `2048` | 默认上下文长度 |
| `OLLAMA_NOPRUNE` | `false` | 禁用自动清理 |
| `OLLAMA_NOHISTORY` | `false` | 禁用历史记录 |
| `OLLAMA_FLASH_ATTENTION` | `false` | 启用 Flash Attention |
| `OLLAMA_KV_CACHE_TYPE` | `f16` | KV 缓存类型 |
| `OLLAMA_LLM_LIBRARY` | (自动检测) | LLM 库路径 |
| `OLLAMA_GPU_OVERHEAD` | (自动) | GPU 内存开销 |
| `OLLAMA_LOAD_TIMEOUT` | `5m` | 模型加载超时时间 |
| `OLLAMA_SCHED_SPREAD` | `false` | 调度分散策略 |
| `OLLAMA_NUM_GPU` | (自动检测) | 使用的 GPU 数量 |

---

## 十一、项目结构总结

```
ollama/
├── main.go                 # 程序入口
├── cmd/                    # CLI 命令实现
│   ├── cmd.go             # 命令定义
│   ├── interactive.go     # 交互式会话
│   └── start*.go          # 平台特定启动
├── server/                 # HTTP 服务器
│   ├── routes.go          # 路由和处理器
│   ├── sched.go           # 模型调度器
│   ├── images.go          # 模型管理
│   ├── download.go        # 模型下载
│   └── upload.go          # 模型上传
├── api/                    # API 客户端和类型
│   ├── client.go          # HTTP 客户端
│   └── types.go           # 请求/响应类型
├── llm/                    # LLM 后端接口
│   ├── llm.go             # 接口定义
│   └── server.go          # 运行器服务器
├── runner/                 # 模型运行器
├── discover/               # GPU 设备发现
├── convert/                # 模型转换工具
├── parser/                 # Modelfile 解析器
├── format/                 # 格式化工具
├── progress/               # 进度显示
├── auth/                   # 认证
├── envconfig/              # 环境配置
├── version/                # 版本信息
├── ml/                     # 机器学习后端
│   └── backend/
│       └── ggml/          # GGML 集成
├── llama/                  # llama.cpp 集成
├── app/                    # 桌面应用
├── docs/                   # 文档
│   └── development.md     # 开发指南
├── CMakeLists.txt         # CMake 构建配置
├── go.mod                 # Go 模块定义
└── README.md              # 项目说明
```

---

## 十二、参考资源

### 12.1 官方文档

- **项目主页**: https://ollama.com
- **GitHub 仓库**: https://github.com/ollama/ollama
- **API 文档**: https://github.com/ollama/ollama/blob/main/docs/api.md
- **开发指南**: https://github.com/ollama/ollama/blob/main/docs/development.md
- **模型库**: https://ollama.com/library

### 12.2 相关技术

- **Cobra**: https://github.com/spf13/cobra
- **Gin**: https://github.com/gin-gonic/gin
- **llama.cpp**: https://github.com/ggml-org/llama.cpp
- **GGML**: https://github.com/ggml-org/ggml
- **Delve**: https://github.com/go-delve/delve

### 12.3 社区资源

- **Discord**: https://discord.gg/ollama
- **Reddit**: https://reddit.com/r/ollama
- **Python 库**: https://github.com/ollama/ollama-python
- **JavaScript 库**: https://github.com/ollama/ollama-js

---

## 结论

Ollama 是一个设计精良的本地 LLM 运行工具，采用模块化架构，支持多种 GPU 加速后端。通过本文档，您应该能够:

1. ✅ 理解项目的入口点和基本结构
2. ✅ 掌握在 Windows 平台上编译项目的方法
3. ✅ 配置 VS Code 或 Delve 进行单步调试
4. ✅ 了解从命令行到模型推理的完整执行流程
5. ✅ 定位和解决常见问题
6. ✅ 优化性能和资源使用

**建议的学习路径**:
1. 先运行 `go run . serve` 启动服务器
2. 使用 `ollama run llama3.2:1b` 测试小模型
3. 在 VS Code 中设置断点，单步调试关键流程
4. 阅读 `server/routes.go` 和 `server/sched.go` 源码
5. 尝试修改代码并重新编译测试

**调试技巧**:
- 始终启用 `OLLAMA_DEBUG=1` 查看详细日志
- 使用 `ollama ps` 监控运行中的模型
- 使用 `ollama show <model> --verbose` 查看模型详情
- 在关键函数设置断点: `RunServer`, `ChatHandler`, `GetRunner`
- 使用 `go tool pprof` 分析性能瓶颈

祝您调试顺利！🚀
