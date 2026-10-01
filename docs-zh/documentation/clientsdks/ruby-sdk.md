---
description: "使用地道的任务定义和工作流管理，用 Ruby 构建 Conductor 工作者。"
source_repo: "https://github.com/conductor-oss/ruby-sdk"
sdk_page: ruby
---

# Ruby SDK

## 特性

- 与 Python SDK **功能完全对等**
- **地道的 Ruby 工作流 DSL** - 简洁的基于块的语法，支持 25+ 种任务类型
- **工作者框架** - 基于类的工作者和基于块的工作者的多线程任务执行
- **LLM/AI 任务** - 聊天补全、嵌入、RAG、图像/音频生成
- **Orkes Cloud 支持** - 身份验证、密钥、集成、提示词
- **全面的测试** - 400+ 个单元测试，110 个集成测试

## 安装

添加到你的 Gemfile：

```ruby
gem 'conductor_ruby'
```

或者直接安装：

```bash
gem install conductor_ruby
```

## 快速开始

### Hello World

```ruby
require 'conductor'

# Configuration (reads CONDUCTOR_SERVER_URL from environment)
config = Conductor::Configuration.new

# Create clients
clients = Conductor::Orkes::OrkesClients.new(config)
executor = clients.get_workflow_executor

# Define a worker
class GreetWorker
  include Conductor::Worker::WorkerModule
  worker_task 'greet'

  def execute(task)
    name = get_input(task, 'name', 'World')
    { 'result' => "Hello, #{name}!" }
  end
end

# Build workflow using new DSL
workflow = Conductor.workflow :greetings, version: 1, executor: executor do
  greet = simple :greet, name: wf[:name]
  output result: greet[:result]
end

# Register and execute
workflow.register(overwrite: true)

# Start workers
runner = Conductor::Worker::TaskRunner.new(config)
runner.register_worker(GreetWorker.new)
runner.start

# Execute workflow
result = workflow.execute(input: { 'name' => 'Ruby' }, wait_for_seconds: 30)
puts "Result: #{result.output['result']}"  # => "Hello, Ruby!"

runner.stop
```

## 工作流 DSL

SDK 提供了一个简洁、地道的 Ruby 风格 DSL 用于构建工作流：

```ruby
workflow = Conductor.workflow :order_processing, version: 1, executor: executor do
  # Access workflow inputs with wf[:param]
  user = simple :get_user, user_id: wf[:user_id]
  
  # Reference task outputs with task[:field]
  order = simple :validate_order, email: user[:email]
  
  # HTTP calls
  http :call_api, url: 'https://api.example.com', method: :post, body: { id: order[:id] }
  
  # Parallel execution
  parallel do
    simple :ship_order, order_id: order[:id]
    simple :send_confirmation, email: user[:email]
  end
  
  # Conditional branching
  decide order[:region] do
    on 'US' do
      simple :us_shipping
    end
    on 'EU' do
      simple :eu_shipping
    end
    otherwise do
      terminate :failed, 'Unsupported region'
    end
  end
  
  # Set workflow output
  output tracking: order[:tracking_number], status: 'completed'
end

# Register and execute
workflow.register(overwrite: true)
result = workflow.execute(input: { user_id: 123 }, wait_for_seconds: 60)
```

### 任务方法参考

#### 基础任务

```ruby
# Simple task (worker execution)
result = simple :task_name, input1: 'value', input2: wf[:param]

# Inline code execution
jq :transform, query: '.items | map(.name)', input: previous[:data]
javascript :compute, script: 'return inputs.a + inputs.b', a: 1, b: 2

# Set workflow variables
set_variable :save_state, user_id: user[:id], status: 'active'

# Human/manual task
human :approval, display_name: 'Manager Approval', form_template: 'approval_form'
```

#### HTTP 任务

```ruby
# HTTP request
http :call_api,
  url: 'https://api.example.com/users',
  method: :post,
  headers: { 'Authorization' => 'Bearer ${workflow.secrets.api_token}' },
  body: { name: wf[:name], email: wf[:email] }

# HTTP polling (wait for condition)
http_poll :wait_for_ready,
  url: 'https://api.example.com/status/${workflow.input.job_id}',
  method: :get,
  termination_condition: '$.status == "ready"',
  polling_interval: 5,
  polling_strategy: :fixed
```

#### 控制流

```ruby
# Parallel execution (fork/join)
parallel do
  simple :branch_a
  simple :branch_b
  simple :branch_c
end

# Conditional branching
decide order[:status] do
  on 'pending' do
    simple :process_pending
  end
  on 'approved' do
    simple :process_approved
  end
  otherwise do
    simple :handle_unknown
  end
end

# Conditional shortcuts
when_true user[:is_premium] do
  simple :apply_discount
end

when_false order[:validated] do
  terminate :failed, 'Order validation failed'
end

# Loop over items
loop_over users[:list], as: :user do
  simple :process_user, user_id: iteration[:user][:id]
end

# Do-while loop
do_while :retry_loop, condition: '${retry_ref.output.success} == false' do
  simple :retry_operation
end
```

#### 子工作流

```ruby
# Call another workflow
sub_workflow :process_order,
  workflow_name: 'order_processor',
  version: 2,
  input: { order_id: wf[:order_id] }

# Start workflow (fire-and-forget)
start_workflow :trigger_notification,
  workflow_name: 'send_notifications',
  input: { user_id: user[:id] }

# Inline sub-workflow definition
inline_workflow :nested_process do
  simple :step1
  simple :step2
end
```

#### 等待与事件

```ruby
# Wait for duration
wait :pause, duration: '30s'   # or '5m', '1h', '2d'

# Wait until specific time
wait :scheduled, until: '2024-12-25T00:00:00Z'

# Wait for external webhook
wait_for_webhook :external_callback,
  matches: { 'type' => 'payment', 'order_id' => '${workflow.input.order_id}' }

# Publish event
event :notify, sink: 'conductor:workflow_events', payload: { status: 'completed' }
```

#### 终止

```ruby
# Complete workflow
terminate :success, 'Processing completed successfully'

# Fail workflow
terminate :failed, 'Validation error: missing required field'
```

#### 动态任务

```ruby
# Dynamic task name (resolved at runtime)
dynamic :run_handler, task_to_execute: wf[:handler_name]

# Dynamic fork (parallel tasks determined at runtime)
dynamic_fork :process_all,
  tasks_input: wf[:items],
  task_name: 'process_item'
```

### LLM/AI 任务

```ruby
workflow = Conductor.workflow :ai_assistant, executor: executor do
  # Chat completion (messages auto-converted from simple format)
  response = llm_chat :chat,
    provider: 'openai',
    model: 'gpt-4',
    messages: [
      { role: :system, message: 'You are a helpful assistant.' },
      { role: :user, message: wf[:question] }
    ],
    temperature: 0.7

  # Text completion
  llm_text :complete,
    provider: 'anthropic',
    model: 'claude-3-sonnet',
    prompt: 'Summarize: ${workflow.input.text}'

  # Generate embeddings
  embeddings = llm_embeddings :embed,
    provider: 'openai',
    model: 'text-embedding-3-small',
    text: wf[:document]

  # Store embeddings in vector DB
  llm_store_embeddings :store,
    provider: 'pinecone',
    index: 'documents',
    embeddings: embeddings[:embeddings],
    metadata: { doc_id: wf[:doc_id] }

  # Search embeddings
  llm_search_embeddings :search,
    provider: 'pinecone',
    index: 'documents',
    query: wf[:search_query],
    max_results: 10

  # Generate image
  generate_image :create_image,
    provider: 'openai',
    model: 'gpt-image-1',
    prompt: 'A sunset over mountains',
    size: '1024x1024'

  # Generate audio (text-to-speech)
  generate_audio :speak,
    provider: 'openai',
    model: 'tts-1',
    text: response[:content],
    voice: 'nova'

  # MCP (Model Context Protocol) integration
  tools = list_mcp_tools :get_tools, server_name: 'my_mcp_server'
  
  call_mcp_tool :use_tool,
    server_name: 'my_mcp_server',
    tool_name: 'search_documents',
    arguments: { query: wf[:query] }

  output answer: response[:content]
end
```

### 输出引用

DSL 使用简洁的语法来引用输出：

```ruby
# Workflow input reference
wf[:user_id]              # => '${workflow.input.user_id}'

# Task output reference
task[:field]              # => '${task_ref.output.field}'
task[:nested][:path]      # => '${task_ref.output.nested.path}'

# Loop iteration references (inside loop_over)
iteration[:current_item]  # Current item being processed
iteration[:index]         # Current index (0-based)
iteration[:user][:name]   # If `as: :user` specified
```

## 示例

`examples/` 目录包含全面的示例：

| Example | Description |
|---------|-------------|
| [`helloworld/`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/helloworld/) | 最简完整示例 - 工作者 + 工作流 + 执行 |
| [`workflow_dsl.rb`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/workflow_dsl.rb) | 全面的新 DSL 展示 |
| [`simple_worker.rb`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/simple_worker.rb) | 工作者模式：基于类、基于块、错误处理 |
| [`kitchensink.rb`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/kitchensink.rb) | 使用新 DSL 的所有主要任务类型 |
| [`dynamic_workflow.rb`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/dynamic_workflow.rb) | 在运行时创建并执行工作流 |
| [`workflow_ops.rb`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/workflow_ops.rb) | 生命周期操作：暂停、恢复、重启、重试 |
| [`agentic_workflows/`](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/agentic_workflows/) | LLM 聊天和 AI 工作流示例 |

运行示例：

```bash
# Set environment variables
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
# For Orkes Cloud:
# export CONDUCTOR_AUTH_KEY=your_key
# export CONDUCTOR_AUTH_SECRET=your_secret

# Run hello world
cd examples/helloworld && bundle exec ruby helloworld.rb

# Run DSL showcase
bundle exec ruby examples/workflow_dsl.rb

# Run kitchen sink
bundle exec ruby examples/kitchensink.rb
```

## 工作者框架

### 基于类的工作者

```ruby
class ImageProcessor
  include Conductor::Worker::WorkerModule

  worker_task 'process_image', poll_interval: 1, thread_count: 4

  def execute(task)
    url = get_input(task, 'image_url')
    # Process image...
    
    result = Conductor::Http::Models::TaskResult.complete
    result.add_output_data('processed_url', processed_url)
    result.log('Image processed successfully')
    result
  end
end
```

### 基于块的工作者

```ruby
worker = Conductor::Worker.define('simple_task') do |task|
  input = task.input_data['value']
  { result: input * 2 }  # Return hash for automatic TaskResult
end
```

### 运行工作者

```ruby
runner = Conductor::Worker::TaskRunner.new(config)
runner.register_worker(ImageProcessor.new)
runner.register_worker(worker)
runner.start(threads: 4)

# Graceful shutdown
trap('INT') { runner.stop }
sleep while runner.running?
```

## 配置

### 环境变量

```bash
export CONDUCTOR_SERVER_URL=http://localhost:8080/api
export CONDUCTOR_AUTH_KEY=your_key        # For Orkes Cloud
export CONDUCTOR_AUTH_SECRET=your_secret  # For Orkes Cloud
```

### 编程方式

```ruby
config = Conductor::Configuration.new(
  server_api_url: 'https://play.orkes.io/api',
  auth_key: 'your_key',
  auth_secret: 'your_secret',
  auth_token_ttl_min: 45,
  verify_ssl: true
)
```

## API 覆盖范围

### 资源 API（17 个类）

| API | Description |
|-----|-------------|
| WorkflowResourceApi | 工作流执行与管理 |
| TaskResourceApi | 任务轮询与更新 |
| MetadataResourceApi | 工作流/任务定义 |
| SchedulerResourceApi | 定时工作流 |
| EventResourceApi | 事件处理器 |
| WorkflowBulkResourceApi | 批量操作 |
| PromptResourceApi | AI 提示词模板 |
| SecretResourceApi | 密钥管理 |
| IntegrationResourceApi | 外部集成 |
| + 8 个更多 | 授权、用户、群组、角色等 |

### 高级客户端（9 个类）

```ruby
clients = Conductor::Orkes::OrkesClients.new(config)

workflow_client = clients.get_workflow_client
task_client = clients.get_task_client
metadata_client = clients.get_metadata_client
scheduler_client = clients.get_scheduler_client
prompt_client = clients.get_prompt_client
secret_client = clients.get_secret_client
authorization_client = clients.get_authorization_client
workflow_executor = clients.get_workflow_executor
```

## 测试

```bash
# Unit tests
bundle exec rspec spec/conductor/

# Integration tests (requires Conductor server)
CONDUCTOR_SERVER_URL=http://localhost:8080/api bundle exec rspec spec/integration/
```

## 要求

- Ruby 2.6+（推荐 Ruby 3+）
- Conductor OSS 3.x 或 Orkes Cloud

## 依赖

- `faraday ~> 2.0` - HTTP 客户端
- `faraday-net_http_persistent ~> 2.0` - 连接池
- `faraday-retry ~> 2.0` - 自动重试
- `concurrent-ruby ~> 1.2` - 线程安全的并发

## 贡献

1. Fork 仓库
2. 创建你的功能分支（`git checkout -b feature/amazing-feature`）
3. 运行测试（`bundle exec rspec`）
4. 提交你的更改（`git commit -m 'Add amazing feature'`）
5. 推送到分支（`git push origin feature/amazing-feature`）
6. 打开一个 Pull Request

## 许可证

Apache 2.0 - 详情见 [LICENSE](https://github.com/conductor-oss/ruby-sdk/blob/main/LICENSE)。

## 链接

- [Conductor OSS](https://github.com/conductor-oss/conductor)
- [Orkes Cloud](https://orkes.io)
- [文档](https://conductor-oss.org)
- [Python SDK](https://github.com/conductor-sdk/conductor-python)
- [社区 Slack](https://join.slack.com/t/orkes-conductor/shared_invite/zt-2vdbx239s-Eacdyqya9giNLHfrCavfaA)


## Examples

Browse all examples on GitHub: [conductor-oss/ruby-sdk/examples](https://github.com/conductor-oss/ruby-sdk/tree/main/examples)

| Example | Type |
|---|---|
| [Agentic Workflows](https://github.com/conductor-oss/ruby-sdk/tree/main/examples/agentic_workflows) | directory |
| [Dynamic Workflow](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/dynamic_workflow.rb) | file |
| [Event Handler](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/event_handler.rb) | file |
| [Event Listener Examples](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/event_listener_examples.rb) | file |
| [Helloworld](https://github.com/conductor-oss/ruby-sdk/tree/main/examples/helloworld) | directory |
| [Kitchensink](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/kitchensink.rb) | file |
| [Metadata Journey](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/metadata_journey.rb) | file |
| [Metrics Example](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/metrics_example.rb) | file |
| [New Dsl Demo](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/new_dsl_demo.rb) | file |
| [Orkes](https://github.com/conductor-oss/ruby-sdk/tree/main/examples/orkes) | directory |
| [Prompt Journey](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/prompt_journey.rb) | file |
| [Rag Workflow](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/rag_workflow.rb) | file |
| [Schedule Journey](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/schedule_journey.rb) | file |
| [Simple Worker](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/simple_worker.rb) | file |
| [Simple Workflow](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/simple_workflow.rb) | file |
| [Task Context Example](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/task_context_example.rb) | file |
| [Task Listener Example](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/task_listener_example.rb) | file |
| [Worker Configuration Example](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/worker_configuration_example.rb) | file |
| [Workflow Dsl](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/workflow_dsl.rb) | file |
| [Workflow Ops](https://github.com/conductor-oss/ruby-sdk/blob/main/examples/workflow_ops.rb) | file |
