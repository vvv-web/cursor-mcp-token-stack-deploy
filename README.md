# Cursor + MCP: экономия токенов (стек по Habr)

Репозиторий описывает **локальное развёртывание** связки из статьи на Хабре  
**«Тюнинг Cursor: как я укротил AI-ассистента и радикально снизил счета за токены с помощью MCP-серверов»**  
→ https://habr.com/ru/articles/1029868/

Цель: **семантический поиск по коду** (RagCode), **сжатие контекста** (lean-ctx), **прокси мета-инструментов** (mcp-on-demand), плюс инфраструктура **Ollama + Qdrant** в Docker.

---

## Что было на машине (апрель–май 2026)

| Ресурс | Значение |
|--------|----------|
| RAM | ~62 GiB (запас под модели и контейнеры) |
| CPU | x86_64, 24 потока |
| Диск `/` | ~1.2 TiB свободно |
| Docker | установлен, используется |
| Node | v22 (для `npx` / mcp-on-demand) |

---

## Компоненты стека

| Компонент | Назначение |
|-----------|------------|
| **rag-code-mcp** | MCP-сервер: AST/векторный индекс, семантический `search_code`, локально (приватность). Стек: бинарь RagCode + **Ollama** (эмбеддинги и LLM) + **Qdrant**. |
| **lean-ctx** | Rust-бинарь: сжатие вывода терминала и кэшируемое чтение (`ctx_*` инструменты в Cursor). |
| **mcp-on-demand** | npm-пакет `@soflution/mcp-on-demand`: прокси с двумя мета-инструментами, lazy-load остальных MCP. |
| **Ollama** | Контейнер `ollama`, порт **11434**. |
| **Qdrant** | Контейнер **ragcode-qdrant** (из установки/окружения), порты **6333–6334**. |

Модели в Ollama (релевантные этому стеку):

- **`nomic-embed-text`** — эмбеддинги для индекса.
- **`phi3:medium`** (~7.9 GB) — LLM для логики RagCode по конфигу (как в статье). Докачка могла занимать долго из‑за размера слоя; **`phi3:mini`** остаётся запасным вариантом при нехватке ресурсов.

---

## Что сделали по шагам

1. **Rust toolchain** (`rustup`) — для сборки **lean-ctx**: `cargo install lean-ctx` → бинарь в `~/.cargo/bin/lean-ctx`.
2. **RagCode MCP** — скачан релиз с GitHub **doITmagic/rag-code-mcp** (linux amd64), распакован в каталог вида  
   `~/Downloads/rag-code-mcp-v1.1.21/` (бинарии `rag-code-mcp`, `ragcode-installer`, при необходимости `index-all`).
3. **Проверка зависимостей**: `rag-code-mcp -health` с URL **Ollama** и **Qdrant** — успешно.
4. **Конфиг RagCode** — файл `config.yaml` рядом с бинарём (или явно указан через `-config`):  
   `ollama_base_url`, `ollama_model`, `ollama_embed`, URL Qdrant.
5. **Симлинк** для стабильного пути в MCP:  
   `~/.local/bin/rag-code-mcp` → фактический бинарь в каталоге загрузки.
6. **Докачка модели** `phi3:medium` в контейнер Ollama:  
   `docker exec ollama ollama pull phi3:medium` (долго из‑за ~8 GB — нормально для одного большого слоя).
7. **`~/.cursor/mcp.json`** — добавлены три сервера (без удаления существующих: serena, playwright, openclaw-gateway и т.д.):

   - **`rag-code`**: `command` на `~/.local/bin/rag-code-mcp`, аргумент `-config` на абсолютный путь к `config.yaml`, `env`:  
     `OLLAMA_BASE_URL`, `OLLAMA_EMBED`, `OLLAMA_MODEL`, `QDRANT_URL`.
   - **`lean-ctx`**: `command` — `~/.cargo/bin/lean-ctx`.
   - **`mcp-on-demand`**: `npx -y @soflution/mcp-on-demand`.

8. **Полный перезапуск Cursor** после правок `mcp.json`.
9. **Проверка UI**: Settings → MCP — серверы **rag-code**, **lean-ctx**, **mcp-on-demand** без ошибок (статус «зелёный»).

Опционально по статье (не обязательно для минимального запуска):

- `lean-ctx init --global` — хук в shell для автосжатия вывода терминала.
- Правила в проекте: `.cursorrules`, `.cursor/rules/*.mdc` с этапами A/B (разведка rag-code → работа через lean-ctx), лимиты токенов.

---

## Где лежат файлы на этой машине (канон для воспроизведения)

| Назначение | Путь |
|------------|------|
| Бинарь RagCode (исходный) | `~/Downloads/rag-code-mcp-v1.1.21/rag-code-mcp` |
| Конфиг RagCode | `~/Downloads/rag-code-mcp-v1.1.21/config.yaml` |
| Симлинк для MCP | `~/.local/bin/rag-code-mcp` |
| lean-ctx | `~/.cargo/bin/lean-ctx` |
| Конфиг MCP Cursor | `~/.cursor/mcp.json` |

**Важно:** `~/.cursor/mcp.json` содержит личные ключи и токены других серверов — **не коммитить** как есть. В этом репозитории только документация и примеры.

---

## Пример фрагмента `mcp.json` (без секретов других серверов)

Подставьте свои пути и при необходимости модель (`phi3:mini` вместо `medium`):

```json
{
  "mcpServers": {
    "rag-code": {
      "command": "/home/USER/.local/bin/rag-code-mcp",
      "args": [
        "-config",
        "/home/USER/Downloads/rag-code-mcp-v1.1.21/config.yaml"
      ],
      "env": {
        "OLLAMA_BASE_URL": "http://127.0.0.1:11434",
        "OLLAMA_EMBED": "nomic-embed-text",
        "OLLAMA_MODEL": "phi3:medium",
        "QDRANT_URL": "http://127.0.0.1:6333"
      }
    },
    "lean-ctx": {
      "command": "/home/USER/.cargo/bin/lean-ctx",
      "args": []
    },
    "mcp-on-demand": {
      "command": "npx",
      "args": ["-y", "@soflution/mcp-on-demand"]
    }
  }
}
```

Остальные MCP объединяются в том же объекте `mcpServers` с существующими записями.

---

## Проверка после установки

- [ ] `docker ps` — контейнеры **ollama** и **qdrant** (ragcode-qdrant) в нужном состоянии.
- [ ] `docker exec ollama ollama list` — есть **`nomic-embed-text`** и выбранная phi3-модель.
- [ ] `rag-code-mcp -health -ollama-base-url http://127.0.0.1:11434 -qdrant-url http://127.0.0.1:6333`
- [ ] Cursor: MCP без ошибок для **rag-code**, **lean-ctx**, **mcp-on-demand**.
- [ ] Первый запрос к коду в IDE может триггерить индексацию — подождать.

---

## Идея из статьи (две фазы)

1. **Этап A** — незнакомый код: семантический поиск через **rag-code** (через **mcp-on-demand** при желании).
2. **Этап B** — известные пути: **lean-ctx** для чтения/поиска с экономией токенов.

Официальная документация Cursor по MCP: https://cursor.com/docs/mcp  

Спецификация MCP: https://modelcontextprotocol.io  

Репозиторий RagCode: https://github.com/doITmagic/rag-code-mcp  

---

## Лицензия этого репозитория

Документация собрана для личного воспроизведения окружения. Бинарии и модели распространяются по лицензиям своих проектов (RagCode MIT, Ollama, Qdrant и т.д.).
