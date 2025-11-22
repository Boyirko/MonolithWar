# PLAN: PROJECT "MONOLITH WAR" (Part 1/2)
**Role:** AI-Developer (GPT-5.1-Codex-Max)
**Stack:** Node.js 22, TypeScript 5.x (Strict), Socket.IO, Canvas API, Vite.
**Architecture:** Data-Oriented (TypedArrays), Chunk-based replication.

---

## PHASE 0: Инициализация и Фундамент (Шаги 1–6)

**1. Подготовка Git репозитория**
*   **Действие:** Склонировать репозиторий, проверить имя ветки.
*   **Детали:**
    *   Если ветка `main`, переименовать в `master`: `git branch -m main master`.
    *   Убедиться, что в настройках GitHub дефолтная ветка изменена на `master`.
*   **Критерий успеха:** Локально команда `git branch` показывает `master`.

**2. Структура Монорепозитория**
*   **Действие:** Создать иерархию папок для разделения ответственности.
*   **Детали:**
    *   `/server` — код бэкенда.
    *   `/client` — код фронтенда.
    *   `/shared` — общий код (типы, константы, баланс).
    *   `/scripts` — служебные скрипты (генераторы, валидаторы).
    *   `/docs` — документация.
    *   Проставить пустые `.gitkeep` в папки, чтобы они попали в git.

**3. Лицензирование и Документация**
*   **Действие:** Добавить юридическую и проектную базу.
*   **Детали:**
    *   `LICENSE`: Текст GPLv3.
    *   `README.md`: Заголовок, ссылка на `docs/concept.md`.
    *   `.editorconfig`: (root, true, utf-8, lf, 2 spaces).
    *   `.gitignore`: Исключить `node_modules`, `dist`, `coverage`, `.env`, `*.log`, `.DS_Store`.
    *   В `docs/` скопировать предоставленные тексты: `concept.md`, `balance.md`, `stack.md`.

**4. Ассеты и Ресурсы**
*   **Действие:** Организовать статику.
*   **Детали:**
    *   Создать пути: `client/public/assets/{tiles, units, buildings, ui, audio}`.
    *   Распределить `.webp` и `.ogg` файлы по соответствующим папкам.
*   **Важно:** Не переименовывать файлы, оставить оригинальные имена.

**5. Root Configuration (NPM & TS)**
*   **Действие:** Инициализировать корень проекта.
*   **Детали:**
    *   `package.json`:
        *   `workspaces`: `["server", "client", "shared"]`.
        *   `private`: true.
        *   `scripts`: Заглушки для `build`, `test`, `dev` (делегируют в воркспейсы).
    *   Установить дев-зависимости в корень: `typescript`, `@types/node`, `vitest`, `ts-node`.
    *   `tsconfig.base.json`: Базовый конфиг.
        *   `strict`: `true` (Критично!).
        *   `target`: `ES2022`.
        *   `moduleResolution`: `Node`.
        *   `paths`: `@shared/*` -> `shared/src/*`.

**6. Первичный коммит**
*   **Команда:** `git add . && git commit -m "chore: init repo structure and typescript base"`
*   **Цель:** Зафиксировать чистый старт перед написанием кода.

---

## PHASE 1: Fail Fast & Shared Logic (Шаги 7–18)

**7. Настройка GitHub Actions (CI) — СДВИНУТО В НАЧАЛО**
*   **Мотивация:** Мы должны ловить ошибки типов и тестов с первого же коммита кода.
*   **Действие:** Создать `.github/workflows/ci.yml`.
*   **Конфиг:**
    *   Triggers: push в `master`, PR в `master`.
    *   Jobs: `lint-and-test`.
    *   Steps: Checkout -> Install Node 22 -> `npm install` -> `npm test` -> `npm run build`.
*   **Результат:** CI упадет или пройдет (пока пусто), но инфраструктура готова.

**8. Настройка пакета `@monolith/shared`**
*   **Действие:** Инициализация общего модуля.
*   **Файлы:**
    *   `shared/package.json`: `main: dist/index.js`, `types: dist/index.d.ts`.
    *   `shared/tsconfig.json`: extends `../tsconfig.base.json`, `include: ["src"]`.

**9. Общие Константы (Constants)**
*   **Файл:** `shared/src/constants.ts`.
*   **Содержание:**
    *   Размеры: `MAP_WIDTH = 400`, `MAP_HEIGHT = 240`.
    *   Чанки: `CHUNK_SIZE = 16`, `CHUNK_COLS = 25`, `CHUNK_ROWS = 15`. (Добавить `assert` проверку, что карта делится на чанки без остатка).
    *   Тики: `SERVER_TPS = 15`, `TICK_MS = 66.66`.
    *   Кулдауны: `PAINT_COOLDOWN = 15`, `FAST_MODE_COOLDOWN = 75`.

**10. Математические Утилиты (Utils)**
*   **Файл:** `shared/src/utils/coords.ts`.
*   **Методы:**
    *   `indexFromXY(x, y)`: делает из 2D 1D индекс.
    *   `xyFromIndex(index)`: обратная операция.
    *   `chunkFromPos(x, y)`: возвращает `{cX, cY}`.
    *   `getChunkId(cX, cY)`: возвращает строку `"chunk_X_Y"`.
    *   **Важно:** Все функции должны быть чистыми (pure) и покрыты типами.

**11. Тесты Математики (Unit Tests)**
*   **Действие:** Написать первые тесты.
*   **Файл:** `shared/src/__tests__/coords.test.ts`.
*   **Проверка:**
    *   Что `chunkFromPos(399, 239)` возвращает последний чанк `24, 14`.
    *   Что `indexFromXY` корректно работает на границах.

**12. Типы Сущностей (Domain Types)**
*   **Файл:** `shared/src/types.ts`.
*   **Содержание:**
    *   `type PlayerId = string`.
    *   `enum EntityType { Unit, Building }`.
    *   Интерфейсы `UnitStats`, `BuildingStats` (на основе JSON баланса).
    *   Сетевые пакеты (набросок): `ServerSnapshot`, `ClientCommand`.

**13. Схема Валидации Баланса**
*   **Файл:** `shared/src/balance/schema.ts`.
*   **Действие:** Описать TS интерфейсы, которые строго соответствуют структуре `balance.json`.

**14. Скрипт Валидации Баланса (Strict Validation)**
*   **Действие:** Создать скрипт `scripts/validate-balance.ts`.
*   **Логика:**
    *   Читает `docs/balance.json`.
    *   Проверяет типы полей (что `damage` это число или строка, которую можно распарсить).
    *   Проверяет логические ошибки (например `cost < 0`).
    *   Проверяет наличие ассетов: если юнит называется "Tank", проверяет наличие файла `client/public/assets/units/tank.webp`.
*   **Интеграция:** Добавить этот скрипт в `npm test` в корневом package.json.

**15. Генератор Типов Ассетов (Asset Map)**
*   **Действие:** Создать скрипт `scripts/gen-assets.ts`.
*   **Логика:**
    *   Сканирует папку `client/public/assets`.
    *   Генерирует файл `shared/src/assets_map.ts`.
    *   Создает `enum AssetKey { ... }`, чтобы в коде мы использовали `AssetKey.UnitTank`, а не строки.

**16. Парсинг Баланса (Runtime)**
*   **Файл:** `shared/src/balance/index.ts`.
*   **Действие:** Экспортировать загруженный и типизированный баланс. Функция `getUnitStats(name: string)`.

**17. Коммит Shared модуля**
*   **Команда:** `git add shared scripts && git commit -m "feat(shared): core types, math utils and balance validation"`
*   **Проверка:** Запушить и убедиться, что GitHub Actions прогнали тесты и валидацию баланса успешно.

---

## PHASE 2: Server Core & Infrastructure (Шаги 18–24)

**18. Инициализация `@monolith/server`**
*   **Файлы:** `server/package.json` (deps: `express`, `socket.io`, `cors`), `server/tsconfig.json`.
*   **Скрипты:** `dev` (c `ts-node-dev`), `build`, `start`.

**19. Точка входа и Socket.IO**
*   **Файл:** `server/src/index.ts` и `server/src/network/SocketServer.ts`.
*   **Действие:**
    *   Поднять HTTP сервер.
    *   Инициализировать `socket.io` с CORS `*`.
    *   Настроить middleware для авторизации (пока заглушка, просто чтение `auth.token` из handshake).

**20. Игровой Цикл (Game Loop Skeleton)**
*   **Файл:** `server/src/core/GameLoop.ts`.
*   **Логика:**
    *   Использовать `setInterval` на `TICK_MS` (66.66 мс).
    *   Класс должен принимать колбек `onTick(tickId)`.
    *   Добавить расчет `deltaTime` (на случай лагов event loop'а), писать варнинг в лог, если тик занял > 66мс.

**21. Менеджер Лобби (Lobby Manager)**
*   **Файл:** `server/src/lobby/LobbyManager.ts`.
*   **Логика:** Хранилище активных лобби (`Map<number, Lobby>`). Метод `joinPlayer(socket, nickname)`, который маршрутизирует игрока в нужное лобби.

**22. Класс Лобби (Lobby)**
*   **Файл:** `server/src/lobby/Lobby.ts`.
*   **Содержание:**
    *   Владеет экземпляром `GameMap` (см. дальше).
    *   Владеет менеджерами юнитов, зданий, игроков.
    *   Метод `tick()`: вызывает обновление всех подсистем.

**23. Базовая Структура Игрока**
*   **Файл:** `server/src/player/Player.ts`.
*   **Поля:** `id`, `socket`, `nickname`, `paintBalance`, `unitLimit`, `stats` (закрашено клеток).

**24. Коммит Инфраструктуры Сервера**
*   **Команда:** `git add server && git commit -m "feat(server): init loop, lobby manager and socket structure"`

---

## PHASE 3: World Model & Debugging (Шаги 25–30)

**25. Реализация Карты (TypedArrays)**
*   **Файл:** `server/src/world/GameMap.ts`.
*   **Поля (Критично для памяти):**
    *   `owners: Uint8Array` (размер 400*240).
    *   `cooldowns: Int8Array` (таймеры блокировки покраски).
    *   `monolithLevels: Uint8Array` (размер 375, по чанкам).
    *   `monolithOwners: Uint8Array` (размер 375).
*   **Методы:**
    *   `setOwner(x, y, id)`, `getOwner(x, y)`.
    *   `isWalkable(x, y)`: проверка, не является ли клетка Войдом (под монолитом).

**26. Инициализация Монолитов (World Gen)**
*   **Метод:** `GameMap.initialize()`.
*   **Логика:** Проход по всем чанкам. Вычисление центра. Пометка 4-х центральных клеток в `owners` специальным ID (например 255 - VOID/WALL), чтобы там нельзя было ходить.

**27. ASCII-Визуализатор (Debug Tool)**
*   **DANGER:** Без этого мы слепы до создания клиента.
*   **Файл:** `server/src/debug/MapPrinter.ts`.
*   **Функционал:**
    *   Метод `printChunk(map, chunkX, chunkY)`.
    *   Выводит в `console.log` сетку 16x16 символов.
    *   `.` - пусто, `#` - стена/монолит, `1,2` - ID игроков.
    *   Добавить команду консоли (или временный вызов при старте), чтобы вывести состояние рандомного чанка.

**28. Система "Грязных Чанков" (Dirty Chunks)**
*   **Цель:** Оптимизация сетевого трафика. Не слать весь мир.
*   **В `GameMap`:** Добавить `dirtyChunks: Set<number>`.
*   **Логика:** При любом изменении в `setOwner`, добавлять индекс чанка в Set.
*   **В конце тика:** Методом `getAndClearDirtyChunks()` забирать список для отправки обновлений.

**29. Тесты Карты**
*   **Файл:** `server/src/__tests__/GameMap.test.ts`.
*   **Проверка:** Запись и чтение владельцев. Проверка, что запись в одном чанке помечает его как Dirty.

**30. Коммит Модели Мира**
*   **Команда:** `git add server && git commit -m "feat(server): gamemap with typedarrays and ascii debugger"`

---

## PHASE 4: Spawning & Lobby Logic (Шаги 31–35)

**31. Алгоритм Спирального Поиска**
*   **Файл:** `server/src/utils/SpiralSearch.ts`.
*   **Задача:** Найти свободный чанк ближе к центру.
*   **Логика:** Генератор координат (dX, dY), двигающийся по квадратной спирали.

**32. Unit Test для Спирали**
*   **Файл:** `server/src/__tests__/SpiralSearch.test.ts` (Critical).
*   **Сценарий:** Замокать функцию `isChunkFree`. Заставить её возвращать `false` для центрального чанка и `true` для соседа. Убедиться, что алгоритм находит соседа, а не уходит в бесконечный цикл.

**33. Логика Спавна (SpawnSystem)**
*   **Файл:** `server/src/gameplay/SpawnSystem.ts`.
*   **Действие:**
    *   Найти чанк через спираль.
    *   **Очистить территорию:** Закрасить квадрат 10x10 вокруг монолита ID игрока (через `GameMap`).
    *   Поставить **Казарму** (Entity) в центре (пока просто заглушка в данных).
    *   Обновить счетчик владения чанком (если >200 клеток, присвоить монолит).

**34. Late Join & Reconnect**
*   **В `LobbyFunctions`:**
    *   Если игрок по `token` уже есть в памяти -> вернуть управление.
    *   Если игрока нет, но `totalCapturedMonoliths < 200` -> `SpawnSystem.spawnNewPlayer()`.
    *   Иначе -> Отказ во входе.

**35. Коммит Спавна**
*   **Команда:** `git add server && git commit -m "feat(server): spiral spawn algorithm and late join logic"`

---

## PHASE 5: Gameplay Implementation (Шаги 36–42)

**36. Менеджер Юнитов (UnitManager)**
*   **Файл:** `server/src/entities/UnitManager.ts`.
*   **Структура:** Хранит `Map<UnitId, Unit>`.
*   **Методы:**
    *   `spawn(type, x, y, owner)`.
    *   `tick()`: вызывает обновление каждого юнита.
*   **Оптимизация:** Пространственное хеширование (Spatial Hash) или списки юнитов по чанкам (для быстрого поиска врагов и коллизий). *Рекомендация агенту: использовать массив Set-ов по индексу чанка `unitsByChunk: Set<UnitId>[]`*.

**37 a. Передвижение: Математика (Movement Math)**
*   **Файл:** `server/src/gameplay/MovementSystem.ts`.
*   **Логика:**
    *   Вектор к цели. Нормализация.
    *   `position += direction * speed`.
    *   Ограничение скорости `ticksPerTile`.

**37 b. Передвижение: Коллизии (Collision Avoidance)**
*   **Логика (Soft Collisions):**
    *   Если юнит входит в клетку с другим юнитом -> оттолкнуть обоих в противоположные стороны (легкое смещение координат, не блокировка).
    *   Если юнит упирается в Здание -> Скольжение вдоль грани (Slide logic).

**37 c. Передвижение: Fast Mode**
*   **Логика:** Счетчик `ticksSinceCombat`. Если > 75, множитель скорости 4x. Сброс счетчика при атаке/уроне.

**38. Менеджер Видимости (Fog of War Logic)**
*   **Файл:** `server/src/visibility/VisibilityManager.ts`.
*   **Логика:**
    *   Хранит `playerVisibleChunks: Map<PlayerId, Set<ChunkId>>`.
    *   `recalculate(player)`: бежит по всем клеткам (дорого) ИЛИ слушает события захвата (дешево).
    *   *Решение:* При захвате первой клетки в чанке -> добавить чанк + 8 соседей в видимость. При потере последней -> удалить.

**39. Интеграция Тумана с Socket.IO (Network Optimization)**
*   **Файл:** `server/src/network/RoomSync.ts`.
*   **Действие:**
    *   Когда `VisibilityManager` говорит "Add Chunk X", делаем `socket.join("chunk_X_Id")`.
    *   Это гарантирует, что клиент физически не получит пакеты из других чанков.

**40 a. Покраска: Механика (Painting Mechanic)**
*   **Файл:** `server/src/gameplay/PaintingSystem.ts`.
*   **Логика Тика:**
    *   Для каждого юнита, если `paintRadius > 0` и нет `cooldown`.
    *   Цикл по клеткам в радиусе.
    *   Валидация: `!building`, `map.cooldown == 0`, `player.money >= cost`.
    *   Применение: `map.setOwner`, `map.setCooldown`.
    *   Списание денег.

**40 b. Покраска: Владение Чанком (Chunk Ownership)**
*   **Логика Trigger-based:**
    *   Когда `setOwner` меняет владельца клетки:
    *   `chunkCounters[oldOwner]--`, `chunkCounters[newOwner]++`.
    *   Если `chunkCounters[newOwner] > 200`: Захват Монолита.
    *   Обновление тира/эффектов монолита.

**41. Боевая система (Combat)**
*   **Файл:** `server/src/gameplay/CombatSystem.ts`.
*   **Логика:**
    *   Поиск ближайшего врага (использовать `unitsByChunk` для поиска только в соседних чанках, а не по всей карте).
    *   Проверка радиуса (`distSq < range * range`).
    *   Нанесение урона (Instant).
    *   Генерация события `AttackEvent` (для отрисовки линии).
    *   Сброс `FastMode` таймеров у обоих (атакующий и жертва).

**42. Экономика (Economy)**
*   **Файл:** `server/src/gameplay/EconomySystem.ts`.
*   **Логика:**
    *   Раз в 15 тиков (1 сек).
    *   Итерируется по игрокам -> по их чанкам.
    *   `Income += ownedCells * monolithTier`.
    *   Отправка пакета обновления ресурсов.

---

## PHASE 6: Сборка Снепшотов (Шаги 43–45)

**43. Формирование Снепшотов (Snapshot Builder)**
*   **Файл:** `server/src/network/SnapshotBuilder.ts`.
*   **Задача:** Преобразовать состояние игры в сжатый JSON для каждого чанка.
*   **Логика:**
    *   Для каждого чанка:
    *   Если чанк не в `dirtyChunks` и юниты не двигались -> скип.
    *   Собрать массив юнитов в этом чанке: `[id, type, x, y, owner]`.
    *   Собрать дельты карты (изменившиеся клетки).
    *   Собрать события выстрелов.

**44. Отправка по комнатам (Broadcasting)**
*   **Действие:**
    *   `io.to(chunkRoomId).emit('u', chunkData)`.
    *   Используем короткие имена событий ('u' = update) для экономии трафика.

**45. Коммит Геймплея**
*   **Команда:** `git add server && git commit -m "feat(server): full gameplay loop (move, paint, fog, combat)"`
*   **Валидация:** Билд должен проходить. Тесты карты и спирали должны быть зелеными.

---

## PHASE 7: Client Initialization & Assets (Шаги 46–50)

**46. Инициализация Vite + TS**
*   **Действие:** Создание клиента.
*   **Команды:**
    *   Перейти в `client/`.
    *   `npm create vite@latest . -- --template vanilla-ts`.
    *   Удалить мусор (`counter.ts`, `style.css`, дефолтный HTML).
    *   Установить: `npm install socket.io-client`.

**47. Настройка Алиасов и TS**
*   **Файл:** `client/vite.config.ts`.
*   **Конфиг:** Настроить `resolve.alias`, чтобы `@shared` указывал на `../shared/src`.
*   **Проверка:** В `client/tsconfig.json` добавить `paths` аналогично корневому конфигу. Это критично, чтобы клиент видел типы пакетов и баланса.

**48. Загрузчик Ассетов (Asset Loader)**
*   **Файл:** `client/src/engine/AssetLoader.ts`.
*   **Задача:** Загрузить картинки и звук ПЕРЕД стартом игры.
*   **Логика:**
    *   Импортировать `AssetKey` из `@shared/assets_map` (сгенерированного на шаге 15).
    *   Метод `loadAll()`: возвращает `Promise<void>`.
    *   Проходит по ключам enum'а, создает `new Image()`, ждет `onload`.
    *   Сохраняет images в `Map<AssetKey, HTMLImageElement>`.
    *   *Деталь:* Добавить простейший прогресс-бар в HTML (`Loading... 0%`), обновлять его по мере загрузки.

**49. Сетевой Клиент (Network Client)**
*   **Файл:** `client/src/network/SocketClient.ts`.
*   **Логика:**
    *   Подключение: `io(import.meta.env.VITE_SERVER_URL, { autoConnect: false })`.
    *   Методы-обертки: `login(name)`, `move(ids, x, y)`, `build(...)`.
    *   Типизация: Использовать интерфейсы из `@shared/types` для payload'ов событий. Мы не должны слать `any`.

**50. Коммит Клиентского Ядра**
*   **Команда:** `git add client && git commit -m "feat(client): vire setup, asset loader and socket wrapper"`

---

## PHASE 8: Client Core Engine (Шаги 51–56)

**51. Клиентский Стейт (Game State Store)**
*   **Файл:** `client/src/state/GameState.ts`.
*   **Структура:**
    *   `snapshots: ServerSnapshot[]` — буфер последних 5 обновлений.
    *   `visibleChunks: Set<string>` — список ID чанков, по которым есть данные.
    *   `entities: Map<number, InterpolatedEntity>` — текущее сглаженное состояние для рендера.
    *   `myPlayer: PlayerData`.
*   **Логика:** При получении snapshot (шаг 49) — пушить в буфер, удалять старые (старше 2 секунд).

**52. Интерполятор (The Smoothness Magic)**
*   **Файл:** `client/src/engine/Interpolator.ts`.
*   **Алгоритм:**
    *   Определить `renderTime = now() - 100ms` (задержка для плавности).
    *   Найти в буфере два снимка: `prev` (время <= renderTime) и `next` (время > renderTime).
    *   Вычислить `alpha = (renderTime - prev.time) / (next.time - prev.time)`.
    *   Линейная интерполяция `x = prev.x + (next.x - prev.x) * alpha`.
    *   Если `next` не найден (лаг сети) — экстраполяция (продолжать движение по вектору) или стоп.

**53. Камера и Viewport**
*   **Файл:** `client/src/engine/Camera.ts`.
*   **Параметры:** `x`, `y` (центр экрана в мировых координатах), `zoom` (масштаб).
*   **Методы:**
    *   `worldToScreen(wx, wy)` -> `{sx, sy}` (учитывая zoom и offset).
    *   `screenToWorld(sx, sy)` -> `{wx, wy}` (для кликов мыши).
    *   Ограничение: Не давать камере улетать за пределы `MAP_WIDTH * CELL_SIZE`.

**54. Рендерер: Основа и Карта**
*   **Файл:** `client/src/engine/Renderer.ts`.
*   **Setup:** Canvas на весь экран, `requestAnimationFrame`.
*   **Слой Карты:**
    *   Определить видимые чанки через `Camera.getVisibleRect()`.
    *   Цикл по видимым чанкам. Если данных нет (туман/не подписан) — рисовать тайл "Void" или темноту.
    *   Если данные есть — рисовать тайлы земли (из ассетов) + оверлей цвета владельца (`globalCompositeOperation` или полупрозрачный rect).
*   **Оптимизация:** Использовать `OffscreenCanvas` для кеширования чанков, если FPS упадет (пока опционально).

**55. Рендерер: Сущности**
*   **Дополнение:**
    *   Поверх карты рисовать Здания (спрайты 2x2).
    *   Затем Юнитов.
    *   Над юнитами — HP Bar (если < 100%).
    *   Линии атаки: рисовать `lineTo` с затуханием opacity в зависимости от времени события.

**56. Управление (Input System)**
*   **Файл:** `client/src/engine/Input.ts`.
*   **События:**
    *   `wheel`: Изменение Zoom камеры.
    *   `mousedown/mousemove` (ПКМ): Drag камеры.
    *   `mousedown` (ЛКМ):
        *   Нажатие -> запомнить startX, startY.
        *   Отпускание -> если дистанция мала, это **Клик** (выделение 1 юнита).
        *   Если велика, это **Рамка** (Box Selection).
    *   Конвертация рамки из экранных в мировые координаты, выборка своих юнитов внутри.

---

## PHASE 9: UI & Polish (Шаги 57–62)

**57. HUD Верстка (Vanilla HTML)**
*   **Файл:** `client/index.html` + `client/src/ui/styles.css`.
*   **Элементы:**
    *   `#loading-screen`: перекрывает всё.
    *   `#login-screen`: форма ввода ника.
    *   `#game-ui` (hidden по умолчанию):
        *   `.top-left`: Лидерборд (таблица).
        *   `.top-center`: Таймер победы (крупно).
        *   `.bottom-left`: Панель строительства (кнопки с иконками зданий + биндинг A/S/D).
        *   `.resource-panel`: Краска и Лимит.

**58. Логика UI (UIManager)**
*   **Файл:** `client/src/ui/UIManager.ts`.
*   **Связка:** Подписывается на обновление стейта.
*   **Действия:**
    *   Обновляет цифры баланса и лимита.
    *   Рендерит строки лидерборда.
    *   Вешает классы `active` на кнопки строительства.
    *   Обрабатывает клики по кнопкам -> посылает команду `build` в SocketClient.

**59. Авто-покраска и Fast Mode UI**
*   **Функция:** Toggle button "Auto Paint (P)".
*   **Логика:**
    *   При нажатии P -> шлет команду на сервер.
    *   Визуально меняет стиль курсора или иконку режима.

**60. Звуковой движок (Audio System)**
*   **Файл:** `client/src/audio/AudioSystem.ts`.
*   **Логика:**
    *   Использовать Web Audio API.
    *   Пул звуков (чтобы можно было играть 10 выстрелов одновременно).
    *   `play(key, volume)`: реагирует на события из snapshot (events array).
    *   *Важно:* Дебаунс звуков. Если пришло 50 атаки в одном тике — проиграть звук атаки 1 раз громче, а не 50 раз, иначе уши лопнут.

**61. Экраны Победы и Поражения**
*   **UI:** Модальное окно поверх канваса.
*   **Логика:**
    *   Событие `GameOver` от сокета.
    *   Показ статистики.
    *   Кнопка "Play Again" (релоад страницы или переподключение).

**62. Коммит Клиента (Final)**
*   **Команда:** `git add client && git commit -m "feat(client): full implementation (render, interpolation, ui, input)"`

---

## PHASE 10: Production Build & Docker (Шаги 63–69)

**63. Конфигурация Сборки**
*   **Задача:** Обеспечить единую точку сборки.
*   **Файл:** Корневой `package.json`.
*   **Скрипт:** `"build": "npm run build --workspace server && npm run build --workspace client"`.
*   **Проверка:** Запустить `npm run build` локально. Убедиться, что появились папки `server/dist` и `client/dist`.

**64. Подготовка Статики на Бэке**
*   **Файл:** `server/src/index.ts`.
*   **Правка:**
    *   Добавить проверку `if (process.env.NODE_ENV === 'production')`.
    *   Если да: `app.use(express.static(path.join(__dirname, '../../client/dist')))`.
    *   На любой GET запрос (кроме /api) отдавать `index.html`.

**65. Dockerfile (Multi-stage Optimised)**
*   **Файл:** `Dockerfile` (в корне).
*   **Stage 1 (Builder):**
    *   `FROM node:22-alpine AS builder`
    *   `COPY . .`
    *   `RUN npm ci && npm run build`
    *   Удаление исходников, оставляем только `dist` и `package.json`.
*   **Stage 2 (Runner):**
    *   `FROM node:22-alpine`
    *   `WORKDIR /app`
    *   Копируем из builder: `node_modules`, `server/dist`, `client/dist`, `shared/dist`.
    *   `CMD ["node", "server/dist/index.js"]`.

**66. Healthcheck в Docker**
*   **Docker:** Добавить инструкцию `HEALTHCHECK --interval=30s --timeout=3s CMD curl -f http://localhost:8080/health || exit 1`.
*   **Server:** В `server/src/index.ts` добавить эндпоинт `app.get('/health', (_, res) => res.sendStatus(200));`.

**67. Локальный Тест Контейнера**
*   **Команды:**
    *   `docker build -t monolith-war .`
    *   `docker run -p 8080:8080 monolith-war`
*   **Проверка:** Открыть браузер, поиграть сам с собой. Убедиться, что ассеты грузятся, вебсокет коннектится.

**68. Коммит Docker config**
*   **Команда:** `git add Dockerfile server/src/index.ts && git commit -m "chore: dockerfile with multi-stage build and production static serving"`

---

## PHASE 11: Advanced CI/CD Pipeline (Шаги 69–73)

**69. Workflow: Build & Push Hooks**
*   **Файл:** `.github/workflows/release.yml`.
*   **Триггер:** `push` в `master`.

**70. Job: Deploy to Build Branch (Artifacts)**
*   **Шаги:**
    *   Checkout -> Install -> Build.
    *   Создать папку `deploy_artifact`.
    *   Скопировать туда `server/dist`, `client/dist`, `package.json`, `Dockerfile`.
    *   Использовать экшен `JamesIves/github-pages-deploy-action` (или аналог).
    *   Target Branch: `build`.
    *   Folder: `deploy_artifact`.
    *   **Результат:** В ветке `build` всегда лежит готовый к запуску код без TS-исходников.

**71. Job: Docker Publish to GHCR**
*   **Шаги:**
    *   Login to GitHub Container Registry (`ghcr.io`).
    *   Build Docker Image.
    *   Push с тегом `:latest` и `:${{ github.sha }}`.
*   **Важно:** Убедиться в настройках репозитория, что Actions имеют права на запись пакетов.

**72. Документация по Деплою**
*   **Файл:** `docs/deploy.md`.
*   **Содержание:**
    *   Команда для запуска на VPS одной строкой:
        `docker run -d --restart always -p 80:8080 ghcr.io/YOUR_USER/monolith-war:latest`
    *   Как обновлять (pull + restart).

**73. Коммит CI/CD**
*   **Команда:** `git add .github docs && git commit -m "ci: automated deployment pipeline to build branch and ghcr"`

---

## PHASE 12: Final Polish & Release (Шаги 74–80)

**74. Код Ревью (Self-Correction)**
*   **Задача Агента:** Пройтись `eslint`'ом и поправить стиль.
*   **Задача Агента:** Проверить наличие `console.log` (удалить дебажные, оставить важные info в server).

**75. Обновление README.md**
*   **Действие:** В корне написать красивый Readme.
*   **Секции:**
    *   "How to play" (Управление, правила).
    *   "Development" (npm install, npm run dev).
    *   "Architecture" (Ссылка на docs/stack.md).
    *   Credits.

**76. Релиз v1.0.0**
*   **Действие:**
    *   `npm version 1.0.0` (обновит package.json).
    *   `git add . && git commit -m "chore: release 1.0.0"`.
    *   `git tag v1.0.0`.
    *   `git push origin master --tags`.

**77. Проверка Релиза**
*   **Действие:** Зайти на GitHub.
*   **Проверить:**
    *   Зеленые галочки у последнего коммита.
    *   Наличие образа в "Packages".
    *   Обновление ветки `build`.

**78. (Опционально) Скрипт "Manager"**
*   **Скрипт:** `scripts/generate-stats.ts`.
*   **Задача:** Подсчитать количество строк кода (LOC) и файлов, вывести в консоль "Project Stats", чтобы агент себя похвалил :)

**79. Финальный промпт агента - "Исполняй"**
*   **Итог:** План готов. Можно скармливать его агенту.

**80. КОНЕЦ.**

---

### Вердикт для User:
Теперь у вас есть **16-фазовый план на 80 шагов**

**Почему этот план сработает:**
1.  **Shared-First:** Агент не начнет писать сервер, пока не договорится сам с собой о типах данных.
2.  **Visual Debug:** ASCII-рендер спасет серверную логику от ошибок "вслепую".
3.  **Chunks & Rooms:** Сетевая архитектура сразу заложена масштабируемой (Socket Rooms), переписывать потом не придется.
4.  **Docker-Ready:** Деплой на VPS займет у вас ровно 1 минуту (копипаст команды из `docs/deploy.md`).