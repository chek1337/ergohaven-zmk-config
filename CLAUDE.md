# CLAUDE.md

Гид по этому репо для агента. Здесь описано как устроен рендер раскладки в SVG и где править что, чтобы менять рендер точечно.

## Пайплайн рендера

1. `.github/workflows/draw-keymaps.yml` — триггерится на push в `config/*.keymap`, `config/*.dtsi`, `config/*.json`, `keymap_drawer.config.yaml`, или сам себя.
2. Использует `caksoylar/keymap-drawer/.github/workflows/draw-zmk.yml@main` для парсинга keymap'а и рендера SVG.
3. Параметры передаются: `keymap_patterns: "config/*.keymap"`, `config_path: "keymap_drawer.config.yaml"`, `output_folder: "keymap-drawer"`.
4. Workflow коммитит результат с сообщением `keymap-drawer render` — после локальных изменений делать `git pull --rebase` перед push'ем.

Артефакты:
- `keymap-drawer/velvet_v3_ui_ruen.yaml` — промежуточный YAML после парсинга keymap'а (полезно открыть, чтобы увидеть что именно `keymap-drawer` извлёк до применения CSS).
- `keymap-drawer/velvet_v3_ui_ruen.svg` — финальный рендер.

## Карта keypos на Velvet (45 позиций)

Выведено из `config/velvet_v3_ui_ruen.keymap` строк 225–228.

| Зона          | Левая половина | Правая половина |
|---------------|----------------|-----------------|
| Ряд 1 (top)   | 0–5            | 6–11            |
| Ряд 2 (home)  | 12–17          | 18–23           |
| Ряд 3 (bot)   | 24–29          | 30–35           |
| Thumbs        | 36–40 (5 шт.)  | 41–45 (5 шт.)   |

Полезные «именные» позиции:
- Home-row mods (Base): 13(GUI/A), 14(ALT/S), 15(SHIFT/D), 16(CTRL/F), 19(CTRL/J), 20(SHIFT/K), 21(ALT/L), 22(GUI/P)
- Активные thumb-кнопки на Base: 38 TAB, 39 SPACE, 40 Nav-mo, 41 Symbols-mo, 42 AddNav-mo
- Структурные `&none` (внешние колонки и неиспользуемые thumbs): 0, 12, 23, 24, 35, 36, 37, 43, 44, 45

## Структура `keymap_drawer.config.yaml`

Файл делится на два корневых ключа.

### `parse_config:` — что парсится из keymap'а в YAML

| Блок | За что отвечает | Куда смотреть для правок |
|------|-----------------|--------------------------|
| `zmk_remove_keycode_prefix` | Срезает префиксы `RU_`/`EN_` с keycode'ов перед маппингом | Если появятся новые префиксы behavior'ов |
| `zmk_combos` | Размещение combo-боксов: `align: top/bottom/left/right`, `offset: 0..1`. Решает наложения, когда несколько combo сидят на одних/соседних клавишах. | Имена combo берутся из `config/velvet_v3_ui_ruen.keymap` строк 79–162 (например `cmben`, `russian_ha`). При появлении нового combo — добавить запись здесь. |
| `raw_binding_map` | Маппит binding-выражение (полное или по `binding_parts[0]`) на `tap`/`hold`/`shifted`/`left`/`right`/`tl`/`tr`/`bl`/`br`/`type`. Имеет приоритет над `zmk_keycode_map` и над дефолтным парсингом keymap-drawer'а (включая `&none`). | Используется для: custom behavior'ов (`&bootloader`, `&caps_word`, `&layer_en/ru`), параметризованных макросов (`&en X` — язык-переключатель, рендерится `tap=символ + left=en`), составных хоткеев с модификатором, мышиных кнопок (`&mkp MB1/2/3`), и затемнения `&none` через `{type: ghost}` (keymap-drawer по умолчанию не вешает классов на пустые клавиши, поэтому правило `.ghost { opacity: 0.4 }` без этого не сработает). По умолчанию keymap-drawer склеивает модификатор и клавишу в одну строку (`Ctl+...`) — чтобы получить визуализацию как у home-row, нужно явно прописать `{tap: ..., hold: ...}` (для обычных клавиш — модификатор едет в угол через CSS) либо `{tap: ..., left: ...}` или `{tap: ..., right: ...}` (для combo — keymap-drawer сам ставит модификатор у бортика бокса с `small_pad` отступом). **Combo-only биндинги** (`&layer_en/ru`, `&kp LA(TAB)`, `&kp LC(SPACE)`, `&en X` и т.п. — встречаются только в `combos {}` блоке keymap'а) используют `left`/`right`. Биндинги, видимые на обычных клавишах (`&kp RS(ENTER)`, `&kp RC(DELETE)` и т.д.) — `hold`. |
| `zmk_keycode_map` | Маппит ZMK keycode (после срезания префикса) на текст или `$$mdi:icon$$`. Поддерживает форму `KEY: {tap: x, shifted: y, hold: z}` для разных слотов. | **Это главный блок для иконок**. Имена иконок — на https://pictogrammers.com/library/mdi/. Кириллические символы (CYRILLIC_*) тоже здесь — порядок не важен, ключи уникальны. |

### `draw_config:` — как рендерится SVG

| Блок | За что отвечает |
|------|------------------|
| `draw_key_sides: false` | Не рисовать «бортики» клавиш (плоский стиль). |
| `key_h: 64` | Высота клавиши в px. **Если меняется — пересчитать `translate` значения в `svg_style` (сейчас `22px / 2px / 4px` под `key_h: 64`)**. |
| `key_rx`/`key_ry: 4` | Радиус скругления клавиш. |
| `combo_w`/`combo_h: 26` | Размер combo-боксов. Сделай больше если combo-метка не помещается. Combo-боксы поддерживают **5 нативных позиций для меток** (см. `keymap_drawer/draw/combo.py`): `tap` (центр), `hold` (низ-середина), `shifted` (верх-середина), `left` (середина-слева), `right` (середина-справа). **Углов (`tl`/`tr`/`bl`/`br`) для combo НЕ поддерживается** — только для обычных клавиш. Поэтому для модификатора на combo используй `left:`/`right:` (а не `hold:`, который вылезет за маленький combo при выбранном `combo_h: 26`). |
| `inner_pad_w/h: 2` | Промежуток между клавишами. |
| `outer_pad_w: 0` | Горизонтальные внешние отступы выключены (SVG плотно к клавишам по бокам). |
| `outer_pad_h: 40` | Вертикальный отступ сверху/между слоями. **Должен быть ≥ font-size label'а слоя** (сейчас 32px) — иначе `text.label` обрезается viewBox'ом сверху первого слоя или налезает на клавиши следующего. |
| `small_pad: 4` | Отступ от края клавиши до вторичных меток (пока не используется напрямую — translate'ы перекрывают). |
| `shrink_wide_legends: 5` | Если tap-лейбл длиннее 5 символов — шрифт уменьшится. |
| `glyph_tap_size: 22` | Размер mdi-иконки на месте tap-метки. |
| `glyph_hold_size: 12` / `glyph_shifted_size: 12` | Размеры иконок на hold/shifted/left/right/tl/tr/bl/br позициях. Глобально, влияет и на обычные клавиши, и на combo. На combo дополнительно tap-glyph уменьшается через CSS scale (см. `.combo.tap.glyph` в `svg_style`). Если меняется — keymap-drawer пересчитает позиции автоматически, но если есть кастомные translate'ы под старый размер — пересмотри их. CSS-трюк `transform: scale(...)` с `transform-box: fill-box; transform-origin: center` масштабирует `<use>` относительно его центра, не сдвигая позицию (просто `width`/`height` через CSS сместил бы иконку, потому что `x`/`y` атрибуты считаются keymap-drawer'ом под исходный размер). |
| `append_colon_to_layer_header: false` | Имена слоёв без двоеточия. |
| `svg_style: \|` | Большой блок CSS. Структура ниже. |

### Структура `svg_style` (CSS)

Внутри `svg_style` блоки идут сверху вниз:

1. **`svg path { fill: inherit }`** — наследование fill для иконок через `<use>` теги.
2. **`svg.keymap { ... }`** — все CSS-переменные:
   - **`--color--material--blue-gray--50…900`** — нейтральная шкала (фон клавиш, текст). Имена legacy, **значения = GitHub Primer neutrals** (`#f6f8fa` → `#1f2328`).
   - **`--color--material--teal--50…500`** — акцент на F/J home-индексах. **Значения = GitHub Primer accent blue** (`#ddf4ff` → `#0969da`).
   - **`--color--bg`, `--color--text`, `--color--layer--text`** — общие переменные темы.
   - **`--color--key--*`** — фон/бордер/цвета текста для **обычных** клавиш.
   - **`--color--combo--*`** — то же для combo-боксов.
   - `font-family`, `font-size`, `font-weight` — типографика. Шрифт `system-ui, "Noto Sans", "Segoe UI", sans-serif` выбран чтобы корректно рендерить кириллицу.
3. **`rect, path { shape-rendering: geometricPrecision }`** — глобально для чёткости границ.
4. **`text { ... }`** — выравнивание (anchor middle, baseline middle).
5. **`text.label { ... }`** — стиль заголовков слоёв (32px, серый цвет).
6. **`rect.key, rect.combo`** — заливка/обводка клавиш и combo (через CSS-переменные).
7. **`.tap`** — цвет основной метки.
8. **`.shifted, .hold`** — общий font-size 14px для вторичных меток.
9. **`.shifted { translate: 22px 2px }`** и **`.hold { translate: 22px 4px }`** — **дефолтная позиция вторичных меток** (правый верх / правый низ). Для правой половины клавиатуры.
10. **`.keypos-N .shifted { translate: -22px 2px }`** для keypos левой половины — **зеркалит метки в левый угол**. Список keypos (0–5, 12–17, 24–29, 36–40) обновлять при изменении layout'а.
11. **`.keypos-N .hold { translate: -22px 4px }`** — то же для hold-меток.
12. **`.keypos-16 .key, .keypos-19 .key`** — **акцент teal на F и J** (home-индексы). Тут же переопределение цветов shifted/hold лейблов на основной — иначе серые на teal не читаются.
13. **`.keypos-38…42 .key`** — акцент blue-gray-100 на **активных thumb-клавишах**.
14. **`.combo`** — переопределяет переменные внутри combo-боксов (используют combo-палитру вместо key-палитры).
15. **`rect.combo { opacity: 0.8 }`** — combo-боксы полупрозрачные.
16. **`path.combo`** — стиль дендронов (линий от combo к ключам): тонкая `stroke-width: 1`, `opacity: 0.5`, цвет берётся из `--color--combo--dendron` (сейчас `blue-gray-400` = `#656d76`, иначе на светлом фоне будет невидно).
17. **`.combo.tap/shifted/hold/left/right`** — размеры шрифта внутри combo-бокса (tap 11px, остальные 8px).
18. **`.combo.tap.glyph { transform: scale(0.6); transform-box: fill-box; transform-origin: center }`** — **уменьшает основную tap-иконку в combo** с 22px до ~13px (это единственный CSS-scale-хак, который остался; модификаторы (`left`/`right`) используют глобальный `glyph_shifted_size: 12` — этого хватает без CSS-magic). `fill-box` обязателен — иначе origin от viewport SVG, и scale смещает иконку.
19. **Никаких CSS translate'ов для combo-меток**: hold/shifted/left/right позиционируются нативно keymap-drawer'ом (см. `keymap_drawer/draw/combo.py:_draw_legend`) — `hold` на нижнем крае центра, `shifted` на верхнем, `left`/`right` у боковых рёбер с `small_pad` отступом. Если разместить modifier через `hold:` в маленьком combo — он вылетит за низ бокса (combo_h/2 + 8px translate из CSS правил `.hold`). Решение — использовать `left:`/`right:` в `raw_binding_map` для combo-only биндингов.
20. **`text.left/right/tl/tr/bl/br` text-anchor + `text.shifted/tl/tr` и `text.hold/bl/br` dominant-baseline**: эти правила восстановлены вручную в начале блока (после `.tap`), потому что custom `svg_style` **полностью заменяет** дефолтный stylesheet keymap-drawer'а (см. `keymap_drawer/config.py:89` комментарий). Без них `text { text-anchor: middle }` сломает позиционирование `left`/`right` меток.
21. **`.trans, .none, .ghost { opacity: 0.4 }`** — приглушённые transparent/none/ghost клавиши.

## Точечные правки — куда лезть

| Что хочется | Куда |
|-------------|------|
| Убрать наложение combo-боксов | `parse_config.zmk_combos` — поправить `align`/`offset` для нужного combo по имени из `.keymap` |
| Поменять цвет конкретной клавиши | `svg_style` — добавить блок `.keypos-N .key { --color--key--bg: var(--color--material--<palette>--<shade>) }` |
| Заменить текст-лейбл на иконку | `parse_config.zmk_keycode_map` — добавить `KEYCODE: $$mdi:<icon-name>$$` (имена на pictogrammers.com/library/mdi) |
| Заменить отображение конкретного binding'а целиком | `parse_config.raw_binding_map` — добавить `"&behavior arg": {tap: ..., hold: ..., shifted: ...}` |
| Затемнить класс клавиш (например `&none`) | `parse_config.raw_binding_map: "&binding": {type: ghost}` — клавиша получит CSS-класс `ghost`, существующее правило `.ghost { opacity: 0.4 }` затемнит её. |
| Иконка наезжает на tap-лейбл в mod-tap'е | Добавить в `svg_style`: `.hold.glyph { translate: ... }` или `.shifted.glyph { translate: ... }` с микро-сдвигом |
| Сменить высоту клавиш | `draw_config.key_h` — **обязательно пересчитать `translate` 22px/2px/4px в `svg_style`** (пропорционально `key_h/64`) |
| Сменить размер шрифта/семейство | `svg_style` → `svg.keymap { font-family: ..., font-size: ... }` |
| Поменять палитру | `svg_style` → переменные `--color--material--*` или сами `--color--key--*` / `--color--combo--*` |
| Добавить новое combo в keymap | После добавления в `.keymap` — добавить запись в `zmk_combos` с тем же именем, иначе будет рендериться по дефолту (середина между ключами). Также: combo получает класс `combopos-N`, где `N` — порядковый индекс combo в `.keymap` (с нуля). Если новое combo попадает на левую/правую половину, добавить его индекс в соответствующий блок CSS `.combopos-N .hold/.shifted` (см. блок translate'ов для combo в `svg_style`), иначе будет рендериться с центрированным fallback'ом `.combo.hold/.shifted`. |
| Поменять имя слоя как оно показано | В `.keymap` через `display-name = "..."` (это уже используется — см. en/ru/sym_en и т.д.) |

## Структура keymap'а (`config/velvet_v3_ui_ruen.keymap`)

| Слой (layer index) | Имя `display-name` | Назначение |
|--------------------|---------------------|-------------|
| 0 | Base | Английский QWERTY с home-row mods |
| 1 | Ru | Русский ЙЦУКЕН с home-row mods |
| 2 | Symbols | Символы (en) |
| 3 | Symbols | Символы (ru — отдельный слой т.к. пунктуация на ru-раскладке другая) |
| 4 | AddNav | Цифры/навигация (en) |
| 5 | AddNav | Цифры/навигация (ru) |
| 6 | Nav | F-keys и стрелки |
| 7 | Adjust | Bluetooth, медиа, bootloader |
| 8 | Mouse | Кнопки мыши |
| 9 | Scroll | Прокрутка трекболом |
| 10 | Sniper | Снижение DPI трекбола |

Combo'ы и их имена — `config/velvet_v3_ui_ruen.keymap` строки 79–162. **Имена в `zmk_combos` должны точно совпадать с именами combo'ов в keymap'е.**

## Чего не делать

- **Не трогать `keymap-drawer/velvet_v3_ui_ruen.yaml` и `.svg` руками** — они генерируются workflow'ом, любая ручная правка будет затёрта при следующем рендере.
- **Не использовать `--no-verify` или amend опубликованных коммитов** — keymap-drawer workflow смотрит на push'и и может задвоить рендер.
- **Не добавлять `Co-Authored-By: Claude`** в коммиты этого репо.
- При commit/push конфликтуем с remote — делать `git pull --rebase origin <branch>` перед `git push`, потому что workflow сам коммитит SVG в ту же ветку.
