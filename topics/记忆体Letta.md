# letta

## paper

### MemGPT: Towards LLMs as Operating Systems

大模型的窗口是有限长的，为了突破这个限制，设计了这么个系统； 为什么不直接extending the context length，两个原因不行
1. a quadratic increase in computational time and memory cost due to the transformer architecture’s self-attention mechanism
2. “加长上下文”并不等于“用足上下文”，U型注意力——信息位于开头或末尾时召回率高，位于中段时显著下降

提到的两个应用场景：
1. document analysis
2. multi-session chat

memgpt把空间分成两个部分：main context (analogous to main memory/physical memory/RAM)and external context (analogous to disk memory/disk storage).

![memgpt.png](../images/memgpt.png)

MemGPT provides function calls that the LLM processor to manage its own memory without any user intervention.

- Main context (prompt tokens)

    **system instructions**: contain information on the MemGPT control flow, the intended usage of the different memory levels, and instructions on how to use the MemGPT functions
    
    **working context**: store key facts, preferences, and other important information about the user and the persona
    
    **FIFO queue**: stores a rolling history of messages, including messages between the agent and user, as well as system
    messages (e.g. memory warnings) and function call inputs and outputs. The first index in the FIFO queue stores a system message containing a recursive summary of messages
    that have been evicted from the queue

- Queue Manager

  1. manages messages in recall storage and the FIFO queue
  
      当来了一条新的msg，他会把他放在FIFO queue的队尾，并且把msg和大模型的输出一起放到recall storage，当function call从recall storage检索的到相关数据的时候，他又会把数据放到context window

  2. 通过“队列驱逐策略”控制上下文溢出

      当token数达到上下文窗口的“警告阈值”（例如 70%）时，队列管理器向队列插入一条系统消息，提醒 LLM“内存压力”即将到来，让LLM可使用function call把FIFO 队列中的重要信息转存到工作上下文或归档存储。
    
      当oken数达到“刷新阈值”（例如 100%）时，队列管理器执行刷新：按预设比例（例如 50%）驱逐最旧消息，并与现有的递归摘要一起生成新的递归摘要，以释放上下文空间。

- Function executor

通过LLM生成的函数调用来main context and external context之间的数据移动。所有记忆的编辑与检索完全由系统自主完成

其实就是个推理循环

![function executor.png](../images/function executor.png)

memgpt是由事件触发LLM推理，可以是：
1. 用户消息
2. 系统消息（如主上下文容量警告）
3. 用户交互事件

函数链（function chaining）允许在一次用户回合内连续执行多个函数，而无需把控制权交还给用户。

在调用函数时，MemGPT 可设置一个特殊标志位（request heartbeat）。

若置位：函数返回后，立即把输出追加到主上下文并立刻再次调用 LLM 处理器，继续下一步推理，若未置位：函数返回后，暂停处理器

## letta源码分析

### 环境搭建

#### 数据库建表

1. postgresql + pgvector

    推荐使用postgresql + pgvector，并且建立letta数据库，不要用默认的sqllite

    ```shell
    apt install -y postgresql-18-pgvector
    sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS vector;"
    SELECT extname, extversion FROM pg_extension WHERE extname='vector';
    netstat -anp | grep 5432
    ```

2. 通过alembic创建表

    uv run alembic upgrade head

3. 配置环境变量

    配置环境变量$LETTA_PG_URI=postgresql+pg8000://{user}:{password}@{ip}:5432/letta

4. llm配置

    用deepseek的api，配置环境变量LETTA_BASE_URL=https://api.deepseek.com，DEEPSEEK_API_KEY=xxx

    deepseek的client代码是有问题的，需要加上

    ```python
    from openai.types.chat import ChatCompletionMessage as _Message
    ```

5. pycharm debug

    ![letta debug.png](../images/letta debug.png)

LETTA_EMBEDDING_MODEL没配置目前能跑简单的demo，后续更新

6. postman demo例子

创建agent

```Text
POST http://localhost:8283/v1/agents

{
    "llm_config": {
        "model": "deepseek-chat",
        "model_endpoint_type": "deepseek",
        "context_window": 16384,
        "model_endpoint": "https://api.deepseek.com/v1",
        "put_inner_thoughts_in_kwargs": true
    },
    "embedding_config": {
        "embedding_model": "glm-4.6",
        "embedding_endpoint_type": "ollama",
        "embedding_dim": 1024
    },
    "memory_blocks": [
        {
            "label": "human",
            "value": "我叫小明，是产品经理"
        },
        {
            "label": "persona",
            "value": "你是 Letta，我的智能助理"
        }
    ]
}
```

修改记忆
```Text
POST http://localhost:8283/v1/agents/{agent-id}/messages
{
    "messages": [
        {
            "role": "user",
            "content": "我不叫小明，我叫阿里巴巴"
        }
    ],
    "stream": false
}
```

修改记忆的流程：

```python
# noinspection PyInconsistentReturns
@router.post(
    "/{agent_id}/messages",
    response_model=LettaResponse,
    operation_id="send_message",
)

async def send_message(
    request_obj: Request,  # FastAPI Request
    agent_id: AgentId,
    server: SyncServer = Depends(get_letta_server), # 拿到单例的letta_server
    request: LettaRequest = Body(...), # 将请求转换成LettaRequest
    headers: HeaderParams = Depends(get_headers),
):
    """
    Process a user message and return the agent's response.
    This endpoint accepts a message from a user and processes it through the agent.
    """
    if len(request.messages) == 0:
        raise ValueError("Messages must not be empty")
    request_start_timestamp_ns = get_utc_timestamp_ns()
    MetricRegistry().user_message_counter.add(1, get_ctx_attributes())

    actor = await server.user_manager.get_actor_or_default_async(actor_id=headers.actor_id)
    # 管理智能体（Agent），包括创建、配置、启动、停止智能体实例等
    agent = await server.agent_manager.get_agent_by_id_async(
        agent_id, actor, include_relationships=["memory", "multi_agent_group", "sources", "tool_exec_environment_variables", "tools"]
    )
    agent_eligible = agent.multi_agent_group is None or agent.multi_agent_group.manager_type in ["sleeptime", "voice_sleeptime"]
    model_compatible = agent.llm_config.model_endpoint_type in [
        "anthropic",
        "openai",
        "together",
        "google_ai",
        "google_vertex",
        "bedrock",
        "ollama",
        "azure",
        "xai",
        "groq",
        "deepseek",
    ]

    # Create a new run for execution tracking
    if settings.track_agent_run:
        runs_manager = RunManager()
        run = await runs_manager.create_run(
            pydantic_run=PydanticRun(
                agent_id=agent_id,
                background=False,
                metadata={
                    "run_type": "send_message",
                },
                request_config=LettaRequestConfig.from_letta_request(request),
            ),
            actor=actor,
        )
    else:
        run = None

    # TODO (cliandy): clean this up
    redis_client = await get_redis_client()
    await redis_client.set(f"{REDIS_RUN_ID_PREFIX}:{agent_id}", run.id if run else None)

    run_update_metadata = None
    try:
        result = None
        if agent_eligible and model_compatible:
            agent_loop = AgentLoop.load(agent_state=agent, actor=actor)
            result = await agent_loop.step(
                request.messages,
                max_steps=request.max_steps,
                run_id=run.id if run else None,
                use_assistant_message=request.use_assistant_message,
                request_start_timestamp_ns=request_start_timestamp_ns,
                include_return_message_types=request.include_return_message_types,
            )
        else:
            result = await server.send_message_to_agent(
                agent_id=agent_id,
                actor=actor,
                input_messages=request.messages,
                stream_steps=False,
                stream_tokens=False,
                # Support for AssistantMessage
                use_assistant_message=request.use_assistant_message,
                assistant_message_tool_name=request.assistant_message_tool_name,
                assistant_message_tool_kwarg=request.assistant_message_tool_kwarg,
                include_return_message_types=request.include_return_message_types,
            )
        run_status = result.stop_reason.stop_reason.run_status
        return result
    except PendingApprovalError as e:
        run_update_metadata = {"error": str(e)}
        run_status = RunStatus.failed
        raise HTTPException(
            status_code=409, detail={"code": "PENDING_APPROVAL", "message": str(e), "pending_request_id": e.pending_request_id}
        )
    except Exception as e:
        run_update_metadata = {"error": str(e)}
        run_status = RunStatus.failed
        raise
    finally:
        if settings.track_agent_run:
            if result:
                stop_reason = result.stop_reason.stop_reason
            else:
                # NOTE: we could also consider this an error?
                stop_reason = None
            await server.run_manager.update_run_by_id_async(
                run_id=run.id,
                update=RunUpdate(
                    status=run_status,
                    metadata=run_update_metadata,
                    stop_reason=stop_reason,
                ),
                actor=actor,
            )

```

不纠结语法，主要的几个点记录下：
1. letta用的是FastAPI，@router.post是FastAPI的路由装饰器，它不会立即执行函数体，只是把
   URL模板 + HTTP 方法 + 函数对象注册到内部的Router.routes列表里，当请求来的时候，会调用send_message函数
2. Depends(get_letta_server)：拿到单例的letta_server
3. request: LettaRequest = Body(...)，将请求转换成LettaRequest

下面会去数据库差相关的信息：
```python
    # 管理智能体（Agent），包括创建、配置、启动、停止智能体实例等
    # memory: The in-context memory of the agent.
    # multi_agent_group: Deprecated
    # sources: The sources used by the agent.
    # tool_exec_environment_variables: Deprecated
    # tools: The tools used by the agent.
    agent = await server.agent_manager.get_agent_by_id_async(
        agent_id, actor, include_relationships=["memory", "multi_agent_group", "sources", "tool_exec_environment_variables", "tools"]
    )
```

agent输出成json格式如下
```json
{
    "created_by_id": "user-00000000-0000-4000-8000-000000000000",
    "last_updated_by_id": "user-00000000-0000-4000-8000-000000000000",
    "created_at": "2025-11-10T15:00:29.257452Z",
    "updated_at": "2025-11-13T05:08:25.879449Z",
    "id": "agent-e14f74eb-9fb7-4cd9-975f-8472d1495ae4",
    "name": "LuxuriousGiraffe",
    "tool_rules": [
        {
            "tool_name": "memory_insert",
            "type": "continue_loop",
            "prompt_template": null
        },
        {
            "tool_name": "send_message",
            "type": "exit_loop",
            "prompt_template": null
        },
        {
            "tool_name": "memory_replace",
            "type": "continue_loop",
            "prompt_template": null
        },
        {
            "tool_name": "conversation_search",
            "type": "continue_loop",
            "prompt_template": null
        }
    ],
    "message_ids": [
        "message-1928573a-4e5f-4f48-af4a-f75e4cc0e445",
        "message-b00fb44d-d491-4dc8-a9a6-2c963d4b710e",
        "message-1433ba65-0bdf-4473-b858-d5493611d81e"
    ],
    "system": "....",
    "agent_type": "memgpt_v2_agent",
    "llm_config": {
        "model": "deepseek-chat"
    },
    "embedding_config": {
        "embedding_endpoint_type": "ollama"
    },
    "response_format": null,
    "description": null,
    "metadata": null,
    "memory": {
        "agent_type": "memgpt_v2_agent",
        "blocks": [
            {
                "value": "你是 Letta，我的智能助理",
                "limit": 20000,
                "project_id": null,
                "template_name": null,
                "is_template": false,
                "template_id": null,
                "base_template_id": null,
                "deployment_id": null,
                "entity_id": null,
                "preserve_on_migration": false,
                "label": "persona",
                "read_only": false,
                "description": "The persona block: Stores details about your current persona, guiding how you behave and respond. This helps you to maintain consistency and personality in your interactions.",
                "metadata": {},
                "hidden": null,
                "id": "block-14b520ea-dcfc-4b89-8cbc-81454de53602",
                "created_by_id": null,
                "last_updated_by_id": null
            },
            {
                "value": "我叫薛大屁，是产品经理",
                "limit": 20000,
                "project_id": null,
                "template_name": null,
                "is_template": false,
                "template_id": null,
                "base_template_id": null,
                "deployment_id": null,
                "entity_id": null,
                "preserve_on_migration": false,
                "label": "human",
                "read_only": false,
                "description": "The human block: Stores key details about the person you are conversing with, allowing for more personalized and friend-like conversation.",
                "metadata": {},
                "hidden": null,
                "id": "block-43c13bd0-7041-4713-aff3-9005ec937593",
                "created_by_id": null,
                "last_updated_by_id": null
            }
        ],
        "file_blocks": [],
        "prompt_template": ""
    },
    "blocks": [
        {
            "value": "你是 Letta，我的智能助理",
            "limit": 20000,
            "project_id": null,
            "template_name": null,
            "is_template": false,
            "template_id": null,
            "base_template_id": null,
            "deployment_id": null,
            "entity_id": null,
            "preserve_on_migration": false,
            "label": "persona",
            "read_only": false,
            "description": "The persona block: Stores details about your current persona, guiding how you behave and respond. This helps you to maintain consistency and personality in your interactions.",
            "metadata": {},
            "hidden": null,
            "id": "block-14b520ea-dcfc-4b89-8cbc-81454de53602",
            "created_by_id": null,
            "last_updated_by_id": null
        },
        {
            "value": "我叫薛大屁，是产品经理",
            "limit": 20000,
            "project_id": null,
            "template_name": null,
            "is_template": false,
            "template_id": null,
            "base_template_id": null,
            "deployment_id": null,
            "entity_id": null,
            "preserve_on_migration": false,
            "label": "human",
            "read_only": false,
            "description": "The human block: Stores key details about the person you are conversing with, allowing for more personalized and friend-like conversation.",
            "metadata": {},
            "hidden": null,
            "id": "block-43c13bd0-7041-4713-aff3-9005ec937593",
            "created_by_id": null,
            "last_updated_by_id": null
        }
    ],
    "tools": [
        {
            "id": "tool-b07b3785-5a85-41a4-9b5b-b74e618ff8da",
            "tool_type": "letta_sleeptime_core",
            "description": "The memory_insert command allows you to insert text at a specific location in a memory block.\n\nExamples:\n    # Update a block containing information about the user (append to the end of the block)\n    memory_insert(label=\"customer\", new_str=\"The customer's ticket number is 12345\")\n\n    # Update a block containing information about the user (insert at the beginning of the block)\n    memory_insert(label=\"customer\", new_str=\"The customer's ticket number is 12345\", insert_line=0)",
            "source_type": "python",
            "name": "memory_insert",
            "tags": [
                "letta_sleeptime_core"
            ],
            "source_code": null,
            "json_schema": {
                "name": "memory_insert",
                "description": "The memory_insert command allows you to insert text at a specific location in a memory block.\n\nExamples:\n    # Update a block containing information about the user (append to the end of the block)\n    memory_insert(label=\"customer\", new_str=\"The customer's ticket number is 12345\")\n\n    # Update a block containing information about the user (insert at the beginning of the block)\n    memory_insert(label=\"customer\", new_str=\"The customer's ticket number is 12345\", insert_line=0)",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "label": {
                            "type": "string",
                            "description": "Section of the memory to be edited, identified by its label."
                        },
                        "new_str": {
                            "type": "string",
                            "description": "The text to insert. Do not include line number prefixes."
                        },
                        "insert_line": {
                            "type": "integer",
                            "description": "The line number after which to insert the text (0 for beginning of file). Defaults to -1 (end of the file)."
                        }
                    },
                    "required": [
                        "label",
                        "new_str"
                    ]
                }
            },
            "args_json_schema": null,
            "return_char_limit": 50000,
            "pip_requirements": null,
            "npm_requirements": null,
            "default_requires_approval": null,
            "enable_parallel_execution": false,
            "created_by_id": "user-00000000-0000-4000-8000-000000000000",
            "last_updated_by_id": "user-00000000-0000-4000-8000-000000000000",
            "metadata_": {}
        },
        {
            "id": "tool-ce03c3dc-d044-46ea-9726-545e38e161b7",
            "tool_type": "letta_sleeptime_core",
            "description": "The memory_replace command allows you to replace a specific string in a memory block with a new string. This is used for making precise edits.\n\nDo NOT attempt to replace long strings, e.g. do not attempt to replace the entire contents of a memory block with a new string.\n\nExamples:\n    # Update a block containing information about the user\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"Their name is Bob\")\n\n    # Update a block containing a todo list\n    memory_replace(label=\"todos\", old_str=\"- [ ] Step 5: Search the web\", new_str=\"- [x] Step 5: Search the web\")\n\n    # Pass an empty string to\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"\")\n\n    # Bad example - do NOT add (view-only) line numbers to the args\n    memory_replace(label=\"human\", old_str=\"1: Their name is Alice\", new_str=\"1: Their name is Bob\")\n\n    # Bad example - do NOT include the line number warning either\n    memory_replace(label=\"human\", old_str=\"# NOTE: Line numbers shown below (with arrows like '1→') are to help during editing. Do NOT include line number prefixes in your memory edit tool calls.\\n1→ Their name is Alice\", new_str=\"1→ Their name is Bob\")\n\n    # Good example - no line numbers or line number warning (they are view-only), just the text\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"Their name is Bob\")",
            "source_type": "python",
            "name": "memory_replace",
            "tags": [
                "letta_sleeptime_core"
            ],
            "source_code": null,
            "json_schema": {
                "name": "memory_replace",
                "description": "The memory_replace command allows you to replace a specific string in a memory block with a new string. This is used for making precise edits.\n\nDo NOT attempt to replace long strings, e.g. do not attempt to replace the entire contents of a memory block with a new string.\n\nExamples:\n    # Update a block containing information about the user\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"Their name is Bob\")\n\n    # Update a block containing a todo list\n    memory_replace(label=\"todos\", old_str=\"- [ ] Step 5: Search the web\", new_str=\"- [x] Step 5: Search the web\")\n\n    # Pass an empty string to\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"\")\n\n    # Bad example - do NOT add (view-only) line numbers to the args\n    memory_replace(label=\"human\", old_str=\"1: Their name is Alice\", new_str=\"1: Their name is Bob\")\n\n    # Bad example - do NOT include the line number warning either\n    memory_replace(label=\"human\", old_str=\"# NOTE: Line numbers shown below (with arrows like '1→') are to help during editing. Do NOT include line number prefixes in your memory edit tool calls.\\n1→ Their name is Alice\", new_str=\"1→ Their name is Bob\")\n\n    # Good example - no line numbers or line number warning (they are view-only), just the text\n    memory_replace(label=\"human\", old_str=\"Their name is Alice\", new_str=\"Their name is Bob\")",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "label": {
                            "type": "string",
                            "description": "Section of the memory to be edited, identified by its label."
                        },
                        "old_str": {
                            "type": "string",
                            "description": "The text to replace (must match exactly, including whitespace and indentation)."
                        },
                        "new_str": {
                            "type": "string",
                            "description": "The new text to insert in place of the old text. Do not include line number prefixes."
                        }
                    },
                    "required": [
                        "label",
                        "old_str",
                        "new_str"
                    ]
                }
            },
            "args_json_schema": null,
            "return_char_limit": 50000,
            "pip_requirements": null,
            "npm_requirements": null,
            "default_requires_approval": null,
            "enable_parallel_execution": false,
            "created_by_id": "user-00000000-0000-4000-8000-000000000000",
            "last_updated_by_id": "user-00000000-0000-4000-8000-000000000000",
            "metadata_": {}
        },
        {
            "id": "tool-fdecc4fc-f087-43c0-9a58-b18d592cf66f",
            "tool_type": "letta_core",
            "description": "Sends a message to the human user.",
            "source_type": "python",
            "name": "send_message",
            "tags": [
                "letta_core"
            ],
            "source_code": null,
            "json_schema": {
                "name": "send_message",
                "description": "Sends a message to the human user.",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "message": {
                            "type": "string",
                            "description": "Message contents. All unicode (including emojis) are supported."
                        }
                    },
                    "required": [
                        "message"
                    ]
                }
            },
            "args_json_schema": null,
            "return_char_limit": 50000,
            "pip_requirements": null,
            "npm_requirements": null,
            "default_requires_approval": null,
            "enable_parallel_execution": false,
            "created_by_id": "user-00000000-0000-4000-8000-000000000000",
            "last_updated_by_id": "user-00000000-0000-4000-8000-000000000000",
            "metadata_": {}
        },
        {
            "id": "tool-52621969-2d65-405b-8e1d-88e20bbe4332",
            "tool_type": "letta_core",
            "description": "Search prior conversation history using hybrid search (text + semantic similarity).\n\nExamples:\n    # Search all messages\n    conversation_search(query=\"project updates\")\n\n    # Search only assistant messages\n    conversation_search(query=\"error handling\", roles=[\"assistant\"])\n\n    # Search with date range (inclusive of both dates)\n    conversation_search(query=\"meetings\", start_date=\"2024-01-15\", end_date=\"2024-01-20\")\n    # This includes all messages from Jan 15 00:00:00 through Jan 20 23:59:59\n\n    # Search messages from a specific day (inclusive)\n    conversation_search(query=\"bug reports\", start_date=\"2024-09-04\", end_date=\"2024-09-04\")\n    # This includes ALL messages from September 4, 2024\n\n    # Search with specific time boundaries\n    conversation_search(query=\"deployment\", start_date=\"2024-01-15T09:00\", end_date=\"2024-01-15T17:30\")\n    # This includes messages from 9 AM to 5:30 PM on Jan 15\n\n    # Search with limit\n    conversation_search(query=\"debugging\", limit=10)",
            "source_type": "python",
            "name": "conversation_search",
            "tags": [
                "letta_core"
            ],
            "source_code": null,
            "json_schema": {
                "name": "conversation_search",
                "description": "Search prior conversation history using hybrid search (text + semantic similarity).\n\nExamples:\n    # Search all messages\n    conversation_search(query=\"project updates\")\n\n    # Search only assistant messages\n    conversation_search(query=\"error handling\", roles=[\"assistant\"])\n\n    # Search with date range (inclusive of both dates)\n    conversation_search(query=\"meetings\", start_date=\"2024-01-15\", end_date=\"2024-01-20\")\n    # This includes all messages from Jan 15 00:00:00 through Jan 20 23:59:59\n\n    # Search messages from a specific day (inclusive)\n    conversation_search(query=\"bug reports\", start_date=\"2024-09-04\", end_date=\"2024-09-04\")\n    # This includes ALL messages from September 4, 2024\n\n    # Search with specific time boundaries\n    conversation_search(query=\"deployment\", start_date=\"2024-01-15T09:00\", end_date=\"2024-01-15T17:30\")\n    # This includes messages from 9 AM to 5:30 PM on Jan 15\n\n    # Search with limit\n    conversation_search(query=\"debugging\", limit=10)",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "String to search for using both text matching and semantic similarity."
                        },
                        "roles": {
                            "type": "array",
                            "items": {
                                "type": "string",
                                "enum": [
                                    "assistant",
                                    "user",
                                    "tool"
                                ]
                            },
                            "description": "Optional list of message roles to filter by."
                        },
                        "limit": {
                            "type": "integer",
                            "description": "Maximum number of results to return. Uses system default if not specified."
                        },
                        "start_date": {
                            "type": "string",
                            "description": "Filter results to messages created on or after this date (INCLUSIVE). When using date-only format (e.g., \"2024-01-15\"), includes messages starting from 00:00:00 of that day. ISO 8601 format: \"YYYY-MM-DD\" or \"YYYY-MM-DDTHH:MM\". Examples: \"2024-01-15\" (from start of Jan 15), \"2024-01-15T14:30\" (from 2:30 PM on Jan 15)."
                        },
                        "end_date": {
                            "type": "string",
                            "description": "Filter results to messages created on or before this date (INCLUSIVE). When using date-only format (e.g., \"2024-01-20\"), includes all messages from that entire day. ISO 8601 format: \"YYYY-MM-DD\" or \"YYYY-MM-DDTHH:MM\". Examples: \"2024-01-20\" (includes all of Jan 20), \"2024-01-20T17:00\" (up to 5 PM on Jan 20)."
                        }
                    },
                    "required": [
                        "query"
                    ]
                }
            },
            "args_json_schema": null,
            "return_char_limit": 50000,
            "pip_requirements": null,
            "npm_requirements": null,
            "default_requires_approval": null,
            "enable_parallel_execution": true,
            "created_by_id": "user-00000000-0000-4000-8000-000000000000",
            "last_updated_by_id": "user-00000000-0000-4000-8000-000000000000",
            "metadata_": {}
        }
    ],
    "sources": [],
    "tags": [],
    "tool_exec_environment_variables": [],
    "secrets": [],
    "project_id": null,
    "template_id": null,
    "base_template_id": null,
    "deployment_id": null,
    "entity_id": null,
    "identity_ids": [],
    "identities": [],
    "message_buffer_autoclear": false,
    "enable_sleeptime": null,
    "multi_agent_group": null,
    "managed_group": null,
    "last_run_completion": null,
    "last_run_duration_ms": null,
    "timezone": "UTC",
    "max_files_open": 5,
    "per_file_view_window_char_limit": 15000,
    "hidden": null
}
```