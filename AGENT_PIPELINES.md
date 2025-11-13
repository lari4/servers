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

