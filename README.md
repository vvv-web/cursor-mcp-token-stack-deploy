# Cursor + MCP: экономия токенов (стек по Habr)

Репозиторий описывает **локальное развёртывание** связки из статьи на Хабре  
**«Тюнинг Cursor: как я укротил AI-ассистента и радикально снизил счета за токены с помощью MCP-серверов»**  
→ https://habr.com/ru/articles/1029868/

Цель: **граф знаний монорепо** (Graph RAG MCP → Neo4j) и при необходимости **query fusion** — тот же MCP **`graph-rag`** с **`graph_rag_fused_code_search`** (**Ollama + Qdrant** в Docker), плюс **сжатие контекста** (lean-ctx), **прокси мета-инструментов** (mcp-on-demand). Отдельный **rag-code** MCP — опционально, если нужен второй векторный контур; см. **`skills/graph-rag/MCP.md`** для переменных **`QDRANT_URL`**, **`OLLAMA_BASE_URL`**, **`OLLAMA_EMBED`**, опционально **`GRAPH_RAG_FUSION_QDRANT_COLLECTION`** на процессе **`graph-rag`**.

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
| **graph-rag** (путь по умолчанию) | MCP **stdio**: `graph_rag_mcp_server.py` — Neo4j (`graph_rag_retrieve`, сущности, multi-hop). Для кода по смыслу с подмешиванием терминов из графа: **`graph_rag_fused_code_search`** при **`QDRANT_URL`**, **`OLLAMA_BASE_URL`**, **`OLLAMA_EMBED`** в `env` сервера (опционально **`GRAPH_RAG_FUSION_QDRANT_COLLECTION`**). Секреты Neo4j: `graph-rag-infra/.env.neo4j`. Канон: **`skills/graph-rag/MCP.md`**. |
| **rag-code-mcp** (опционально) | Отдельный векторный MCP `search_code`, если нужен контур без fusion. Тот же **Ollama + Qdrant**, что и для fusion на **`graph-rag`**. |
| **lean-ctx** | Rust-бинарь: сжатие вывода терминала и кэшируемое чтение (`ctx_*` инструменты в Cursor). |
| **mcp-on-demand** | npm-пакет `@soflution/mcp-on-demand`: прокси с двумя мета-инструментами, lazy-load остальных MCP (подключайте сюда же **graph-rag** и **lean-ctx** в конфиге on-demand, если используете мета-обёртку). |
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
7. **`~/.cursor/mcp.json`** — добавлены серверы (без удаления существующих: serena, playwright, openclaw-gateway и т.д.):

   - **`graph-rag`** (основной RAG + опционально fusion): `python3` на `Desktop/skills/graph-rag/scripts/graph_rag_mcp_server.py`, `env`: `GRAPH_RAG_WORKSPACE`, `PYTHONPATH` на `.../skills/graph-rag/scripts`, при необходимости **`QDRANT_URL`**, **`OLLAMA_BASE_URL`**, **`OLLAMA_EMBED`**, опционально **`GRAPH_RAG_FUSION_QDRANT_COLLECTION`** (см. **`skills/graph-rag/MCP.md`**); `pip install -r skills/graph-rag/requirements-mcp.txt`.
   - **`rag-code`** (опционально, только если нужен отдельный бинарь RagCode): `command` на `~/.local/bin/rag-code-mcp`, `-config` на `config.yaml`, `env` Ollama/Qdrant.
   - **`lean-ctx`**: `command` — `~/.cargo/bin/lean-ctx`.
   - **`mcp-on-demand`**: `npx -y @soflution/mcp-on-demand`.

8. **Полный перезапуск Cursor** после правок `mcp.json`.
9. **Проверка UI**: Settings → MCP — серверы **graph-rag** (и при необходимости **rag-code**), **lean-ctx**, **mcp-on-demand** без ошибок (статус «зелёный»).

Опционально по статье (не обязательно для минимального запуска):

- `lean-ctx init --global` — хук в shell для автосжатия вывода терминала.
- Правила в проекте: `.cursor/rules/*.mdc` с этапами A/B (разведка **graph-rag** → работа через **lean-ctx**), лимиты токенов.

---

## Где лежат файлы на этой машине (канон для воспроизведения)

| Назначение | Путь |
|------------|------|
| Бинарь RagCode (исходный) | `~/Downloads/rag-code-mcp-v1.1.21/rag-code-mcp` |
| Конфиг RagCode | `~/Downloads/rag-code-mcp-v1.1.21/config.yaml` |
| MCP Graph RAG (stdio) | `~/Desktop/skills/graph-rag/scripts/graph_rag_mcp_server.py` |
| lean-ctx | `~/.cargo/bin/lean-ctx` |
| Конфиг MCP Cursor | `~/.cursor/mcp.json` |

**Важно:** `~/.cursor/mcp.json` содержит личные ключи и токены других серверов — **не коммитить** как есть. В этом репозитории только документация и примеры.

---

## Пример фрагмента `mcp.json` (без секретов других серверов)

Подставьте свои пути и при необходимости модель (`phi3:mini` вместо `medium`):

```json
{
  "mcpServers": {
    "graph-rag": {
      "command": "python3",
      "args": [
        "/home/USER/Desktop/skills/graph-rag/scripts/graph_rag_mcp_server.py"
      ],
      "env": {
        "GRAPH_RAG_WORKSPACE": "/home/USER/Desktop",
        "PYTHONPATH": "/home/USER/Desktop/skills/graph-rag/scripts",
        "QDRANT_URL": "http://127.0.0.1:6333",
        "OLLAMA_BASE_URL": "http://127.0.0.1:11434",
        "OLLAMA_EMBED": "nomic-embed-text"
      }
    },
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

- [ ] Neo4j для Graph RAG запущен; файл **`graph-rag-infra/.env.neo4j`** на месте (`NEO4J_PASSWORD`).
- [ ] `pip install -r ~/Desktop/skills/graph-rag/requirements-mcp.txt` и `python3 ~/Desktop/skills/graph-rag/scripts/graph_rag_mcp_server.py` не падает при импорте (проверка без долгого stdio).
- [ ] (Если используете RagCode) `docker ps` — **ollama** и **qdrant**; `rag-code-mcp -health ...`
- [ ] Cursor: MCP без ошибок для **graph-rag**, **lean-ctx**, **mcp-on-demand** (и **rag-code**, если включён).
- [ ] Для RagCode: первый запрос к коду может триггерить индексацию — подождать.

---

## Две фазы (обновлённая связка)

1. **Этап A** — **`graph-rag`**: граф (`graph_rag_retrieve`, сущности, multi-hop). Семантика по коду с графом — **`graph_rag_fused_code_search`**, если в `env` сервера заданы **Qdrant + Ollama** (см. **`skills/graph-rag/MCP.md`**). Отдельный **rag-code** — только при необходимости второго контура.
2. **Этап B** — известные пути к файлам: **lean-ctx** для чтения/поиска с экономией токенов.

**mcp-on-demand** по-прежнему служит ленивому подключению MCP; в его конфиге можно указать те же имена серверов (**graph-rag**, **lean-ctx**), что и в основном `mcp.json`.

Официальная документация Cursor по MCP: https://cursor.com/docs/mcp  

Спецификация MCP: https://modelcontextprotocol.io  

Репозиторий RagCode: https://github.com/doITmagic/rag-code-mcp  

---

## Лицензия этого репозитория

Документация собрана для личного воспроизведения окружения. Бинарии и модели распространяются по лицензиям своих проектов (RagCode MIT, Ollama, Qdrant и т.д.).
