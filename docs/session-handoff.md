# Handoff: состояние работы (для продолжения в новой сессии)

> Цель файла — быстро передать контекст новой сессии ИИ, не перечитывая
> крупные файлы целиком. Обновляйте по мере продвижения.

## ПРОЕКТ

- Путь: `G:\9potok\1210`
- Веб-сервис чата с LLM: VPS (слабая модель) + ПК (сильная модель через
  reverse-SSH-туннель).
- Пакет: `rtk_app` (`config`, `agent`, `local_server`, `session_store`,
  `rag`, `mcp_client`, `key_store`).
- Сервер: `web/server.py` (класс `WebRequestHandler`, обработчики
  `_handle_*`, ветки `_handle_local`).
- Фронтенд: `index.html`, `js/app.js` (ES-модули в `js/core`, `js/features`),
  `css/style.css`.
- Тесты: `tests/check_task1_tunnel.py` (55 проверок).
- Документация: `docs/task1-local-llm-from-pc.md`, `docs/task5.md`,
  `docs/vps-llm-service.md`.

## ВАЖНЫЕ ОГРАНИЧЕНИЯ ИНСТРУМЕНТОВ (НЕ ТРАТИТЬ ВРЕМЯ ЗАНОВО)

- `edit_existing_file` / `single_find_and_replace` **стабильно НЕ находят**
  фрагменты, содержащие кириллицу (сбой кодировки при чтении файла).
  - **Обход:** временный Python-скрипт:
    ```python
    import io
    p = r'G:\9potok\1210\web\server.py'
    t = io.open(p, encoding='utf-8').read()
    anchor = '...уникальный фрагмент...'
    assert t.count(anchor) == 1
    t = t.replace(anchor, new_text, 1)
    io.open(p, 'w', encoding='utf-8', newline='').write(t)
    ```
  - После правки удалить временный скрипт.
- Проверка синтаксиса:
  - `python -X utf8 -m py_compile web\server.py`
  - `node --check js\app.js`
  - `python -X utf8 tests\check_task1_tunnel.py`
- **НЕ выводить** большие файлы через `read_file` целиком (например,
  `web/server.py` ≈ 127 тыс. символов, `js/app.js` — большой). Читать только
  нужные участки: `grep_search` или `python -c` c `splitlines` по диапазону.
- В текстах JS можно использовать unicode-escape (в файл попадает корректная
  строка).

## СДЕЛАНО: Задача 1, «Вариант А»

Кнопка «Поднять сильную модель (ПК)» для НЕСПЕЦИАЛИСТА.

Архитектурная правда: reverse-SSH-туннель физически поднимает ПК-агент
**НА ПК**, поэтому VPS не может запустить его сам. Кнопка:
1. отдаёт готовую команду запуска ПК-агента с уже подставленными параметрами
   этого VPS;
2. следит за появлением туннеля (фоновый опрос);
3. при успехе автоматически переключает движок на сильную модель
   (`tunnel_up`).

### Изменённые файлы

- **`web/server.py`**
  - `_pair_payload()` — собирает команду запуска ПК-агента:
    - `command` — однострочная для PowerShell;
    - `script`  — многострочный `.ps1`;
    - параметры подставляются из `config`: `TUNNEL_SSH`, `LOCAL_PORT`,
      `TUNNEL_PORT`, `AGENT_PORT`; ключ `%USERPROFILE%\.ssh\rtk_pc`;
    - при пустом `TUNNEL_SSH` — плейсхолдер `rtk@<IP-этого-VPS>`.
  - ветка `action == "pair"` в `_handle_local` — отдаёт `pair` (команда) +
    текущий `tunnel` (статус).

- **`index.html`**
  - кнопка `#local-pc-pair` «Поднять сильную модель (ПК)»;
  - панель `#local-pair-box` (скрыта по умолчанию): подсказка, поле
    `#local-pair-cmd`, кнопки `#local-pair-copy`, `#local-pair-check`,
    `#local-pair-hide`, статус `#local-pair-status`.

- **`css/style.css`**
  - стили `.local-pc-pair`, `.local-pair-box`, `.local-pair-hint`,
    `.local-pair-cmd`.

- **`js/app.js`**
  - ссылки на элементы панели;
  - `renderLocalPair(d)` — отрисовка команды и статуса;
  - `startLocalPairPoll()` / `stopLocalPairPoll()` — фоновый опрос
    `/api/local` (action=`pair`) каждые 4 с, пока панель открыта;
  - обработчики кнопок: показать команду, копировать в буфер,
    «Проверить подключение» (при `tunnel.pc_alive` → `tunnel_up`), скрыть;
  - проброс `d.pair` в `localState` внутри `applyLocalState`.

### Дошито в этой сессии

- **Тесты:** добавлена секция 5 в `tests/check_task1_tunnel.py` —
  проверки `_pair_payload`: набор ключей, соответствие портов `config`,
  формат `command` (одна строка) / `script` (многострочный), подстановка
  `TUNNEL_SSH` и плейсхолдер `rtk@<IP-этого-VPS>` при пустом значении.
  Итог: **55 OK, 0 FAIL** (было 31).
- **Документация:** в `docs/task1-local-llm-from-pc.md` обновлены таблица
  §12 (UI, ВПС-логика, тесты) и добавлен раздел **§13 «Вариант А»**
  (проблема → решение → компоненты → тесты).

### Статус проверки

- `web/server.py` — `py_compile` OK.
- `js/app.js` — `node --check` OK.
- `tests/check_task1_tunnel.py` — **55 OK, 0 FAIL**.

## КОНФИГ (значения по умолчанию)

- `LOCAL_PORT = 8080`
- `TUNNEL_PORT = 18080`
- `AGENT_PORT = 18081`
- `TUNNEL_SSH = ''` (пусто → плейсхолдер в команде)

## ЧТО ДАЛЬШЕ (варианты)

- Протестировать панель подключения ПК в браузере вживую (нужен запущенный
  сервер `web/server.py` и открытая страница `index.html`).
- Дописать тест на ветку `action == "pair"` в `_handle_local` (сейчас
  покрыт только `_pair_payload`, не HTTP-ветка).
- Обновить `docs/task5.md` / `docs/vps-llm-service.md` при необходимости.

## КАК НАЧАТЬ НОВУЮ СЕССИЮ

Начните с фразы: «Продолжаем проект `G:\9potok\1210` по выжимке в
`docs/session-handoff.md`» — и сразу переходите к задаче, не перечитывая
файлы целиком.
