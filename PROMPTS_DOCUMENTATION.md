# Документация по AI Промтам MCP Servers

Этот документ содержит полное описание всех промтов, используемых в приложении Model Context Protocol (MCP) Servers.

## Содержание
- [1. Промты Everything Server](#1-промты-everything-server)
- [2. Промты Fetch Server](#2-промты-fetch-server)
- [3. Серверы без промтов](#3-серверы-без-промтов)

---

## 1. Промты Everything Server

Everything Server - это комплексный демонстрационный сервер, который показывает все возможности MCP, включая различные типы промтов.

**Расположение**: `src/everything/everything.ts` (строки 347-463)

### 1.1. Simple Prompt (Простой промт)

**Название**: `simple_prompt`

**Описание**: Базовый промт без параметров, демонстрирующий простейшую форму промта в MCP.

**Назначение**:
- Демонстрация минималистичного промта без аргументов
- Пример статического текстового промта
- Базовый шаблон для начала работы с промтами

**Параметры**: Нет

**Код промта**:
```typescript
// Регистрация промта в списке доступных промтов
{
  name: PromptName.SIMPLE,
  description: "A prompt without arguments",
}

// Реализация обработчика промта
if (name === PromptName.SIMPLE) {
  return {
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: "This is a simple prompt without arguments.",
        },
      },
    ],
  };
}
```

**Возвращаемый формат**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "This is a simple prompt without arguments."
      }
    }
  ]
}
```

---

### 1.2. Complex Prompt (Сложный промт с параметрами)

**Название**: `complex_prompt`

**Описание**: Промт с параметрами, демонстрирующий многоэтапный диалог с аргументами и мультимодальным контентом.

**Назначение**:
- Демонстрация промта с обязательными и опциональными аргументами
- Пример многоэтапной беседы (user -> assistant -> user)
- Включение изображений в промт
- Использование параметров для настройки поведения промта

**Параметры**:
- `temperature` (обязательный) - настройка температуры для генерации
- `style` (опциональный) - стиль вывода

**Поддержка автодополнения**: Да, через `CompleteRequestSchema` для аргументов `temperature` и `style`

**Код промта**:
```typescript
// Регистрация промта с аргументами
{
  name: PromptName.COMPLEX,
  description: "A prompt with arguments",
  arguments: [
    {
      name: "temperature",
      description: "Temperature setting",
      required: true,
    },
    {
      name: "style",
      description: "Output style",
      required: false,
    },
  ],
}

// Реализация обработчика промта
if (name === PromptName.COMPLEX) {
  return {
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: `This is a complex prompt with arguments: temperature=${args?.temperature}, style=${args?.style}`,
        },
      },
      {
        role: "assistant",
        content: {
          type: "text",
          text: "I understand. You've provided a complex prompt with temperature and style arguments. How would you like me to proceed?",
        },
      },
      {
        role: "user",
        content: {
          type: "image",
          data: MCP_TINY_IMAGE,
          mimeType: "image/png",
        },
      },
    ],
  };
}
```

**Автодополнение аргументов**:
```typescript
const EXAMPLE_COMPLETIONS = {
  style: ["casual", "formal", "technical", "friendly"],
  temperature: ["0", "0.5", "0.7", "1.0"],
};
```

**Возвращаемый формат**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "This is a complex prompt with arguments: temperature=0.7, style=formal"
      }
    },
    {
      "role": "assistant",
      "content": {
        "type": "text",
        "text": "I understand. You've provided a complex prompt with temperature and style arguments. How would you like me to proceed?"
      }
    },
    {
      "role": "user",
      "content": {
        "type": "image",
        "data": "<base64_image_data>",
        "mimeType": "image/png"
      }
    }
  ]
}
```

---

### 1.3. Resource Prompt (Промт со встроенным ресурсом)

**Название**: `resource_prompt`

**Описание**: Промт, который включает встроенную ссылку на ресурс из системы ресурсов сервера.

**Назначение**:
- Демонстрация встраивания ресурсов в промты
- Пример работы с resource references
- Показ интеграции между промтами и ресурсами
- Валидация параметров ресурсов

**Параметры**:
- `resourceId` (обязательный) - ID ресурса для включения (1-100)

**Валидация**: Проверка диапазона resourceId (1-100) и существования ресурса

**Код промта**:
```typescript
// Регистрация промта с параметром ресурса
{
  name: PromptName.RESOURCE,
  description: "A prompt that includes an embedded resource reference",
  arguments: [
    {
      name: "resourceId",
      description: "Resource ID to include (1-100)",
      required: true,
    },
  ],
}

// Реализация обработчика промта с валидацией
if (name === PromptName.RESOURCE) {
  const resourceId = parseInt(args?.resourceId as string, 10);
  if (isNaN(resourceId) || resourceId < 1 || resourceId > 100) {
    throw new Error(
      `Invalid resourceId: ${args?.resourceId}. Must be a number between 1 and 100.`
    );
  }

  const resourceIndex = resourceId - 1;
  const resource = ALL_RESOURCES[resourceIndex];

  return {
    messages: [
      {
        role: "user",
        content: {
          type: "text",
          text: `This prompt includes Resource ${resourceId}. Please analyze the following resource:`,
        },
      },
      {
        role: "user",
        content: {
          type: "resource",
          resource: resource,
        },
      },
    ],
  };
}
```

**Структура ресурсов**:
```typescript
// Ресурсы создаются динамически (100 ресурсов)
const ALL_RESOURCES: Resource[] = Array.from({ length: 100 }, (_, i) => {
  const uri = `test://static/resource/${i + 1}`;
  if (i % 2 === 0) {
    // Четные - текстовые ресурсы
    return {
      uri,
      name: `Resource ${i + 1}`,
      mimeType: "text/plain",
      text: `Resource ${i + 1}: This is a plaintext resource`,
    };
  } else {
    // Нечетные - бинарные ресурсы
    const buffer = Buffer.from(`Resource ${i + 1}: This is a base64 blob`);
    return {
      uri,
      name: `Resource ${i + 1}`,
      mimeType: "application/octet-stream",
      blob: buffer.toString("base64"),
    };
  }
});
```

**Возвращаемый формат**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "This prompt includes Resource 5. Please analyze the following resource:"
      }
    },
    {
      "role": "user",
      "content": {
        "type": "resource",
        "resource": {
          "uri": "test://static/resource/5",
          "name": "Resource 5",
          "mimeType": "application/octet-stream",
          "blob": "<base64_data>"
        }
      }
    }
  ]
}
```

---

## 2. Промты Fetch Server

Fetch Server предоставляет функциональность для загрузки веб-контента и его конвертации в формат markdown.

**Расположение**: `src/fetch/src/mcp_server_fetch/server.py` (строки 209-284)

### 2.1. Fetch Prompt (Промт для загрузки веб-страниц)

**Название**: `fetch`

**Описание**: Промт для загрузки URL-адреса и извлечения его содержимого в формате markdown.

**Назначение**:
- Загрузка веб-контента по URL
- Автоматическая конвертация HTML в markdown
- Упрощение контента для анализа AI
- Ручная загрузка контента (в отличие от автоматической через инструмент)

**Параметры**:
- `url` (обязательный) - URL для загрузки

**Особенности**:
- Использует user-agent для ручных запросов (не проверяет robots.txt)
- Автоматически упрощает HTML контент
- Обрабатывает ошибки загрузки

**Код промта**:
```python
@server.list_prompts()
async def list_prompts() -> list[Prompt]:
    return [
        Prompt(
            name="fetch",
            description="Fetch a URL and extract its contents as markdown",
            arguments=[
                PromptArgument(
                    name="url", description="URL to fetch", required=True
                )
            ],
        )
    ]

@server.get_prompt()
async def get_prompt(name: str, arguments: dict | None) -> GetPromptResult:
    if not arguments or "url" not in arguments:
        raise McpError(ErrorData(code=INVALID_PARAMS, message="URL is required"))

    url = arguments["url"]

    try:
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
                role="user", content=TextContent(type="text", text=prefix + content)
            )
        ],
    )
```

**Функция загрузки контента**:
```python
async def fetch_url(
    url: str, user_agent: str, force_raw: bool = False, proxy_url: str | None = None
) -> Tuple[str, str]:
    """
    Fetch the URL and return the content in a form ready for the LLM,
    as well as a prefix string with status information.
    """
    async with AsyncClient(proxies=proxy_url) as client:
        try:
            response = await client.get(
                url,
                follow_redirects=True,
                headers={"User-Agent": user_agent},
                timeout=30,
            )
        except HTTPError as e:
            raise McpError(ErrorData(code=INTERNAL_ERROR, message=f"Failed to fetch {url}: {e!r}"))
        if response.status_code >= 400:
            raise McpError(ErrorData(
                code=INTERNAL_ERROR,
                message=f"Failed to fetch {url} - status code {response.status_code}",
            ))

        page_raw = response.text

    content_type = response.headers.get("content-type", "")
    is_page_html = (
        "<html" in page_raw[:100] or "text/html" in content_type or not content_type
    )

    if is_page_html and not force_raw:
        return extract_content_from_html(page_raw), ""

    return (
        page_raw,
        f"Content type {content_type} cannot be simplified to markdown, but here is the raw content:\n",
    )
```

**Конвертация HTML в Markdown**:
```python
def extract_content_from_html(html: str) -> str:
    """Extract and convert HTML content to Markdown format."""
    ret = readabilipy.simple_json.simple_json_from_html_string(
        html, use_readability=True
    )
    if not ret["content"]:
        return "<error>Page failed to be simplified from HTML</error>"
    content = markdownify.markdownify(
        ret["content"],
        heading_style=markdownify.ATX,
    )
    return content
```

**User-Agent константы**:
```python
DEFAULT_USER_AGENT_AUTONOMOUS = "ModelContextProtocol/1.0 (Autonomous; +https://github.com/modelcontextprotocol/servers)"
DEFAULT_USER_AGENT_MANUAL = "ModelContextProtocol/1.0 (User-Specified; +https://github.com/modelcontextprotocol/servers)"
```

**Возвращаемый формат (успешный)**:
```json
{
  "description": "Contents of https://example.com",
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "# Example Domain\n\nThis domain is for use in illustrative examples..."
      }
    }
  ]
}
```

**Возвращаемый формат (ошибка)**:
```json
{
  "description": "Failed to fetch https://example.com",
  "messages": [
    {
      "role": "user",
      "content": {
        "type": "text",
        "text": "Failed to fetch https://example.com - status code 404"
      }
    }
  ]
}
```

---

## 3. Серверы без промтов

Следующие серверы реализуют только инструменты (tools), но не предоставляют промты:

### 3.1. Memory Server
**Расположение**: `src/memory/index.ts`
**Функциональность**: Управление графом знаний с сущностями, отношениями и наблюдениями
**Инструменты**: create_entities, create_relations, add_observations, delete_entities, delete_observations, delete_relations, read_graph, search_nodes, open_nodes

### 3.2. Sequential Thinking Server
**Расположение**: `src/sequentialthinking/index.ts`
**Функциональность**: Динамическое решение проблем через последовательные шаги мышления
**Инструменты**: sequentialthinking

### 3.3. Git Server
**Расположение**: `src/git/src/mcp_server_git/server.py`
**Функциональность**: Операции с Git репозиториями
**Инструменты**: git_status, git_diff_unstaged, git_diff_staged, git_diff, git_commit, git_add, git_reset, git_log, git_create_branch, git_checkout, git_show, git_branch

### 3.4. Filesystem Server
**Расположение**: `src/filesystem/index.ts`
**Функциональность**: Безопасные операции с файловой системой
**Инструменты**: read_file, read_text_file, read_media_file, read_multiple_files, write_file, edit_file, create_directory, list_directory, list_directory_with_sizes, directory_tree, move_file, search_files, get_file_info

### 3.5. Time Server
**Расположение**: `src/time/src/mcp_server_time/server.py`
**Функциональность**: Утилиты для работы со временем и часовыми поясами
**Инструменты**: (утилиты времени)

---

## Общие паттерны использования промтов

### Архитектура промтов в MCP

1. **Регистрация промтов**: Через `ListPromptsRequestSchema`
2. **Получение промта**: Через `GetPromptRequestSchema`
3. **Структура ответа**: Массив сообщений с ролями (user/assistant)
4. **Типы контента**: text, image, resource

### Типы промтов

1. **Статические** (simple_prompt) - без параметров, фиксированный контент
2. **Параметризованные** (complex_prompt) - с аргументами, динамический контент
3. **Ресурсные** (resource_prompt) - включают ссылки на ресурсы
4. **Функциональные** (fetch) - выполняют операции и возвращают результаты

### Рекомендации по использованию

1. **Простые промты**: Используйте для стандартных шаблонов без настройки
2. **Параметризованные промты**: Используйте когда нужна гибкость и настройка поведения
3. **Ресурсные промты**: Используйте для работы с данными из системы ресурсов
4. **Функциональные промты**: Используйте для операций загрузки/обработки данных

---

## Технические детали реализации

### TypeScript (Everything Server)

**SDK**: `@modelcontextprotocol/sdk`
**Обработчики**:
- `setRequestHandler(ListPromptsRequestSchema, ...)` - список промтов
- `setRequestHandler(GetPromptRequestSchema, ...)` - получение конкретного промта

### Python (Fetch Server)

**SDK**: `mcp.server`
**Декораторы**:
- `@server.list_prompts()` - список промтов
- `@server.get_prompt()` - получение конкретного промта

---

## Заключение

В данном репозитории реализовано **4 промта** в **2 серверах**:
- **Everything Server**: 3 промта (simple, complex, resource)
- **Fetch Server**: 1 промт (fetch)

Остальные серверы (Memory, Sequential Thinking, Git, Filesystem, Time) предоставляют только инструменты (tools) для выполнения операций, но не используют промты для взаимодействия с AI.
