---
name: developer-role-pine-v5
description: Роль разработчика Pine Script v5 для TradingView, установлена для проекта High-Low Levels
metadata:
  type: feedback
---

# Роль: Разработчик Pine Script v5 для TradingView

**Установлена**: 2026-10-07 для проекта High-Low Levels Indicator

## Ключевые правила

1. **Всегда использую `var` для `line` и `label` в UDT**
   - ❌ `line ln` → ✅ `var line ln`
   - Это позволяет оставить поле неинициализированным

2. **Level.new() вызываю без line/label аргументов**
   - ❌ `Level.new(..., na(line), na(label))` → ✅ `Level.new(price, tag, bar, 0, is_high, type)`
   - line/label присваиваю позже через `line.new()` и `label.new()`

3. **Всегда вызываю `array.set()` после изменения UDT в массиве**
   - UDT передаются по значению, изменения локальной копии не сохраняются без `array.set()`

4. **Проверка `ta.change()` должна быть `!= 0`**
   - ❌ `if ta.change(time("D"))` → ✅ `if ta.change(time("D")) != 0`
   - `ta.change()` возвращает `series bool`, требуется сравнение для простого `bool`

5. **Документирую все ошибки в ERRORS_LOG.md**
   - Причина, решение, применение
   - Для предотвращения повторения в будущем

## Типичные ошибки, которых избегаю

| Ошибка | Решение |
|--------|---------|
| `na(line)` как аргумент | Использовать `var line ln` в типе |
| `line()` конструктор | Использовать `line.new()` |
| Забывчивость `array.set()` | Всегда после изменения UDT в массиве |
| `timeframe.in_minutes()` | Использовать `timeframe.period` |
| `string.contains()` | Использовать `str.contains()` |

## Контекст проекта

- **Проект**: High-Low Levels Indicator для TradingView
- **Язык**: Pine Script v5
- **Основной файл**: HighLow.pine (277 строк)
- **Статус**: ✅ Готов к запуску в TradingView

**Почему**: Много специфических ошибок при работе с UDT и line/label типами в Pine v5. Роль фиксирует знание этих особенностей и ошибок для консистентной разработки.

**Как применять**: При любой разработке на Pine v5 для этого проекта или подобных индикаторов, следовать этим правилам.
