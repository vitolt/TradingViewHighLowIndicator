# Лог ошибок и решений

Документирование всех найденных ошибок и их решений для предотвращения повторений.

---

## Ошибка 1: `timeframe.in_minutes()` не существует

**Дата**: 2026-10-07  
**Тяжесть**: 🔴 Critical — препятствует компиляции  
**Строка**: 84 в HighLow.pine

### Проблема
```pine
f_chart_rank() =>
    m = timeframe.in_minutes()  // ❌ Нет такой функции в Pine v5
    switch
        m < 1440  => 0
        ...
```

### Причина
В Pine Script v5 нет встроенной функции `timeframe.in_minutes()`. Нужно использовать `timeframe.multiplier` с `timeframe.period` для определения ранга.

### Решение
Использовать встроенные константы времени и сравнение строк:
```pine
f_chart_rank() =>
    switch timeframe.period
        "1" => 0    // 1m
        "3" => 0    // 3m
        "5" => 0    // 5m
        "15" => 0   // 15m
        "30" => 0   // 30m
        "60" => 0   // 1h
        "120" => 0  // 2h
        "240" => 0  // 4h
        "D" => 1    // 1D
        "W" => 2    // 1W
        "M" => 3    // 1M
        => 3        // M+ (месячные и выше)
```

**Применение**: ✅ Исправлено в HighLow.pine строка 84–95

---

## Ошибка 2: Изменения элементов UDT в массиве не сохраняются

**Дата**: 2026-10-07  
**Тяжесть**: 🔴 Critical — логика не работает  
**Строка**: 293 в HighLow.pine

### Проблема
```pine
lvl = array.get(levels, i)
lvl.crossBar := bar_index      // ❌ Изменение не сохраняется в массиве
array.set(levels, i, lvl)      // ← нужна эта строка
```

Линии не завершались при пересечении, потому что состояние уровня (флаг `crossBar`)
менялось в локальной переменной, но в массиве оставалось старое значение.

### Причина
В Pine Script UDT (User Defined Types) передаются по значению, не по ссылке.
Изменение локальной копии не влияет на элемент массива.

### Решение
После изменения поля UDT, полученного из массива, нужно явно обновить его:
```pine
lvl = array.get(levels, i)
lvl.crossBar := bar_index      // изменение в локальной копии
// ... другие операции ...
array.set(levels, i, lvl)      // ✅ сохранить изменения в массиве
```

**Применение**: ✅ Исправлено в HighLow.pine строка 296

---

## Ошибка 3: Логика создания уровней — проверка пересечения работает неправильно

**Дата**: 2026-10-07  
**Тяжесть**: 🔴 Critical — основной функционал не работает  
**Строка**: 264–279 в HighLow.pine (старая версия)

### Проблема
Линии не заканчивались при пересечении, даже после исправления `array.set()`.

Уровни создавались только в моменты переходов периодов (`if new_day`, `if new_week`).
В остальные дни блоки создания не выполнялись, но проверка пересечения всё равно
искала эти уровни в массиве. Однако большинства уровней там вообще не было!

Пример:
- День 1 (понедельник): создаётся DH
- День 2–6: DH уже не создаётся снова, но проверка ищет его в массиве
- День 7 (пятница): цена может пересечь DH, но проверка уже не видит этот уровень
  в новых добавлениях

### Причина
Ошибочное разделение логики:
- **Фаза создания** (только при `new_*`) — асинхронна, редко запускается
- **Фаза проверки** (каждый бар) — ищет уровни, которых может не быть в массиве

### Решение
**Архитектура 2.0**: Создавать все уровни один раз в начале, затем каждый бар
обновлять их положение и состояние.

```pine
// При переходе на новый период → добавить новый уровень в массив
if new_day
    f_add(d_high, 1, true, day_start)   // добавляем только при new_day
    day_start := bar_index

// Каждый бар → проверяем ВСЕ уровни в массиве
if array.size(levels) > 0
    for i = 0 to array.size(levels) - 1
        lvl = array.get(levels, i)
        if lvl.crossBar == -1
            crossed = lvl.isHigh ? high >= lvl.price : low <= lvl.price
            if crossed
                lvl.crossBar := bar_index
                // ...
```

**Применение**: ✅ Переписана структура главного цикла в HighLow.pine

### Дополнительное улучшение
Удалено кэширование экстремумов (`var` переменные). Теперь каждый бар переполучаются
свежие значения через `request.security()`, что гарантирует:
- Корректное отображение при масштабировании по вертикали
- Отсутствие стеков данных при переключении инструментов
- Всегда актуальные экстремумы

## Справочник: работа с массивами UDT в Pine v5

| Операция | Синтаксис | Примечание |
|----------|-----------|-----------|
| Получить элемент | `lvl = array.get(levels, i)` | Копирует значение |
| Изменить элемент | `array.set(levels, i, lvl)` | ✅ Правильно |
| Добавить | `array.push(levels, lvl)` | — |
| Размер | `array.size(levels)` | — |

⚠️ **Важно**: После `get()` и изменения поля → обязательно `set()` обратно!

## Справочник: функции TradingView, которых нет в Pine v5

| Функция | Статус | Альтернатива |
|---------|--------|--------------|
| `timeframe.in_minutes()` | ❌ | `timeframe.period`, `timeframe.multiplier` |
| `string.contains()` | ❌ | `str.contains()` |
| `bar.index` | ❌ | `bar_index` |
| `chart.symbol` | ❌ | `syminfo.tickerid` |

---

## Справочник: правильный синтаксис Pine v5

### Типы данных и объявления
```pine
// ✅ Правильно
label lb = na
if condition
    lb := label.new(...)

// ❌ Неправильно
lb = condition ? label.new(...) : na  // может не скомпилироваться
```

### Работа со строками
```pine
// ✅ Правильно
str.contains(tag, "DH")

// ❌ Неправильно
string.contains(tag, "DH")  // нет такой функции
```

### Объекты типов line/label
```pine
// ✅ Правильно
line.set_x2(ln, bar_index)
label.set_text(lb, "DH")

// ❌ Неправильно
ln.set_x2(bar_index)  // Pine использует функции, а не методы
```

### Работа с массивами
```pine
// ✅ Правильно
array.push(levels, lvl)
array.get(levels, i)
array.size(levels)

// ❌ Неправильно
levels.push(lvl)  // методов нет
levels[i]  // индексация не поддерживается для некоторых типов
```

---

## Дополнительные проверки перед компиляцией

Перед загрузкой в TradingView проверить:

1. **Функции**
   - [ ] Все функции из встроенных: `request.security`, `line.new`, `label.new`, `ta.change`
   - [ ] Нет опечаток вроде `timeframe.in_minutes` или `string.contains`

2. **Типы данных**
   - [ ] `line`, `label` объявлены явно, если присваиваются условно
   - [ ] UDT поля используются правильно

3. **Синтаксис**
   - [ ] Все переменные инициализированы
   - [ ] `for` циклы используют `to` (не `in`)
   - [ ] Ветки `if` не используют `else if` для разных condition (вместо этого `switch`)

4. **Производительность**
   - [ ] Нет бесконечных циклов
   - [ ] Нет рекурсии
   - [ ] Кэширование используется для дорогостоящих операций

---

---

## Ошибка 4: Инициализация типов line и label в UDT (РЕШЕНО v3)

**Дата**: 2026-10-07 18:40 → 19:00  
**Тяжесть**: 🔴 Critical → ✅ Решено  
**Строки**: 71-78 в HighLow.pine

### Проблема
```pine
// ❌ Попытка 1: использовать na(line)
type Level
    line ln
    label lb

new_lv = Level.new(..., na(line), na(label))  // Ошибка: Cannot call with argument 'ln'='call na (series bool)'
```

### Попытка 1 (не сработало)
```pine
// ❌ Попытка 2: использовать var в типе
type Level
    var line ln  // Ошибка: The keywords 'var' cannot be used in a type declaration
    var label lb
```

### Правильное решение (v3)
```pine
// ✅ Шаг 1: Определить тип без инициализации
type Level
    float price
    string label_tags
    int start_bar
    int cross_bar
    bool is_high
    string level_type
    line ln      // Будет инициализирована как na по умолчанию
    label lb     // Будет инициализирована как na по умолчанию

// ✅ Шаг 2: Создать глобальные переменные с var
var line dummy_line = na
var label dummy_label = na

// ✅ Шаг 3: Передавать dummy значения в конструктор
new_lv = Level.new(dh_val, "DH", bar_index, 0, true, "D", dummy_line, dummy_label)

// ✅ Шаг 4: Присваивать реальные значения позже через line.new() и label.new()
if na(lv.ln)
    lv.ln := line.new(...)
```

**Почему это работает**:
- Типы `line` и `label` в UDT инициализируются как `na` автоматически
- Через конструктор можно передавать `na` значения (dummy переменные)
- Позже через `line.new()` и `label.new()` присваиваем реальные объекты

**Применение**: ✅ Исправлено в HighLow.pine v3. Все 6 вызовов Level.new() теперь передают `dummy_line, dummy_label`.

---

## Ошибка 5: Проверка ta.change() возвращает series bool

**Дата**: 2026-10-07 18:50  
**Тяжесть**: 🟡 Warning  
**Строка**: 146, 177, 208 в HighLow.pine

### Проблема
```pine
if ta.change(time("D"))  // ⚠️ Warning: accepts 'bool', got 'series bool'
```

### Причина
Функция `ta.change()` возвращает `series bool`, а `if` требует простой `bool`.

### Решение
Сравнить результат с нулём или используется оператор `!= 0`:

```pine
// ✅ Правильно: сравнение с нулём
if ta.change(time("D")) != 0
    // ...
```

**Применение**: ✅ Исправлено в HighLow.pine v2. Все `ta.change()` обёрнуты в `!= 0`.

---

---

## Справочник: работа с типами line/label в Pine v5

| Операция | ✅ Правильно | ❌ Неправильно |
|----------|-----------|-------------|
| Создать линию | `ln = line.new(...)` | `ln = na(line)` |
| Удалить линию | `line.delete(ln)` | `del(ln)` |
| Проверить null | `if not na(ln)` | `if ln != na` |
| Присвоить null | `ln := na(line)` | `ln = na(line)` (глобально) |
| В структуре UDT | Инициализировать поля отдельно | Передавать в конструктор |

**Правило**: Типы `line`, `label`, `box`, `polyline` нельзя инициализировать в конструкторе UDT. Только после создания объекта через `.new()`.



## Ошибка 6: Полный набор ошибок исправлен в v1.0.5

**Дата**: 2026-10-07 19:15  
**Тяжесть**: 🔴 Critical → ✅ РЕШЕНО  
**Строки**: 79-80, 174, 207, 224, 240 в HighLow.pine

### Проблема (v1.0.4 → v1.0.5)

В версии 1.0.4 оставались 2 критические ошибки:

1. **var в типе Level** (строки 79-80):
```pine
type Level
    ...
    var line ln      // ❌ Ошибка: The keywords 'var' cannot be used in a type declaration
    var label lb
```

2. **Неполные вызовы Level.new()** (строки 174, 207, 224, 240):
```pine
new_lv = Level.new(dl_val, "DL", bar_index, 0, false, "D")  // ❌ Не хватает 2 параметров
new_lv = Level.new(wl_val, "WL", bar_index, 0, false, "W")  // ❌ Ошибка: Missing argument 'ln'
new_lv = Level.new(mh_val, "MH", bar_index, 0, true, "M")   // ❌ Ошибка: Missing argument 'lb'
new_lv = Level.new(ml_val, "ML", bar_index, 0, false, "M")  // ❌ Ошибка: Missing argument 'ln'
```

### Причина

1. **var в типе**: В Pine Script v5 ключевое слово `var` не может использоваться внутри определения типа (type declaration). `var` зарезервирован только для глобальных переменных и переменных функций для сохранения состояния между барами.

2. **Неполные вызовы**: Конструктор `Level.new()` требует 8 аргументов согласно определению типа:
```pine
type Level
    float price          // arg 1
    string label_tags    // arg 2
    int start_bar        // arg 3
    int cross_bar        // arg 4
    bool is_high         // arg 5
    string level_type    // arg 6
    line ln              // arg 7 (требуется!)
    label lb             // arg 8 (требуется!)
```

Но некоторые вызовы передавали только 6 аргументов.

### Решение (v1.0.5)

**Шаг 1**: Удалить `var` из типа Level:
```pine
// ✅ ПРАВИЛЬНО
type Level
    float price
    string label_tags
    int start_bar
    int cross_bar
    bool is_high
    string level_type
    line ln           // Без var — будет инициализирована как na автоматически
    label lb          // Без var — будет инициализирована как na автоматически
```

**Шаг 2**: Объявить `var` глобальные переменные для dummy значений:
```pine
// ✅ ПРАВИЛЬНО
var line dummy_line = na
var label dummy_label = na
```

**Шаг 3**: Передавать все 8 аргументов в каждый Level.new():
```pine
// ✅ ПРАВИЛЬНО - все 4 вызова исправлены
new_lv = Level.new(dh_val, "DH", bar_index, 0, true, "D", dummy_line, dummy_label)  // строка 158
new_lv = Level.new(dl_val, "DL", bar_index, 0, false, "D", dummy_line, dummy_label) // строка 174
new_lv = Level.new(wh_val, "WH", bar_index, 0, true, "W", dummy_line, dummy_label)  // строка 191
new_lv = Level.new(wl_val, "WL", bar_index, 0, false, "W", dummy_line, dummy_label) // строка 207
new_lv = Level.new(mh_val, "MH", bar_index, 0, true, "M", dummy_line, dummy_label)  // строка 224
new_lv = Level.new(ml_val, "ML", bar_index, 0, false, "M", dummy_line, dummy_label) // строка 240
```

### Почему это работает

1. Типы `line` и `label` в UDT инициализируются как `na` автоматически (не требуют явного `na(line)`)
2. Через конструктор можно передавать `na` значения (dummy переменные)
3. Позже в коде через `line.new()` и `label.new()` присваиваются реальные объекты:
```pine
if na(lv.ln)
    lv.ln := line.new(...)  // Присвоить реальный line
if na(lv.lb)
    lv.lb := label.new(...) // Присвоить реальный label
```

**Применение**: ✅ Исправлено в HighLow.pine v1.0.5. Все ошибки устранены.

---

## Итоговый чек-лист Pine Script v5 для типов line/label

| Операция | ✅ Правильно | ❌ Неправильно |
|----------|-----------|-------------|
| Объявление в типе | `line ln` | `var line ln` |
| Инициализация глобального | `var line ln = na` | `line ln = na` |
| Передача в конструктор | `Level.new(..., dummy_line)` | `Level.new(..., na(line))` |
| Проверка null | `if not na(lv.ln)` | `if lv.ln != na` |
| Присвоение | `lv.ln := line.new(...)` | `lv.ln = line.new(...)` |
| Создание линии | `line.new(x1, y1, x2, y2)` | `line(x1, y1, x2, y2)` |

---
