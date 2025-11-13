# Документация по пайплайнам и схемам работы агентов MCP Servers

Этот документ описывает все возможные пайплайны работы агентов, последовательность вызовов промтов и инструментов, а также передачу данных между этапами.

## Содержание
- [1. Everything Server Pipelines](#1-everything-server-pipelines)
- [2. Fetch Server Pipelines](#2-fetch-server-pipelines)
- [3. Sequential Thinking Server Pipelines](#3-sequential-thinking-server-pipelines)
- [4. Memory Server Pipelines](#4-memory-server-pipelines)
- [5. Git Server Pipelines](#5-git-server-pipelines)
- [6. Общие паттерны взаимодействия](#6-общие-паттерны-взаимодействия)

---

## 1. Everything Server Pipelines

Everything Server демонстрирует все возможности MCP протокола через различные пайплайны.

**Расположение**: `src/everything/everything.ts`

### 1.1. LLM Sampling Pipeline (Пайплайн запроса к LLM)

**Назначение**: Демонстрация возможности MCP сервера запрашивать генерацию текста у LLM через клиента.

**Триггер**: Вызов инструмента `sampleLLM`

**Схема потока данных**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    LLM Sampling Pipeline                         │
└─────────────────────────────────────────────────────────────────┘

1. AI Agent вызывает tool
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool: sampleLLM                                                   │
│ Input: { prompt: string, maxTokens: number }                     │
└──────────────────────────────────────────────────────────────────┘
   ↓
2. Сервер формирует запрос к клиенту
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Server → Client: sampling/createMessage                          │
│ {                                                                 │
│   method: "sampling/createMessage",                              │
│   params: {                                                       │
│     messages: [                                                   │
│       {                                                           │
│         role: "user",                                             │
│         content: {                                                │
│           type: "text",                                           │
│           text: "Resource <uri> context: <prompt>"               │
│         }                                                         │
│       }                                                           │
│     ],                                                            │
│     systemPrompt: "You are a helpful test server.",              │
│     maxTokens: <maxTokens>,                                       │
│     temperature: 0.7,                                             │
│     includeContext: "thisServer"                                  │
│   }                                                               │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
   ↓
3. Клиент вызывает LLM и возвращает результат
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Client → Server: CreateMessageResult                             │
│ {                                                                 │
│   content: {                                                      │
│     text: "<generated_text>"                                      │
│   }                                                               │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
   ↓
4. Сервер возвращает результат агенту
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool Response                                                     │
│ {                                                                 │
│   content: [                                                      │
│     {                                                             │
│       type: "text",                                               │
│       text: "LLM sampling result: <generated_text>"              │
│     }                                                             │
│   ]                                                               │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Код реализации**:
```typescript
// 1. Определение инструмента
{
  name: ToolName.SAMPLE_LLM,
  description: "Samples from an LLM using MCP's sampling feature",
  inputSchema: zodToJsonSchema(SampleLLMSchema) as ToolInput,
}

// 2. Вспомогательная функция для запроса семплирования
const requestSampling = async (
  context: string,
  uri: string,
  maxTokens: number = 100,
  sendRequest: SendRequest
) => {
  const request: CreateMessageRequest = {
    method: "sampling/createMessage",
    params: {
      messages: [
        {
          role: "user",
          content: {
            type: "text",
            text: `Resource ${uri} context: ${context}`,
          },
        },
      ],
      systemPrompt: "You are a helpful test server.",
      maxTokens,
      temperature: 0.7,
      includeContext: "thisServer",
    },
  };

  return await sendRequest(request, CreateMessageResultSchema);
};

// 3. Обработчик инструмента
if (name === ToolName.SAMPLE_LLM) {
  const validatedArgs = SampleLLMSchema.parse(args);
  const { prompt, maxTokens } = validatedArgs;

  const result = await requestSampling(
    prompt,
    ToolName.SAMPLE_LLM,
    maxTokens,
    extra.sendRequest
  );
  return {
    content: [
      { type: "text", text: `LLM sampling result: ${result.content.text}` },
    ],
  };
}
```

**Параметры**:
- **Входные**: `prompt` (string), `maxTokens` (number, default: 100)
- **Промежуточные**: системный промт, температура 0.7, includeContext
- **Выходные**: сгенерированный текст от LLM

---

### 1.2. Elicitation Pipeline (Пайплайн запроса данных у пользователя)

**Назначение**: Демонстрация возможности MCP сервера запрашивать структурированные данные у пользователя через диалоговое окно.

**Триггер**: Вызов инструмента `startElicitation`

**Схема потока данных**:
```
┌─────────────────────────────────────────────────────────────────┐
│                   Elicitation Pipeline                           │
└─────────────────────────────────────────────────────────────────┘

1. AI Agent вызывает tool
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool: startElicitation                                            │
│ Input: {} (no parameters)                                        │
└──────────────────────────────────────────────────────────────────┘
   ↓
2. Сервер создает запрос на elicitation
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Server → Client: elicitation/create                              │
│ {                                                                 │
│   method: "elicitation/create",                                  │
│   params: {                                                       │
│     message: "Please provide inputs for the following fields:",  │
│     requestedSchema: {                                            │
│       type: "object",                                             │
│       properties: {                                               │
│         name: {                                                   │
│           title: "Full Name",                                     │
│           type: "string",                                         │
│           description: "Your full, legal name"                    │
│         },                                                        │
│         check: {                                                  │
│           title: "Agree to terms",                                │
│           type: "boolean",                                        │
│           description: "A boolean check"                          │
│         },                                                        │
│         color: {                                                  │
│           title: "Favorite Color",                                │
│           type: "string",                                         │
│           description: "Favorite color (open text)",              │
│           default: "blue"                                         │
│         },                                                        │
│         email: {                                                  │
│           title: "Email Address",                                 │
│           type: "string",                                         │
│           format: "email"                                         │
│         },                                                        │
│         homepage: {                                               │
│           type: "string",                                         │
│           format: "uri"                                           │
│         },                                                        │
│         birthdate: {                                              │
│           type: "string",                                         │
│           format: "date"                                          │
│         },                                                        │
│         integer: {                                                │
│           type: "integer",                                        │
│           minimum: 1,                                             │
│           maximum: 100,                                           │
│           default: 42                                             │
│         },                                                        │
│         number: {                                                 │
│           type: "number",                                         │
│           minimum: 0,                                             │
│           maximum: 1000,                                          │
│           default: 3.14                                           │
│         },                                                        │
│         petType: {                                                │
│           type: "string",                                         │
│           enum: ["cats", "dogs", "birds", "fish", "reptiles"],   │
│           enumNames: ["Cats", "Dogs", "Birds", "Fish", ...]      │
│         }                                                         │
│       },                                                          │
│       required: ["name"]                                          │
│     }                                                             │
│   },                                                              │
│   timeout: 600000  // 10 minutes                                 │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
   ↓
3. Клиент показывает UI форму пользователю
   ↓
4. Пользователь заполняет форму или отклоняет/отменяет
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Client → Server: ElicitResult                                    │
│                                                                   │
│ Вариант A (accept):                                              │
│ {                                                                 │
│   action: "accept",                                               │
│   content: {                                                      │
│     name: "John Doe",                                             │
│     check: true,                                                  │
│     color: "blue",                                                │
│     email: "john@example.com",                                    │
│     homepage: "https://example.com",                              │
│     birthdate: "1990-01-15",                                      │
│     integer: 42,                                                  │
│     number: 3.14,                                                 │
│     petType: "dogs"                                               │
│   }                                                               │
│ }                                                                 │
│                                                                   │
│ Вариант B (decline):                                             │
│ {                                                                 │
│   action: "decline"                                               │
│ }                                                                 │
│                                                                   │
│ Вариант C (cancel):                                              │
│ {                                                                 │
│   action: "cancel"                                                │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
   ↓
5. Сервер обрабатывает результат
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool Response                                                     │
│ {                                                                 │
│   content: [                                                      │
│     {                                                             │
│       type: "text",                                               │
│       text: "✅ User provided the requested information!"         │
│     },                                                            │
│     {                                                             │
│       type: "text",                                               │
│       text: "User inputs:\n- Name: John Doe\n- ..."              │
│     },                                                            │
│     {                                                             │
│       type: "text",                                               │
│       text: "Raw result: {...}"                                   │
│     }                                                             │
│   ]                                                               │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Код реализации**:
```typescript
// 1. Определение инструмента
{
  name: ToolName.ELICITATION,
  description: "Elicitation test tool that demonstrates how to request user input with various field types",
  inputSchema: zodToJsonSchema(ElicitationSchema) as ToolInput,
}

// 2. Обработчик инструмента
if (name === ToolName.ELICITATION) {
  ElicitationSchema.parse(args);

  const elicitationResult = await extra.sendRequest({
    method: 'elicitation/create',
    params: {
      message: 'Please provide inputs for the following fields:',
      requestedSchema: {
        type: 'object',
        properties: {
          name: {
            title: 'Full Name',
            type: 'string',
            description: 'Your full, legal name',
          },
          check: {
            title: 'Agree to terms',
            type: 'boolean',
            description: 'A boolean check',
          },
          // ... остальные поля
          petType: {
            title: 'Pet type',
            type: 'string',
            enum: ['cats', 'dogs', 'birds', 'fish', 'reptiles'],
            enumNames: ['Cats', 'Dogs', 'Birds', 'Fish', 'Reptiles'],
            default: 'dogs',
            description: 'Your favorite pet type',
          },
        },
        required: ['name'],
      },
    },
  }, ElicitResultSchema, { timeout: 10 * 60 * 1000 /* 10 minutes */ });

  // Обработка результата
  const content = [];

  if (elicitationResult.action === 'accept' && elicitationResult.content) {
    content.push({
      type: "text",
      text: `✅ User provided the requested information!`,
    });

    const userData = elicitationResult.content;
    const lines = [];
    if (userData.name) lines.push(`- Name: ${userData.name}`);
    if (userData.check !== undefined) lines.push(`- Agreed to terms: ${userData.check}`);
    // ... обработка всех полей

    content.push({
      type: "text",
      text: `User inputs:\n${lines.join('\n')}`,
    });
  } else if (elicitationResult.action === 'decline') {
    content.push({
      type: "text",
      text: `❌ User declined to provide the requested information.`,
    });
  } else if (elicitationResult.action === 'cancel') {
    content.push({
      type: "text",
      text: `⚠️ User cancelled the elicitation dialog.`,
    });
  }

  content.push({
    type: "text",
    text: `\nRaw result: ${JSON.stringify(elicitationResult, null, 2)}`,
  });

  return { content };
}
```

**Поддерживаемые типы полей**:
- `string` - текстовое поле
- `boolean` - чекбокс
- `email` (string + format) - email поле с валидацией
- `uri` (string + format) - URL поле с валидацией
- `date` (string + format) - дата с date picker
- `integer` - целое число с min/max
- `number` - число с плавающей точкой с min/max
- `enum` - выпадающий список с предопределенными значениями

**Параметры**:
- **Входные**: Нет параметров
- **Конфигурация схемы**: JSON Schema для формы с полями разных типов
- **Timeout**: 10 минут для заполнения формы
- **Выходные**: Структурированные данные от пользователя или статус отказа/отмены

**Возможные действия пользователя**:
1. **accept** - заполнил и отправил форму
2. **decline** - отклонил запрос
3. **cancel** - отменил диалог

---

## 2. Fetch Server Pipelines

Fetch Server предоставляет функциональность для загрузки веб-контента и его конвертации в markdown.

**Расположение**: `src/fetch/src/mcp_server_fetch/server.py`

### 2.1. Autonomous Fetch Pipeline (Автоматическая загрузка через инструмент)

**Назначение**: Автоматическая загрузка веб-страницы с проверкой robots.txt для автономных запросов.

**Триггер**: Вызов инструмента `fetch`

**Схема потока данных**:
```
┌─────────────────────────────────────────────────────────────────┐
│               Autonomous Fetch Pipeline                          │
└─────────────────────────────────────────────────────────────────┘

1. AI Agent вызывает tool
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool: fetch                                                       │
│ Input: {                                                          │
│   url: string,                                                    │
│   max_length: number (default: 5000),                            │
│   start_index: number (default: 0),                              │
│   raw: boolean (default: false)                                  │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
   ↓
2. Сервер проверяет robots.txt (если не ignore_robots_txt)
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ check_may_autonomously_fetch_url()                               │
│ GET https://example.com/robots.txt                               │
│ User-Agent: ModelContextProtocol/1.0 (Autonomous; ...)          │
│                                                                   │
│ Проверяет:                                                        │
│ - Доступность robots.txt (игнорирует 404)                        │
│ - Правила для user-agent                                         │
│ - Разрешен ли доступ к URL                                       │
└──────────────────────────────────────────────────────────────────┘
   ↓
3a. Если robots.txt запрещает
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ McpError                                                          │
│ "The sites robots.txt specifies that autonomous fetching         │
│  of this page is not allowed"                                    │
│                                                                   │
│ Рекомендация: использовать fetch prompt для ручной загрузки     │
└──────────────────────────────────────────────────────────────────┘

3b. Если robots.txt разрешает или отсутствует
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ fetch_url()                                                       │
│ GET <url>                                                         │
│ User-Agent: ModelContextProtocol/1.0 (Autonomous; ...)          │
│ Timeout: 30 seconds                                               │
│ Follow redirects: Yes                                             │
└──────────────────────────────────────────────────────────────────┘
   ↓
4. Анализ content-type и конвертация
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Content-Type Analysis                                             │
│                                                                   │
│ If HTML && !raw:                                                  │
│   ├─> readabilipy.simple_json_from_html_string()                │
│   │    (упрощение HTML)                                          │
│   └─> markdownify.markdownify()                                  │
│        (конвертация в Markdown)                                   │
│                                                                   │
│ Else:                                                             │
│   └─> return raw content                                          │
└──────────────────────────────────────────────────────────────────┘
   ↓
5. Обработка длины и truncation
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Length Processing                                                 │
│                                                                   │
│ content = content[start_index : start_index + max_length]       │
│                                                                   │
│ If truncated && has more content:                                │
│   └─> Add message: "Content truncated. Call fetch with          │
│        start_index=N to get more content."                        │
└──────────────────────────────────────────────────────────────────┘
   ↓
6. Возврат результата
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Tool Response                                                     │
│ {                                                                 │
│   type: "text",                                                   │
│   text: "Contents of <url>:\n<markdown_content>"                 │
│ }                                                                 │
│                                                                   │
│ Или (если truncated):                                            │
│ {                                                                 │
│   type: "text",                                                   │
│   text: "Contents of <url>:\n<partial_content>\n\n              │
│          <error>Content truncated. Call the fetch tool with      │
│          a start_index of 5000 to get more content.</error>"     │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Код реализации**:
```python
@server.call_tool()
async def call_tool(name, arguments: dict) -> list[TextContent]:
    try:
        args = Fetch(**arguments)
    except ValueError as e:
        raise McpError(ErrorData(code=INVALID_PARAMS, message=str(e)))

    url = str(args.url)
    if not url:
        raise McpError(ErrorData(code=INVALID_PARAMS, message="URL is required"))

    # Проверка robots.txt для автономных запросов
    if not ignore_robots_txt:
        await check_may_autonomously_fetch_url(url, user_agent_autonomous, proxy_url)

    # Загрузка контента
    content, prefix = await fetch_url(
        url, user_agent_autonomous, force_raw=args.raw, proxy_url=proxy_url
    )

    # Обработка длины
    original_length = len(content)
    if args.start_index >= original_length:
        content = "<error>No more content available.</error>"
    else:
        truncated_content = content[args.start_index : args.start_index + args.max_length]
        if not truncated_content:
            content = "<error>No more content available.</error>"
        else:
            content = truncated_content
            actual_content_length = len(truncated_content)
            remaining_content = original_length - (args.start_index + actual_content_length)

            if actual_content_length == args.max_length and remaining_content > 0:
                next_start = args.start_index + actual_content_length
                content += f"\n\n<error>Content truncated. Call the fetch tool with a start_index of {next_start} to get more content.</error>"

    return [TextContent(type="text", text=f"{prefix}Contents of {url}:\n{content}")]
```

**Параметры**:
- **url**: URL для загрузки
- **max_length**: Максимальная длина возвращаемого контента (default: 5000)
- **start_index**: Начальная позиция для продолжения (default: 0)
- **raw**: Вернуть сырой HTML вместо markdown (default: false)

**User-Agent**: `ModelContextProtocol/1.0 (Autonomous; +https://github.com/modelcontextprotocol/servers)`

**Особенности**:
- Проверка robots.txt обязательна (если не отключена)
- Автоматическая конвертация HTML в Markdown
- Поддержка пагинации через start_index
- Обработка redirects

---

### 2.2. Manual Fetch Pipeline (Ручная загрузка через промт)

**Назначение**: Ручная загрузка веб-страницы пользователем, без проверки robots.txt.

**Триггер**: Использование промта `fetch`

**Схема потока данных**:
```
┌─────────────────────────────────────────────────────────────────┐
│                Manual Fetch Pipeline                             │
└─────────────────────────────────────────────────────────────────┘

1. User использует prompt
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ Prompt: fetch                                                     │
│ Arguments: { url: string }                                       │
└──────────────────────────────────────────────────────────────────┘
   ↓
2. Сервер загружает URL (БЕЗ проверки robots.txt)
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ fetch_url()                                                       │
│ GET <url>                                                         │
│ User-Agent: ModelContextProtocol/1.0 (User-Specified; ...)      │
│ Timeout: 30 seconds                                               │
│ Follow redirects: Yes                                             │
│                                                                   │
│ NO robots.txt check!                                              │
└──────────────────────────────────────────────────────────────────┘
   ↓
3. Конвертация контента (аналогично autonomous)
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ HTML → Markdown conversion                                        │
│ (если контент HTML)                                               │
└──────────────────────────────────────────────────────────────────┘
   ↓
4. Возврат результата как промт
   ↓
┌──────────────────────────────────────────────────────────────────┐
│ GetPromptResult                                                   │
│ {                                                                 │
│   description: "Contents of <url>",                              │
│   messages: [                                                     │
│     {                                                             │
│       role: "user",                                               │
│       content: {                                                  │
│         type: "text",                                             │
│         text: "<prefix><markdown_content>"                        │
│       }                                                           │
│     }                                                             │
│   ]                                                               │
│ }                                                                 │
│                                                                   │
│ Или (если ошибка):                                               │
│ {                                                                 │
│   description: "Failed to fetch <url>",                          │
│   messages: [                                                     │
│     {                                                             │
│       role: "user",                                               │
│       content: {                                                  │
│         type: "text",                                             │
│         text: "<error_message>"                                   │
│       }                                                           │
│     }                                                             │
│   ]                                                               │
│ }                                                                 │
└──────────────────────────────────────────────────────────────────┘
```

**Код реализации**:
```python
@server.get_prompt()
async def get_prompt(name: str, arguments: dict | None) -> GetPromptResult:
    if not arguments or "url" not in arguments:
        raise McpError(ErrorData(code=INVALID_PARAMS, message="URL is required"))

    url = arguments["url"]

    try:
        # Использует user_agent_manual (без проверки robots.txt)
        content, prefix = await fetch_url(url, user_agent_manual, proxy_url=proxy_url)
    except McpError as e:
        return GetPromptResult(
            description=f"Failed to fetch {url}",
            messages=[
                PromptMessage(
                    role="user",
                    content=TextContent(type="text", text=str(e)),
                )
            ],
        )

    return GetPromptResult(
        description=f"Contents of {url}",
        messages=[
            PromptMessage(
                role="user",
                content=TextContent(type="text", text=prefix + content)
            )
        ],
    )
```

**Параметры**:
- **url**: URL для загрузки (обязательный)

**User-Agent**: `ModelContextProtocol/1.0 (User-Specified; +https://github.com/modelcontextprotocol/servers)`

**Отличия от Autonomous Fetch**:
- ❌ НЕ проверяет robots.txt
- ❌ НЕ поддерживает пагинацию (start_index/max_length)
- ✅ Возвращает весь контент сразу
- ✅ Используется когда robots.txt запрещает автономную загрузку

**Когда использовать**:
- Когда autonomous fetch заблокирован robots.txt
- Когда пользователь явно хочет загрузить контент
- Для одноразовой загрузки небольших страниц

---

