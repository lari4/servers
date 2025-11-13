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

