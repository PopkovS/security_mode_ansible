# Опасные модули/возможности Ansible — памятка для парсера

Итоговый набор: 13 плейбуков вместо исходных 17, дублирующиеся варианты
одного и того же риска объединены. Каждый файл = один риск, в комментариях —
объяснение, безопасная опция (если есть) и совет по детекту.

## Структура

```
delegation_to_server.yml   — connection: local / delegate_to / local_action (САМОЕ ВАЖНОЕ)
lookup_query.yml           — lookup()/query()/with_<plugin>
vars_file_read.yml         — vars_files / include_vars
include_tasks.yml          — include_tasks / import_tasks / import_playbook
uri_get_url.yml            — uri/get_url (опасны только в связке с делегированием)
files_operation/
  copy.yml                 — remote_src: true чинит
  template.yml             — safe-опции нет, чинится только запретом lookup
  assemble.yml              — remote_src: true чинит
  unarchive.yml             — remote_src: true чинит
  script.yml                — safe-опции нет, только allowlist путей
  fetch.yml                  — safe-опции нет, только allowlist dest + проверка пути
  synchronize.yml            — safe-опции нет, рекомендуется банить целиком
  include_role.yml           — path traversal + плагины роли, allowlist + скан содержимого
```

## Общие замечания, важные для архитектуры парсера

1. **Делегирование важнее списка модулей.** `delegate_to: localhost` /
   `connection: local` / `local_action` превращают ЛЮБОЙ модуль в опасный.
   Это правило должно проверяться независимо от того, есть ли модуль
   в вашем блок-листе.

2. **Модуль можно вызвать 3 способами:**
   - как ключ таска: `copy: {...}`
   - полным именем (FQCN): `ansible.builtin.copy: {...}` /
     `ansible.legacy.copy: {...}`
   - через `action:` — `action: copy src=... dest=...`
   Матчить нужно все три формы, иначе `action:` станет готовым обходом.

3. **Парсите YAML как дерево, а не текст.** Регулярки полезны как первый
   грубый фильтр (быстро найти кандидатов), но финальное решение должно
   приниматься по структуре: в каком именно поле таска встретилось имя
   модуля/директивы, какие у него соседние параметры (remote_src,
   delegate_to и т.д.).

4. **Jinja-выражения могут быть где угодно.** `lookup(`/`query(` и
   `with_<plugin>:` не привязаны к конкретному модулю (debug и т.п.) —
   нужно обходить все строковые значения во всём YAML-дереве.

5. **Рекурсия обязательна.** `include_tasks`/`import_tasks`/
   `import_playbook`/`include_role`/`import_role`/`roles:` подключают
   внешние файлы — если сканер не идёт внутрь них, это готовый обход
   проверки одним вложенным файлом.

6. **Динамические значения регуляркой не поймать.** Если `delegate_to`
   или путь `src`/`dest` заданы через переменную (`{{ some_var }}`),
   статический анализ текста бессилен — либо запрещать такие поля
   без литеральных значений в принципе, либо резолвить переменные
   перед проверкой.

## Таблица: модуль → риск → как обезопасить

| Модуль/директива | Риск | Safe-опция | Если safe-опции нет |
|---|---|---|---|
| `delegate_to` / `connection: local` / `local_action` | любой таск выполняется на сервере | нет — allowlist хостов, никогда localhost | — |
| `lookup()` / `query()` / `with_<plugin>` | чтение файлов/env/сети сервера | allowlist безопасных плагинов | — |
| `vars_files` / `include_vars` | чтение файла сервера как переменных | нет | allowlist путей внутри проекта |
| `copy` | чтение файла сервера | `remote_src: true` или `content:` | — |
| `template` | чтение файла + рендер Jinja на сервере | нет | запрет lookup внутри шаблонов |
| `assemble` | чтение директории сервера | `remote_src: true` | — |
| `unarchive` | чтение архива сервера | `remote_src: true` | — |
| `script` | чтение и запуск скрипта с сервера | нет | allowlist директорий src |
| `fetch` | запись файла клиента на сервер | нет | allowlist dest + нормализация пути |
| `synchronize` | rsync всегда на сервере, `rsync_opts` обходит ограничения | нет | банить модуль целиком |
| `include_role` / `import_role` / `roles:` | path traversal + python-плагины роли | allowlist путей + запрет library/ и т.п. подпапок | — |
| `include_tasks` / `import_tasks` / `import_playbook` | обход сканера через вложенный файл | рекурсивная проверка подключаемых файлов | — |
| `uri` / `get_url` | сами по себе безопасны (выполняются на клиенте) | опасны только вместе с delegate_to/connection: local | — |
