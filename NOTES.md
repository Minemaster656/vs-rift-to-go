# RiftBottles Mod — Заметки после первого теста (3.9.2026)

## Архитектура визуальных эффектов

### Как работает vanilla temporal stability visual:
- `BehaviorTemporalStabilityAffected.OnGameTick()` — каждый кадр
  - Строка 388: `GlitchStrength = 0` (всегда сбрасывает)
  - Строка 346: `target = Math.Max(0, (0.2 - ownStability) / 0.2) + glitchEffectExtraStrength`
  - Строка 347: плавная интерполяция `glitchEffectStrength += (target - current) * deltaTime/3`
  - Строки 391-397: Устанавливает GlitchStrength, GlitchWaviness, GlobalWorldWarp ТОЛЬКО если `fogEffectStrength > 0.05 || glitchEffectStrength > 0.05`
  - **VANILLA BUG**: Когда условие false — GlitchWaviness и GlobalWorldWarp НЕ сбрасываются в 0
- `DefaultShaderUniforms`: `GlitchStrength`, `GlitchWaviness`, `GlobalWorldWarp` — public float поля
- ModSystem tick listeners выполняются ПОСЛЕ entity ticks в одном кадре

### Коэффициент визуала (НЕ для силы!)
Коэффициент определяет **КАКОЙ** визуал получаем:
- `< -0.5` → принудительная нестабильность (bad trip visuals)
- `-0.5..+0.5` → игра сама решает (vanilla behavior based on real stability)
- `> 0.5` → принудительная стабильность (purifier, подавление визуала)

Bad trip = -3, Tier1 = -3, Tier2 = -1, Purifier = +3

### Исправленное поведение:
- Tier 1: ДОЛЖЕН управлять визуалом (coeff -3), НО сила = как при 0% стабильности vanilla (0.75 GlitchStrength, не 0.85-1.0)
- Tier 2: ДОЛЖЕН управлять визуалом (coeff -1)
- Bad trip: принудительный максимум (coeff -3)
- Purifier: подавление визуала (coeff +3)

## Исправленные проблемы

### 1. Мерцание (flickering)
**Причина**: Client tick каждые 32мс vs behavior каждый кадр (~16мс)
**Фикс**: `RegisterGameTickListener(OnClientTick, 0)` — каждый кадр

### 2. "Залипание" GlitchWaviness/GlobalWorldWarp
**Причина**: Vanilla bug — не сбрасываются когда glitchEffectStrength < 0.05
**Фикс**: В нейтральной зоне проверяем `GlitchStrength < 0.05` и обнуляем оба

### 3. Tier 1 экстримальный
**Причина**: Фиксированная сила 0.85-1.0 в ApplyBadTripVisuals
**Фикс**: Сила = 0.75 (как vanilla при 0%), с multiply от instabilityWavingStrength

### 4. Разлом на (0,0,0) при release
**Причина**: /giveitem с {filled:true} не записывает данные о позиции
**Фикс**: Release использует `byEntity.ServerPos.XYZ` (игрок), НЕ сохранённые атрибуты

### 5. Дрифтеры в воздухе
**Причина**: Спавн ищет блоки ВВЕРХ от позиции
**Фикс**: Ищет землю ВНИЗ (до -10), потом воздух ВВЕРХ

### 6. Дрифтеры дёргаются
**Причина**: `ServerPos.SetPos()` deprecated
**Фикс**: Устанавливаем X/Y/Z напрямую + `ServerPos.SetFrom(ServerPos)`

### 7. Визуалrift в бутылке — через модельку
Rift визуализируется через 3D модель (rift-core + rift-aura элементы), НЕ через vanilla рендерер.
9 shape файлов: empty и filled варианты для каждой бутылки.

## Debug команды
- `/settempstability 0-100 [playername]` — через `entity.WatchedAttributes.SetDouble("temporalStability", value)`
- `/placerift [x y z]` — создаёт Rift и добавляет в `riftSystem.riftsById`

## Текстуры модели
| Элемент | Дефолт | Path |
|---------|--------|------|
| Стены | quartz | `block/glass/quartz` |
| Крышка (tier 1) | rust | `block/metal/lantern/rust` |
| Крышка (tier 2) | resin | `block/resin` |
| Крышка (tier 3, bad trip, purifier flask) | temporalgear | `item/resource/temporalgear` |
| Крышка (purifier) | rust | `block/metal/lantern/rust` |
| Rift core | leaded-brown | `block/glass/leaded-brown` |
| Rift aura | violet | `block/glass/violet` |
| Bad trip core | temporalgear | `item/resource/temporalgear` |
| Bad trip aura | leaded-brown | `block/glass/leaded-brown` |
| Purifier walls | plain | `block/glass/plain` |

## Shape файлы — 9 штук
1. `riftbottle1.json` — empty (без rift)
2. `riftbottle1-filled.json` — с rift (quartz стены, rust крышка)
3. `riftbottle2.json` — empty
4. `riftbottle2-filled.json` — с rift (quartz стены, resin крышка)
5. `riftbottle3.json` — empty
6. `riftbottle3-filled.json` — с rift (quartz стены, temporalgear крышка)
7. `riftbottlebadtrip.json` — с rift (violet стены, temporalgear крышка, temporalgear core, leaded-brown aura)
8. `portablepurifier.json` — без rift (plain стены, temporalgear крышка)
9. `purifier.json` — без rift (plain стены, rust крышка)

## Debugging notes
- Регистрация команд через `api.RegisterCommand` deprecated, нужен `api.ChatCommand`
- `CmdArgs.PopWord()` вместо `NextWord()`, `float.TryParse()` вместо `NextFloat()`
- `IServerPlayer.PlayerName` с большой N
- `DefaultShaderUniforms` из `Vintagestory.API.Client`
- ModSystem tick (interval=0) = каждый кадр, выполняется ПОСЛЕ entity ticks

## Ресурсы
- VS core: `/home/minemaster/git-clones/vs-core/`
- Game dir: `/home/minemaster/Загрузки/vintagestory-1.22.5/`
- Deploy: `~/.config/VintagestoryData/Mods/RiftBottles/`
- Build: `http_proxy=http://127.0.0.1:9060 https_proxy=http://127.0.0.1:9060 ALL_PROXY=socks5://127.0.0.1:9050 dotnet build /home/minemaster/dev/vs-rift-to-go/RiftBottles/RiftBottles.csproj`
