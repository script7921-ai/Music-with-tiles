---

# ТЕХНИЧЕСКОЕ ЗАДАНИЕ
## Мобильная ритм-игра «MWT» (Music With Tiles)
**Версия документа:** 1.0 FINAL · **Платформа:** Android · **Тип:**离线 single-player, без сервера

---

# 0. Правила для исполнителя (читать первым)

## 0.1. Приоритеты
1. **Точность тайминга и отклик** — 90% успеха продукта. Важнее любого визуала.
2. Детерминизм: одинаковый вход → одинаковый результат, всегда.
3. Стабильность: некорректный файл уровня не должен крашить приложение.
4. Всё остальное.

## 0.2. Жёсткие запреты
| Запрещено | Причина |
|---|---|
| `System.currentTimeMillis()` / `nanoTime()` как источник игрового времени для судейства | Рассинхрон с музыкой |
| Любая случайная генерация нот (`Random`, `shuffle`) в чартах и Endless | Требование заказчика |
| Сетевые запросы, аналитика, рекламные SDK, Firebase | Игра полностью офлайн |
| `MANAGE_EXTERNAL_STORAGE` | Используем SAF |
| Исполнение кода/скриптов из `.mwt` | Безопасность |
| Аллокации объектов внутри кадра игрового цикла | GC-фризы |
| Изменение скорости аудио для усложнения | Ломает синхронизацию |

## 0.3. При неоднозначности
Брать значение из таблиц этого ТЗ. Если значения нет — выбрать простейшее детерминированное решение и записать в `DECISIONS.md`.

---

# 1. Продукт

| Параметр | Значение |
|---|---|
| Жанр | Ритм-аркада, 4 вертикальные дорожки, ноты падают сверху вниз |
| Ядро | Промах = мгновенная смерть (хардкор) |
| Ориентация | Только портретная, блокировка поворота |
| Язык UI | Русский (основной), English (второй) |
| Монетизация | Отсутствует. Внутриигровая валюта, без покупок за деньги |
| Целевой размер APK | ≤ 30 МБ без пользовательского контента |
| Min SDK / Target SDK | 24 / 34 |
| Целевой FPS | 60 обязательно, 120 если поддерживается |

## 1.1. Стек (обязательный)
- **Kotlin**, JDK 17
- **UI меню:** Jetpack Compose
- **Рендер игры:** `SurfaceView` + `Canvas` (при просадках — OpenGL ES 2.0)
- **Аудио:** Oboe (C++/NDK), AAudio при API ≥ 27, OpenSL ES ниже. Fallback — Media3/ExoPlayer
- **Сериализация:** kotlinx.serialization (JSON)
- **Хранение:** JSON-файлы во внутреннем хранилище + DataStore для мелких флагов
- **Асинхронность:** Coroutines
- **Тесты:** JUnit5 + Robolectric для парсера/судьи

---

# 2. Архитектура

```
com.mwt.game/
├── audio/         AudioEngine, OboeEngine, FallbackEngine, AudioClock
├── engine/        GameLoop, GameSession, Judge, NoteStateMachine, ScoreEngine
├── input/         TouchRouter, PointerTracker, GestureClassifier
├── render/        GameRenderer, LaneLayout, NoteSprites, ParticlePool
├── model/         MwtLevel, Note, RuntimeNote, PlayResult, GameConstants
├── loader/        MwtLoader, MwtValidator, MwtWriter, ZipPackage
├── modes/         CampaignRunner, EndlessDirector, PracticeRunner
├── casual/        CasualSimplifier, PreLevelChecker
├── calibration/   CalibrationStore, AutoCalibrator
├── editor/        EditorState, GridQuantizer, TapRecorder, EditorRenderer
├── progress/      ProfileRepo, RecordRepo, Wallet, UnlockGraph
├── meta/          DailyService, MissionService, ChestService, ShopService
├── ui/            экраны Compose + ViewModel
└── util/          Clock, MathExt, ObjectPool, Logger
```

**Правило зависимостей:** `ui → engine → model`. Обратных зависимостей нет. `engine` ничего не знает о Compose.

---

# 3. Формат `.mwt`

## 3.1. Два варианта одного формата

| Вариант | Как определить | Содержимое |
|---|---|---|
| **TEXT** | файл начинается с `{` (после trim BOM/пробелов) | Один JSON, аудио по внешней ссылке |
| **PACK** | первые 2 байта `0x50 0x4B` (`PK`) | ZIP-архив |

Структура PACK:
```
manifest.json     ← тот же JSON, что в TEXT, но notes обязателен
audio.ogg         ← обязателен
cover.webp        ← опционально, 512×512
preview.ogg       ← опционально, ≤ 15 сек
```
Аудио в ZIP хранить **без сжатия** (`STORED`) — быстрее чтение.

## 3.2. Полная JSON-схема

```json
{
  "format": "mwt",
  "version": 1,
  "meta": {
    "id": "neon_sakura",
    "title": "Neon Sakura",
    "artist": "DJ Banana",
    "author": "chartmaker",
    "bpm": 170.0,
    "offsetMs": 45,
    "durationMs": 128500,
    "difficulty": 8,
    "goldMode": "MANDATORY",
    "allowCasualSimplification": true,
    "theme": "NEON",
    "audioRef": "audio.ogg",
    "coverRef": "cover.webp",
    "previewRef": "preview.ogg",
    "tags": ["anime", "fast"]
  },
  "grid": {
    "visible": "1/4",
    "hiddenSubdivisions": ["1/8", "1/16", "1/32"],
    "minGapMs": 70,
    "maxSimultaneousNotes": 4
  },
  "requiredNoteTypes": ["TAP", "HOLD", "SWIPE"],
  "notes": [
    { "id": 1, "beat": 16.0, "lane": 0, "type": "TAP" },
    { "id": 2, "beat": 16.5, "lane": 1, "type": "HOLD", "durationBeats": 2.0 },
    { "id": 3, "beat": 20.0, "lane": 3, "type": "SWIPE", "direction": "RIGHT" },
    { "id": 4, "beat": 24.0, "lane": 0, "type": "HOLD_MOVE",
      "durationBeats": 4.0, "path": [0, 1, 2] }
  ],
  "endless": {
    "usableAsBlock": true,
    "intensity": 6,
    "blockType": "FULL_TRACK",
    "segmentStartMs": null,
    "segmentEndMs": null,
    "transitionMs": 400
  }
}
```

## 3.3. Правила полей

| Поле | Тип | Обяз. | Валидация / default |
|---|---|---|---|
| `format` | string | да | ровно `"mwt"` |
| `version` | int | да | `1`. Больше — отклонить с сообщением |
| `meta.id` | string | да | `[a-z0-9_]{3,40}`, уникален |
| `meta.bpm` | double | да | `20.0 … 400.0` |
| `meta.offsetMs` | int | нет | `-2000 … 2000`, default `0` |
| `meta.durationMs` | long | да | `> 0`; сверять с реальной длиной аудио ±250 мс |
| `meta.difficulty` | int | да | `1 … 15` |
| `meta.goldMode` | enum | нет | `OFF` \| `BONUS` \| `MANDATORY`, default `OFF` |
| `meta.allowCasualSimplification` | bool | нет | default `true` |
| `meta.theme` | enum | нет | `NEON`(default), `CLASSIC`, `SPACE`, `INFERNO`, `SAKURA`, `CYBER` |
| `grid.minGapMs` | int | нет | default `70`, min `30` |
| `requiredNoteTypes` | array | нет | default `["TAP","HOLD"]` |
| `notes[].beat` | double | да | `≥ 0`, точность до 1/192 |
| `notes[].lane` | int | да | `0..3` |
| `notes[].durationBeats` | double | для HOLD-типов | `> 0` |
| `notes[].path` | int[] | для `*_MOVE` | ≥ 2 элементов, каждый `0..3`, первый = `lane` |
| `notes[].direction` | enum | для SWIPE-типов | `LEFT/RIGHT/UP/DOWN` |
| `endless` | object | нет | если отсутствует — уровень не участвует в Endless |

## 3.4. Перевод времени (единственная формула)
```kotlin
fun beatToMs(beat: Double, bpm: Double, offsetMs: Int): Double =
    offsetMs + beat * (60_000.0 / bpm)
```
Ноты **всегда** хранятся в долях. Миллисекунды вычисляются при загрузке в `RuntimeNote.startMs / endMs`.

---

# 4. Модель данных

```kotlin
enum class NoteType {
    TAP, HOLD, SWIPE, GOLD,
    SWIPE_HOLD, HOLD_SWIPE, HOLD_WITH_SWIPE,
    HOLD_MOVE, HOLD_MOVE_SWIPE,
    DOUBLE_TAP, MULTI_HOLD, FLICK
}
enum class SwipeDirection { LEFT, RIGHT, UP, DOWN }
enum class GoldMode { OFF, BONUS, MANDATORY }
enum class NoteState { PENDING, ACTIVE, JUDGED, FAILED, SIMPLIFIED }
enum class HitGrade { PERFECT, GOOD, MISS }
enum class PlayMode { CAMPAIGN, ENDLESS, PRACTICE, CUSTOM }
enum class RecordCategory { NORMAL, CASUAL, TRUE_HARDCORE }
enum class LevelSource { OFFICIAL, CUSTOM }

@Serializable
data class Note(
    val id: Int = 0,
    val beat: Double,
    val lane: Int,
    val type: NoteType,
    val durationBeats: Double = 0.0,
    val path: List<Int> = emptyList(),
    val direction: SwipeDirection? = null
)

/** Создаётся один раз при загрузке, переиспользуется между попытками. */
class RuntimeNote(val def: Note) {
    var startMs = 0.0
    var endMs = 0.0
    var state = NoteState.PENDING
    var effectiveType = def.type       // после CasualSimplifier
    var pointerId = -1
    var holdBrokenAtMs = -1.0          // для Hold Assist
    var pathIndex = 0
    var judgedGrade: HitGrade? = null
    fun reset() { /* сброс всех полей */ }
}
```

**Требование:** массив `RuntimeNote[]` аллоцируется при загрузке уровня и переиспользуется. В кадре — ноль аллокаций.

---

# 5. Аудио и часы (критический раздел)

## 5.1. Интерфейс
```kotlin
interface AudioEngine {
    suspend fun prepare(source: AudioSource): Result<Unit>
    fun start(fromMs: Double = 0.0)
    fun pause()
    fun resume()
    fun stop()
    fun seek(ms: Double)

    /** MASTER CLOCK. Позиция воспроизведения в мс. */
    fun positionMs(): Double

    /** Якорь для перевода времени тача в аудио-время. */
    fun clockAnchor(): ClockAnchor

    fun isPlaying(): Boolean
    val outputLatencyMs: Int
}

data class ClockAnchor(val audioMs: Double, val uptimeMs: Long)
```

## 5.2. Реализация OboeEngine
- Декодировать OGG → PCM (float, stereo, 44100) через `MediaCodec` в фоне при загрузке уровня.
- Воспроизведение: Oboe `AudioStream`, `PerformanceMode::LowLatency`, `SharingMode::Exclusive`, `AudioApi::AAudio` (API ≥ 27) иначе OpenSL ES.
- `positionMs() = framesPlayed * 1000.0 / sampleRate`.
- `clockAnchor()` возвращается из audio-callback: пара (текущая позиция, `SystemClock.uptimeMillis()`), обновляется каждый callback через атомарную ссылку.
- Для треков > 6 минут — стриминговое декодирование с буфером 5 секунд.

## 5.3. Перевод времени касания — обязательно
`MotionEvent.eventTime` — это **uptimeMillis**, а не аудио-время. Прямое сравнение с позицией плеера = баг.

```kotlin
fun touchToAudioMs(event: MotionEvent, anchor: ClockAnchor, touchOffsetMs: Int): Double {
    val uptime = event.eventTime                      // uptimeMillis
    val audioAtTouch = anchor.audioMs + (uptime - anchor.uptimeMs)
    return audioAtTouch - touchOffsetMs               // калибровка
}
```
Для `ACTION_MOVE` использовать также `getHistoricalEventTime(i)` / `getHistoricalX(i)` — иначе теряется точность на 120 Гц.

## 5.4. Игровое время кадра
```
audioNow = audioEngine.positionMs() - settings.audioOffsetMs
renderTime = audioNow + settings.visualOffsetMs
```
Судейство использует время касания (переведённое), рендер — `renderTime`.

## 5.5. Контроль дрейфа
Каждые 5 секунд сверять `positionMs()` с ожидаемым по uptime. Расхождение > 40 мс → пересинхронизация якоря + запись в лог. За 5-минутный трек накопленный дрейф ≤ 15 мс.

## 5.6. Прерывания
| Событие | Реакция |
|---|---|
| Потеря аудиофокуса (звонок) | Авто-пауза, экран Pause |
| Возврат фокуса | НЕ автовозобновление. Экран «Продолжить» + отсчёт 3-2-1 |
| Уход в фон (`onStop`) | Пауза, аудио стоп, сброс активных холдов без штрафа |
| Смена устройства вывода (наушники) | Пауза + предложение перекалибровки |
| Bluetooth-гарнитура | При старте уровня показать предупреждение о задержке + кнопку «Калибровка» |

## 5.7. Форматы контента
- Музыка: **OGG Opus** (предпочтительно) или **OGG Vorbis**, 44100 Hz, stereo, 96–160 kbps.
- SFX: короткие OGG/WAV в resources, проигрывать через `SoundPool` (отдельно от основного потока, не влияет на master clock).

---

# 6. Константы геймплея (`GameConstants.kt`)

## 6.1. Судейство

| Константа | Обычный режим | True Hardcore |
|---|---|---|
| `PERFECT_WINDOW_MS` | 50 | 50 |
| `GOOD_WINDOW_MS` | 120 | 120 |
| `HOLD_TOLERANCE_PERCENT` | 10 | 5 |
| `HOLD_MIN_TOLERANCE_MS` | 80 | 50 |
| `HOLD_MAX_TOLERANCE_MS` | 300 | 150 |
| `HOLD_ASSIST_GRACE_MS` | 150 | 0 (выкл) |
| `HOLD_DRIFT_TOLERANCE_LANES` | 0.6 | 0.6 |
| `HOLD_DRIFT_TIME_MS` | 100 | 100 |
| `SWIPE_MIN_DISTANCE_DP` | 24 | 24 |
| `SWIPE_MAX_DURATION_MS` | 200 | 200 |
| `DOUBLE_TAP_SYNC_MS` | 80 | 80 |

**Обоснование минимума 50 мс:** экраны опрашивают сенсор с частотой 60–120 Гц (шаг 8.3–16.7 мс). Окно уже 50 мс даёт менее 3–6 сэмплов и начинает наказывать за железо, а не за игру.

## 6.2. Скорость падения

| scrollSpeed (настройка) | approachMs (время падения от верха до линии) |
|---|---|
| 1 | 1200 |
| 2 | 1050 |
| 3 | 900 |
| 4 | 800 |
| **5 (default)** | **700** |
| 6 | 600 |
| 7 | 520 |
| 8 | 450 |
| 9 | 380 |
| 10 | 320 |

**Скорость падения не влияет на судейство.** Меняется только визуальный подход.

## 6.3. Геометрия
```
laneWidth   = screenWidth / 4
hitLineY    = screenHeight * 0.82
noteHeight  = laneWidth * 0.55   (для TAP)
spawnY      = -noteHeight
```
Позиция ноты: `y = hitLineY - (startMs - renderTime) / approachMs * (hitLineY - spawnY)`
Ноты рендерить, если `startMs - renderTime ∈ [-250, approachMs + 250]`.

## 6.4. Очки

| Событие | База |
|---|---|
| PERFECT | 300 |
| GOOD | 100 |
| Завершение HOLD | +300 |
| Завершение составной ноты (SWIPE_HOLD и т.п.) | +500 |
| GOLD попадание | база × 3 + 1 монета |

Множитель комбо:
| Комбо | Множитель |
|---|---|
| 0–9 | ×1.0 |
| 10–29 | ×1.2 |
| 30–49 | ×1.5 |
| 50–99 | ×2.0 (Fever активен) |
| 100+ | ×2.5 |

```
accuracy = (perfectCount + goodCount * 0.6) / judgedCount * 100
```

## 6.5. Ранги и звёзды

| Ранг | Условие |
|---|---|
| SS | accuracy = 100, все PERFECT, без продолжений |
| S | accuracy = 100, без продолжений |
| A | accuracy ≥ 97 |
| B | accuracy ≥ 90 |
| C | accuracy ≥ 80 |
| D | уровень пройден |

Звёзды: 1 — пройден; 2 — accuracy ≥ 90; 3 — accuracy ≥ 97.

## 6.6. Экономика

| Действие | Монеты |
|---|---|
| Прохождение уровня | +50 |
| За каждую звезду | +25 |
| GOLD нота | +1 |
| Ежедневная награда (день 1…7) | 50 / 60 / 75 / 100 / 125 / 150 / 200 |
| Задание | 30–150 |
| Сундук | 50–300 (детерминированно по прогрессу, не рандом) |
| **Продолжение после смерти** | **−100** |

Стартовый баланс: 300 монет.

---

# 7. Судейство (Judge) — точная логика

## 7.1. TAP
```
на ACTION_DOWN(lane, touchAudioMs):
  candidates = ноты этого lane, state=PENDING, effectiveType=TAP-подобный
  candidate = argmin |touchAudioMs - startMs|
  если candidate == null ИЛИ |delta| > GOOD_WINDOW_MS:
      FAIL(EMPTY_TAP)
  иначе:
      grade = PERFECT если |delta| <= 50 иначе GOOD
      note.state = JUDGED; combo++; score += ...
```

## 7.2. HOLD
```
tolerance = clamp(durationMs * pct/100, min, max)

START (ACTION_DOWN):
  как TAP. При промахе → FAIL(HOLD_START_MISS)
  успех → state=ACTIVE, pointerId=id

DURING (каждый кадр):
  если палец ушёл с дорожки более чем на 0.6*laneWidth дольше 100 мс:
      FAIL(HOLD_DRIFT)

RELEASE (ACTION_UP) в момент t:
  если t < endMs - tolerance:
      если HoldAssist включён:
          holdBrokenAtMs = t; ждём возврата пальца
          если ACTION_DOWN на той же дорожке в течение 150 мс → продолжаем ACTIVE
          иначе → FAIL(HOLD_BREAK)
      иначе → FAIL(HOLD_BREAK)
  если endMs - tolerance <= t <= endMs + tolerance:
      SUCCESS, grade=PERFECT
  (t > endMs + tolerance обработается автоматом ниже)

AUTO (каждый кадр, если ACTIVE и renderTime > endMs + tolerance):
  FAIL(HOLD_OVERRUN)

AUTO-COMPLETE: если палец ещё держится в момент endMs,
  нота визуально "завершается" и ждёт релиза в пределах tolerance.
```

## 7.3. SWIPE / FLICK
```
ACTION_DOWN в окне → state=ACTIVE, ждём жест
к моменту startMs + SWIPE_MAX_DURATION_MS:
  если смещение >= 24dp в требуемом направлении → SUCCESS (grade по времени DOWN)
  если направление неверное → FAIL(SWIPE_WRONG)
  если время вышло → FAIL(SWIPE_TIMEOUT)
```
Направление определяется по доминирующей оси смещения.

## 7.4. Составные типы

| Тип | Логика |
|---|---|
| `SWIPE_HOLD` | Свайп в окне старта → сразу HOLD длиной `durationBeats` от момента свайпа |
| `HOLD_SWIPE` | HOLD, в конце вместо релиза требуется свайп в окне `endMs ± tolerance` |
| `HOLD_WITH_SWIPE` | HOLD, в середине (`startMs + duration/2`) требуется свайп, окно ±120 мс |
| `HOLD_MOVE` | HOLD; путь `path` делится на равные отрезки по времени; в каждом отрезке палец должен быть на целевой дорожке (допуск перехода ±120 мс) |
| `HOLD_MOVE_SWIPE` | `HOLD_MOVE` + свайп в конце |
| `DOUBLE_TAP` | Две ноты с одинаковым `beat`, разные дорожки. Обе должны быть нажаты, разброс между нажатиями ≤ 80 мс, иначе FAIL |
| `MULTI_HOLD` | ≥2 одновременных HOLD, каждый своим пальцем. Логика каждого независима |
| `GOLD` | Как TAP. При `goldMode=BONUS` пропуск **не** убивает (нота исчезает, комбо не растёт, но не сбрасывается). При `MANDATORY` — как обычный TAP |

## 7.5. Авто-MISS
```
каждый кадр: для всех PENDING где renderTime > startMs + GOOD_WINDOW_MS:
    если GOLD и goldMode == BONUS → note.state = JUDGED (без штрафа)
    иначе → FAIL(NOTE_MISSED)
```

## 7.6. Мультитач
- `PointerTracker` ведёт массив на 10 слотов: `pointerId → (lane, x, y, downTimeMs, activeNote)`.
- Обрабатывать `ACTION_POINTER_DOWN` / `ACTION_POINTER_UP` через `actionIndex`, не через `actionMasked` в одиночку.
- `ACTION_CANCEL` трактовать как релиз всех пальцев **с применением Hold Assist**, если он включён (системный жест не должен наказывать).

---

# 8. Смерть, продолжение, True Hardcore

## 8.1. FailReason
`NOTE_MISSED, EMPTY_TAP, HOLD_START_MISS, HOLD_BREAK, HOLD_OVERRUN, HOLD_DRIFT, SWIPE_WRONG, SWIPE_TIMEOUT, DOUBLE_TAP_DESYNC`

## 8.2. Реакция на FAIL
1. Мгновенно остановить аудио с эффектом pitch-down 250 мс.
2. Красная вспышка 120 мс, вибрация 60 мс (если включена).
3. Заморозить кадр на 300 мс, показать экран смерти.

## 8.3. Продолжение (обычный режим)

| Параметр | Значение |
|---|---|
| Лимит | **1 раз за попытку уровня** |
| Способ 1 | 100 монет |
| Способ 2 | «Награда (заглушка)» — экран с таймером **5 секунд**, затем кнопка «Продолжить» |
| Точка возврата | `failTimeMs − 1500` |
| Подготовка | Отсчёт **3-2-1 (2 сек)**, аудио стартует с точки возврата |
| Счёт | Сохраняется |
| Комбо | Сбрасывается в 0 |
| Ноты в откате | Сбрасываются в PENDING, но помечаются `noScore=true` (повторно очки не дают) |
| Флаг результата | `usedContinue = true` (ранг S/SS недоступен) |

Никаких рекламных SDK. Заглушка — локальный экран.

## 8.4. True Hardcore (переключатель в настройках)

**Отключает:**
- продолжение за монеты и за таймер
- Casual-упрощение
- Hold Assist
- разбивку длинных холдов
-Practice-подсказки во время игры

**Ужесточает:** окно холда до 5% / 50–150 мс.

**Сохраняет:** окна TAP (50/120 мс), калибровку, честную обработку мультитача. Игра жёсткая, но не несправедливая.

**Блокирует запуск**, если требуемые типы нот выключены:
```
Для True Hardcore уровень проходится только с оригинальными типами нот.
Требуется включить: SWIPE, HOLD_MOVE
[Включить и играть]  [Назад]
```

---

# 9. Casual-режим

## 9.1. Предстартовая проверка
```kotlin
val missing = level.requiredNoteTypes - settings.enabledNoteTypes
when {
    missing.isEmpty() -> запуск NORMAL
    settings.trueHardcore -> БЛОК + окно включения
    !level.meta.allowCasualSimplification -> БЛОК + окно включения
    else -> окно выбора
}
```

Окно выбора:
```
Этот уровень использует: SWIPE, HOLD_MOVE

[Включить и играть]   → NORMAL
[Играть в Casual]     → CASUAL
[Назад]
```

## 9.2. Таблица упрощения (детерминированная)

| Оригинал | Замена |
|---|---|
| `TAP`, `HOLD` | без изменений |
| `SWIPE`, `FLICK` | `TAP` |
| `GOLD` | `TAP` (монеты не дают) |
| `SWIPE_HOLD`, `HOLD_SWIPE`, `HOLD_WITH_SWIPE` | `HOLD` той же длительности |
| `HOLD_MOVE` | `HOLD` на стартовой дорожке |
| `HOLD_MOVE_SWIPE` | `HOLD` на стартовой дорожке |
| `DOUBLE_TAP` | `TAP` на дорожке с меньшим индексом, вторая удаляется |
| `MULTI_HOLD` | самый длинный `HOLD`, остальные удаляются |

Замена **не сдвигает** `beat`. Удалённые ноты не учитываются в `judgedCount`.

## 9.3. Статус «КАЗУАЛ ГЕЙМЕР»
- Присваивается при первом Casual-прохождении, **перманентно**, снятию не подлежит.
- Хранится в `profile.casualBadge = true` + `casualSince = timestamp`.
- Отображается: в профиле, на экране результатов, рядом с рекордами.
- **Не является наказанием:** не блокирует контент, не отнимает валюту, не мешает прогрессу.
- Смысл — сохранение престижа чистых рекордов.

## 9.4. Раздельные рекорды
Ключ: `levelId + playMode + recordCategory`.
`NORMAL`, `CASUAL`, `TRUE_HARDCORE` — три независимые таблицы. Casual-результат **никогда** не перезаписывает NORMAL/TH.

## 9.5. Разбивка длинных холдов (accessibility)

| Параметр | Значение |
|---|---|
| Настройка | `longHoldBreakEnabled`, default `false` |
| Порог | `longHoldThresholdMs = 3000` |
| Длина сегмента | 2500 мс |
| Зазор между сегментами | 120 мс |
| Доступность | `PRACTICE`, `CUSTOM` — свободно. `CAMPAIGN`/`ENDLESS` — принудительная категория `CASUAL` |
| True Hardcore | недоступна, настройка игнорируется |

Обоснование: дешёвые тачскрины срывают длинные удержания. Это accessibility, а не чит.

---

# 10. Режимы игры

## 10.1. Кампания
Файл `assets/campaign.json`:
```json
{
  "chapters": [
    { "id": "ch1", "title": "Пробуждение",
      "levelIds": ["tut_01","tut_02","easy_01","easy_02","boss_01"] }
  ]
}
```
- Уровни открываются последовательно: следующий при ≥1 звезде на предыдущем.
- Последний уровень главы — босс (difficulty +2 к среднему по главе).
- Прогресс: `Map<levelId, {stars, bestScore, bestAccuracy, cleared}>`.

## 10.2. Бесконечный режим (детерминированный)

Файл `assets/endless_main.json`:
```json
{
  "playlistId": "endless_main",
  "blocks": [
    { "levelId":"easy_01", "intensity":1, "blockType":"SHORT_SEGMENT",
      "segmentStartMs":0, "segmentEndMs":30000, "transitionMs":400 },
    { "levelId":"mid_01",  "intensity":3, "blockType":"SHORT_SEGMENT",
      "segmentStartMs":15000, "segmentEndMs":45000, "transitionMs":400 },
    { "levelId":"full_01", "intensity":5, "blockType":"FULL_TRACK", "transitionMs":600 },
    { "levelId":"boss_01", "intensity":8, "blockType":"FULL_TRACK", "transitionMs":800 }
  ],
  "loopRules": {
    "scrollMultiplierPerLoop": 1.05,
    "maxScrollMultiplier": 1.5,
    "intensityShiftPerLoop": 1,
    "scoreMultiplierPerLoop": 0.1,
    "maxScoreMultiplier": 2.0
  }
}
```

**EndlessDirector:**
```
step = 0, 1, 2, ...
index = step % blocks.size
loop  = step / blocks.size
block = blocks[index]

effectiveScroll = baseApproachMs / min(1.05^loop, 1.5)
scoreMultiplier = min(1.0 + 0.1*loop, 2.0)
```
- Порядок **строго последовательный**, без shuffle.
- Переход между блоками — кроссфейд аудио `transitionMs`, ноты следующего блока появляются заранее.
- Аудио **не ускоряется никогда**.
- Смерть = конец забега. Результат: суммарные очки, число блоков, номер цикла, макс. комбо.
- Продолжение в Endless: **1 на весь забег**, теми же способами.

## 10.3. Практика
Без смерти (мисс подсвечивается, но не убивает), скорость аудио 0.5–1.0×, разбивка холдов разрешена, рекорды и награды не сохраняются.

## 10.4. Свои уровни
Отдельная вкладка, `LevelSource.CUSTOM`, свои рекорды, никаких официальных наград/достижений.

---

# 11. Редактор уровней

## 11.1. Разблокировка
Настройки → пункт «Версия 1.0.0». **10 нажатий**, разрыв между нажатиями ≤ 3000 мс (иначе счётчик сбрасывается). На 7-м нажатии — тост «Ещё 3…». После 10-го — постоянный флаг `editorUnlocked = true` и новый пункт меню «Редактор уровней».

Редактор **не даёт**: валюту, читы, доступ к закрытым уровням, правку прогресса.

## 11.2. Обязательные функции

| Функция | Требование |
|---|---|
| Загрузка аудио | SAF, OGG/MP3/WAV |
| Волновая форма | Даунсэмпл до 2000 пиков на экран, кэш в файле |
| BPM / offset | Ручной ввод + «Tap BPM» (усреднение по 8 тапам, медиана) |
| Видимая сетка | 1/1, 1/2, 1/4, 1/8, 1/16 (переключатель) |
| Скрытые деления | 1/32, 1/64 для квантования |
| Квантование | При постановке — привязка к ближайшему скрытому делению; тумблер «Free» отключает |
| Инструменты | TAP, HOLD (растягивание), SWIPE (выбор направления), GOLD, DELETE, SELECT |
| Мультиселект | Рамка, перемещение, копирование, вставка, зеркало по дорожкам |
| Undo/Redo | Стек 100 операций |
| Зум | 0.25×–8× по времени |
| Playback | Старт с курсора, скорости 0.25/0.5/0.75/1.0 |
| Test Play | Запуск в реальном движке с текущей позиции |
| Автосохранение | Каждые 30 с + при уходе с экрана, файл `editor_draft.json` |
| Восстановление | После краша предложить восстановить черновик |
| Экспорт | `.mwt` TEXT и PACK через `ACTION_CREATE_DOCUMENT` / Share |

## 11.3. Tap Record Mode
1. Задать BPM и offset.
2. Нажать «Запись» → обратный отсчёт 4 доли → играет музыка.
3. Поверх — 4 невидимые зоны дорожек.
4. `ACTION_DOWN` → фиксируем `beat` через аудио-часы.
5. `ACTION_UP`: удержание > 200 мс → `HOLD` с длительностью; иначе `TAP`.
6. Смещение ≥ 24 dp за ≤ 200 мс → `SWIPE` с направлением.
7. По окончании — авто-квантование всех нот к ближайшему скрытому делению.
8. Отчёт: «Записано N нот, скорректировано M».
9. Результат открывается в редакторе сетки для доводки.

## 11.4. Валидатор

**Ошибки (блокируют экспорт):**
перекрытие нот на одной дорожке; `lane ∉ 0..3`; `durationBeats ≤ 0` у HOLD; SWIPE без direction; `path[0] ≠ lane` или путь короче 2; нота за пределами `durationMs`; отсутствие аудио; `bpm ≤ 0`; дублирующийся `id` уровня.

**Предупреждения (экспорт разрешён):**
зазор < `minGapMs`; одновременных нот > `maxSimultaneousNotes`; плотность > 12 нот/сек при difficulty ≤ 5; тишина в первые 2 с; более 3 сек без нот в середине трека.

UI: список проблем с таймкодами, тап по проблеме → переход курсора к месту.

---

# 12. Настройки (полный список)

| Раздел | Пункт | Тип | Default |
|---|---|---|---|
| **Звук** | Громкость музыки | 0–100 | 80 |
| | Громкость эффектов | 0–100 | 70 |
| | Звуки интерфейса | bool | true |
| | Вибрация | bool | true |
| **Геймплей** | Скорость падения | 1–10 | 5 |
| | Калибровка аудио | −200…+200 мс | 0 |
| | Калибровка визуала | −200…+200 мс | 0 |
| | Калибровка тача | −200…+200 мс | 0 |
| | Автокалибровка | кнопка | — |
| | Hold Assist | bool | true |
| | Разбивка длинных холдов | bool | false |
| | Порог разбивки | 2000–5000 мс | 3000 |
| | **True Hardcore** | bool | **false** |
| **Типы нот** | TAP, HOLD | залочено ON | on |
| | SWIPE | bool | false |
| | GOLD | bool | false |
| | DOUBLE_TAP | bool | false |
| | SWIPE_HOLD / HOLD_SWIPE | bool | false |
| | HOLD_MOVE | bool | false |
| | HOLD_MOVE_SWIPE | bool | false |
| | MULTI_HOLD | bool | false |
| | FLICK | bool | false |
| **Графика** | Качество эффектов | LOW/MED/HIGH | MED |
| | Частицы | bool | true |
| | Тряска экрана | bool | true |
| | Пульсация фона | bool | true |
| | Лимит FPS | 60/120/auto | auto |
| **Доступность** | Режим дальтоника | bool | false |
| | Контрастные ноты | bool | false |
| | Уменьшенные эффекты | bool | false |
| **Прочее** | Язык | RU/EN | по системе |
| | Экспорт/импорт сейва | кнопка | — |
| | Сброс прогресса | кнопка + подтверждение | — |
| | Версия | 10 тапов → редактор | — |

## 12.1. Автокалибровка
Метроном 100 BPM, 16 ударов. Первые 4 отбрасываются. Собираем дельты `touchAudio − tickTime`, отбрасываем выбросы > 150 мс, берём **медиану**. Если MAD > 60 мс — «Результат нестабилен, повторите». Иначе предложить применить значение в `audioOffsetMs`.

---

# 13. Прогресс и хранение

## 13.1. Файлы
```
files/
├── save/
│   ├── profile.json
│   ├── records.json
│   ├── settings.json
│   ├── unlocks.json
│   ├── wallet.json
│   └── meta.json          (ежедневки, задания, сундуки)
├── mwt/
│   ├── official/          (из assets при первом запуске)
│   ├── custom/            (созданные в редакторе)
│   └── imported/          (импорт извне)
├── editor/
│   └── editor_draft.json
└── cache/                 (волны, обложки)
```
Запись атомарная: `write to .tmp → fsync → rename`. При старте — проверка checksum, при повреждении откат на `.bak`.

## 13.2. Схемы

```json
// profile.json
{ "version":1, "nickname":"Player", "xp":0, "playerLevel":1,
  "casualBadge":false, "casualSince":null,
  "editorUnlocked":false, "createdAt":0, "totalPlayMs":0,
  "titlesOwned":["novice"], "titleEquipped":"novice" }
```
```json
// records.json
{ "version":1, "records":[
  { "levelId":"easy_01", "mode":"CAMPAIGN", "category":"NORMAL",
    "score":128400, "accuracy":98.2, "maxCombo":312, "rank":"A",
    "stars":3, "usedContinue":false, "achievedAt":1730000000000 } ] }
```
```json
// unlocks.json
{ "version":1, "unlockedLevels":["tut_01","tut_02"],
  "unlockedFeatures":["play","settings"],
  "ownedSkins":["default"], "ownedThemes":["NEON"],
  "achievements":["FIRST_STEP"] }
```

## 13.3. График открытия фич (UnlockGraph)

| Условие | Открывается |
|---|---|
| Старт | Играть, настройки, туториал (2 уровня) |
| 3 уровня пройдено | Список песен, рекорды, профиль |
| 5 уровней | Ежедневная награда, серия входов, ежедневные задания |
| 7 уровней | Бесконечный режим |
| 10 уровней | Достижения, магазин, скины |
| 15 уровней | Сундуки, коллекции, еженедельные задания |
| 20 уровней | Локальные события, сезоны, титулы |
| 10 тапов по версии | Редактор (в любой момент) |

---

# 14. Мета-системы (удержание без давления)

## 14.1. Обязательно реализовать
Ежедневная награда, серия входов, ежедневные задания (3/день), еженедельные (5/неделю), достижения (30+), сундуки, магазин скинов, коллекции, локальные сезоны, титулы.

## 14.2. Запрещено
Таймеры ожидания, энергия/жизни, «сундук откроется через 2 часа», FOMO-давление, всплывающие окна чаще 1 за сессию, пуш-уведомления.

## 14.3. Правила
- **Сундуки открываются мгновенно.** Награда детерминирована прогрессом (не `Random`).
- **Серия входов мягкая:** пропуск не наказывает, серия просто начинается заново. Есть предмет «заморозка серии» (1 бесплатно в неделю).
- **Локальные сезоны** привязаны к дате устройства. Смену даты не отслеживать и не наказывать — просто флаг «сезон пройден».
- Магазин: только косметика (скины нот, темы, эффекты попаданий, эффекты Fever, титулы, рамки, аватары). Ничего, влияющего на сложность или очки.

## 14.4. Примеры достижений
`FIRST_STEP` (1 уровень), `COMBO_100`, `COMBO_500`, `PERFECT_LEVEL` (100% perfect), `NO_CONTINUE_10` (10 уровней без продолжений), `BPM_180` (пройти трек ≥180 BPM), `ENDLESS_10_BLOCKS`, `ENDLESS_LOOP_3`, `MADMAN` (включить все типы нот), `CREATOR` (создать уровень в редакторе), `COLLECTOR_10` (10 скинов), `TRUE_HARDCORE_1`.

---

# 15. UI/UX

## 15.1. Экраны
`Splash → Home → SongList → PreLevel → Countdown → Gameplay → (Dead → Continue?) → Result → Home`
Ветки: `Settings (+Calibration)`, `Profile`, `Shop`, `Missions`, `Achievements`, `Endless`, `CustomLevels`, `Editor`, `Practice`.

## 15.2. Home
Крупная кнопка ИГРАТЬ (≥ 40% ширины), баланс монет, аватар + титул (включая «КАЗУАЛ ГЕЙМЕР»), нижняя навигация (Дом / Песни / Магазин / Настройки), баннер события — только после разблокировки. Тёмный фон, неоновые акценты, пульсация в такт превью.

## 15.3. SongList
Карточки: обложка, название, исполнитель, difficulty (цвет + число), звёзды, лучший счёт, бейдж категории рекорда, иконки требуемых типов нот, замок если закрыт. Фильтры: все / пройденные / непpойденные / свои.

## 15.4. Gameplay HUD
Сверху: счёт, комбо (крупно по центру), точность, тонкий прогресс-бар трека. Fever-индикатор. Кнопка паузы (левый верх, 48dp).
**HUD не перекрывает верхние 25% зоны падения нот.**

## 15.5. Эффекты (бюджеты)

| Эффект | LOW | MED | HIGH |
|---|---|---|---|
| Частиц на попадание | 4 | 10 | 20 |
| Тряска экрана | выкл | ≤4 dp / 100 мс | ≤6 dp / 120 мс |
| Вспышка дорожки | 60 мс | 80 мс | 100 мс |
| Фон-пульсация | выкл | вкл | вкл |

Авто-понижение качества на ступень при FPS < 50 в течение 2 секунд, с записью в лог.

## 15.6. Result
Крупный анимированный ранг, звёзды поочерёдно, счёт, точность, макс. комбо, breakdown PERFECT/GOOD, начисленные монеты, бейдж категории (NORMAL/CASUAL/TRUE_HARDCORE), конфетти при новом рекорде. Кнопки: Ещё раз / Дальше / Домой.

## 15.7. Темы
`NEON` (default), `CLASSIC`, `SPACE`, `INFERNO`, `SAKURA`, `CYBER`.
Тема влияет на фон, цвет нот, частицы. **Не влияет** на тайминги. Контраст линии удара к фону ≥ 4.5:1 в любой теме.

---

# 16. Импорт/экспорт и безопасность

| Операция | Реализация |
|---|---|
| Импорт `.mwt` | `ACTION_OPEN_DOCUMENT`, mime `*/*`, копирование в `files/mwt/imported/` |
| Экспорт `.mwt` | `ACTION_CREATE_DOCUMENT` или FileProvider + Share |
| Экспорт сейва | ZIP всех save-файлов через SAF |

**Лимиты:** файл ≤ 60 МБ, аудио ≤ 10 мин, нот ≤ 20000, ZIP-энтри ≤ 20, распакованный размер ≤ 120 МБ.

**Безопасность:** `.mwt` — только данные. Запрещено исполнение скриптов, zip-slip (пути с `../`), симлинки, вложенные архивы, любые обращения к сети из контента. Битый файл → понятная ошибка в UI, приложение живо.

**Разрешения в манифесте:** только `VIBRATE`, `WAKE_LOCK` (на время игры), `MODIFY_AUDIO_SETTINGS`. Ни памяти, ни сети, ни телефона.

---

# 17. Производительность

| Метрика | Требование |
|---|---|
| End-to-end задержка (тач → визуальный отклик) | ≤ 80 мс на устройстве среднего класса |
| Аудиозадержка (Oboe LowLatency) | ≤ 40 мс |
| FPS | ≥ 60 стабильно при 2000+ нот на экране-треке |
| Аллокации в кадре | 0 байт (проверять Profiler) |
| Cold start до Home | ≤ 2.5 с |
| Загрузка уровня (2000 нот + OGG 3 мин) | ≤ 1.5 с |
| Память peak | ≤ 350 МБ |
| Дрейф тайминга за 5 мин | ≤ 15 мс |

---

# 18. Тестирование

## 18.1. Unit-тесты (обязательны)
- `beatToMs` при разных BPM/offset, включая дробные BPM
- `Judge`: границы окон 49/50/51 мс, 119/120/121 мс
- `holdToleranceMs`: 0.3 с → 80 мс; 1 с → 100 мс; 4 с → 300 мс; TH 4 с → 150 мс
- `CasualSimplifier`: все 12 типов, бит не сдвигается
- `MwtValidator`: все ошибки и предупреждения из 11.4
- `EndlessDirector`: 100 шагов детерминированы, scroll не превышает 1.5
- `MwtLoader`: TEXT и PACK, битый JSON, битый ZIP, zip-slip, отсутствие аудио

## 18.2. Интеграционные
- Один трек, 10 автопрогонов с идентичным синтетическим вводом → идентичный счёт
- Дрейф на 5-минутном треке ≤ 15 мс
- Пауза/резюм 20 раз подряд → позиция не уплывает
- Hold Assist: разрыв 100 мс прощается, 200 мс — нет
- TH: разрыв 100 мс не прощается

## 18.3. Устройства
Минимум 5 устройств: 60 Гц экран, 120 Гц экран, слабый (≤3 ГБ RAM), Bluetooth-наушники, дешёвый тачскрин с пропуском удержаний.

## 18.4. Негативные
Битый `.mwt`, отсутствующее аудио, `version: 99`, 20000 нот, аудио 11 минут, файл 61 МБ, поворот экрана (заблокирован), входящий звонок, `ACTION_CANCEL` посреди холда.

---

# 19. Этапы реализации и критерии приёмки

### M0 — Аудио-ядро и отклик
Проект, Oboe-движок, `AudioClock` с якорем, `SurfaceView`, 4 дорожки, мультитач, одна TAP-нота, лог дельт.
**Приёмка:** на 2 устройствах после калибровки стабильно |delta| ≤ 50 мс; дрейф ≤ 15 мс за 3 мин; в логе нет аллокаций в кадре.

### M1 — MVP
`.mwt` TEXT + валидатор, TAP и HOLD полностью, Judge, смерть, экран результата, рекорд в файл, меню выбора (1 трек), ручная и автокалибровка.
**Приёмка:** импорт `.mwt` → игра → смерть от промаха → результат → рекорд пережил перезапуск. Битый JSON не крашит.

### M2 — Спец-ноты и Casual
SWIPE, GOLD, DOUBLE_TAP, составные типы, настройки типов, PreLevelCheck, CasualSimplifier, перманентный бейдж, 3 категории рекордов, Hold Assist, разбивка холдов.
**Приёмка:** уровень со SWIPE при выключенном SWIPE предлагает выбор; Casual-рекорд не пересекается с NORMAL; бейдж не снимается ничем.

### M3 — True Hardcore и продолжения
TH-переключатель, окно 5%/50–150, блокировка Casual в TH, continue за 100 монет и за 5-сек заглушку, откат 1500 мс + отсчёт, лимит 1.
**Приёмка:** в TH нет кнопок продолжения; в обычном continue работает ровно 1 раз и ставит `usedContinue`.

### M4 — Редактор
Скрытая разблокировка, сетка + квантование, инструменты, undo/redo, Tap Record, валидатор, автосейв, Test Play, экспорт TEXT+PACK, изоляция CUSTOM.
**Приёмка:** уровень, созданный тап-рекордом, экспортирован → импортирован обратно → проходим; официальные рекорды не изменились.

### M5 — Бесконечный режим
`endless_main.json`, EndlessDirector, сегменты + полные треки, кроссфейды, циклы с эскалацией, рекорд забега.
**Приёмка:** два прогона с одинаковым вводом дают идентичную последовательность блоков; аудио не ускоряется.

### M6 — Мета
Монеты, магазин, скины, темы, ежедневка, задания, достижения, сундуки, сезоны, UnlockGraph, прогрессивное открытие UI.
**Приёмка:** новый аккаунт видит только Играть/Настройки; после 5 уровней появляется ежедневка; ни одного таймера ожидания.

### M7 — Полировка
Эффекты, вибрация, темы, доступность, оптимизация, RU/EN, тесты на 5 устройствах, экспорт сейва.
**Приёмка:** все метрики раздела 17 выполнены, чек-лист раздела 20 закрыт.

---

# 20. Финальный чек-лист приёмки

- [ ] Судейство выводится из аудио-часов; `eventTime` конвертируется через anchor
- [ ] Ни одного `Random` в генерации нот и Endless (проверка grep)
- [ ] Ноты хранятся в долях, не в миллисекундах
- [ ] Пустой тап убивает; промах фиксируется по истечении GOOD-окна
- [ ] HOLD: старт / дрифт / обрыв / релиз / оверран — все пять случаев обработаны
- [ ] Окно холда 10% (80–300) обычно и 5% (50–150) в TH
- [ ] Hold Assist работает и выключается в TH
- [ ] Casual-бейдж перманентен; 3 категории рекордов изолированы
- [ ] Разбивка холдов недоступна в TH и не пишет NORMAL-рекорды
- [ ] Continue: 1 раз, монеты или 5-сек заглушка, отсутствует в TH
- [ ] Endless детерминирован; циклы ускоряют скролл, а не аудио
- [ ] Редактор: 10 тапов, изоляция, undo, автосейв, Tap Record, экспорт обоих вариантов
- [ ] PACK и TEXT `.mwt` читаются; zip-slip заблокирован
- [ ] SAF без широких разрешений; в манифесте нет INTERNET
- [ ] Пауза при потере аудиофокуса, возврат только через отсчёт
- [ ] 0 аллокаций в игровом цикле; 60 FPS при 2000 нот
- [ ] Битые файлы дают ошибку в UI, не краш
- [ ] Нет таймеров ожидания, энергии, пушей
- [ ] RU/EN вынесены в ресурсы, хардкод-строк в UI нет

---

# 21. Приложения

## 21.1. Минимальный валидный уровень для M1
```json
{
  "format": "mwt", "version": 1,
  "meta": { "id":"test_01", "title":"Test Track", "artist":"Dev",
            "bpm":120.0, "offsetMs":0, "durationMs":30000,
            "difficulty":1, "audioRef":"audio.ogg" },
  "requiredNoteTypes": ["TAP","HOLD"],
  "notes": [
    { "id":1, "beat":4,  "lane":0, "type":"TAP" },
    { "id":2, "beat":6,  "lane":1, "type":"TAP" },
    { "id":3, "beat":8,  "lane":2, "type":"HOLD", "durationBeats":2 },
    { "id":4, "beat":12, "lane":3, "type":"TAP" },
    { "id":5, "beat":14, "lane":0, "type":"TAP" },
    { "id":6, "beat":16, "lane":1, "type":"HOLD", "durationBeats":4 }
  ]
}
```

## 21.2. GameConstants.kt (эталон)
```kotlin
object GameConstants {
    const val LANES = 4
    const val PERFECT_MS = 50L
    const val GOOD_MS = 120L

    const val HOLD_PCT_NORMAL = 10;  const val HOLD_PCT_TH = 5
    const val HOLD_MIN_NORMAL = 80L; const val HOLD_MAX_NORMAL = 300L
    const val HOLD_MIN_TH = 50L;     const val HOLD_MAX_TH = 150L
    const val ASSIST_GRACE_MS = 150L
    const val DRIFT_TOLERANCE_LANES = 0.6f
    const val DRIFT_TIME_MS = 100L

    const val SWIPE_MIN_DP = 24f
    const val SWIPE_MAX_MS = 200L
    const val DOUBLE_TAP_SYNC_MS = 80L

    const val HIT_LINE_RATIO = 0.82f
    const val NOTE_HEIGHT_RATIO = 0.55f
    val APPROACH_MS = mapOf(1 to 1200, 2 to 1050, 3 to 900, 4 to 800,
        5 to 700, 6 to 600, 7 to 520, 8 to 450, 9 to 380, 10 to 320)
    const val DEFAULT_SCROLL = 5

    const val CONTINUE_LIMIT = 1
    const val CONTINUE_COST = 100
    const val CONTINUE_STUB_SEC = 5
    const val CONTINUE_REWIND_MS = 1500.0
    const val COUNTDOWN_MS = 2000L

    const val FEVER_COMBO = 50
    const val SCORE_PERFECT = 300; const val SCORE_GOOD = 100
    const val SCORE_HOLD_END = 300; const val SCORE_COMPLEX_END = 500
    const val GOLD_MULTIPLIER = 3

    const val LONG_HOLD_THRESHOLD_MS = 3000L
    const val LONG_HOLD_SEGMENT_MS = 2500.0
    const val LONG_HOLD_GAP_MS = 120.0

    const val EDITOR_TAPS = 10
    const val EDITOR_TAP_TIMEOUT_MS = 3000L
    const val AUTOSAVE_MS = 30_000L
    const val UNDO_STACK = 100

    const val MAX_NOTES = 20_000
    const val MAX_FILE_BYTES = 60L * 1024 * 1024
    const val MAX_AUDIO_MS = 600_000L

    const val COINS_CLEAR = 50; const val COINS_PER_STAR = 25
    const val START_COINS = 300
}
```

## 21.3. Глоссарий
**Доля (beat)** — единица музыкального времени. **Approach time** — время падения ноты от верха до линии удара. **Anchor** — пара «аудио-время ↔ uptime» для перевода времени касания. **Judge** — модуль судейства. **TH** — True Hardcore. **PACK/TEXT** — два варианта `.mwt`. **Категория рекорда** — NORMAL / CASUAL / TRUE_HARDCORE.

## 21.4. Лицензии на контент
Игра поддерживает любые локальные аудиофайлы. В поставляемый APK включать только собственную или свободно лицензированную музыку (CC0 / CC-BY с указанием автора в экране Credits). Ответственность за права на импортированный контент лежит на пользователе.

---
