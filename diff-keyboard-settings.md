# Разбор изменений с `feature/update_my_kb` для переноса в `main_ref`

Назначение: инвентарь всех отличий экспериментальной ветки `feature/update_my_kb` от точки
разветвления, плюс чек-лист переноса. Формат — «что было → что стало → где (file:line) → статус».
Использовать как задание для агента: переносить по пунктам чек-листа (раздел 11), сверяя значения
в разделах 4–10.

## 1. Провенанс

- Точка разветвления: `082ecea` («Update keymap drawings»). Все сравнения ниже: `082ecea` (база) →
  `feature/update_my_kb` (эксперимент) и `082ecea` → `main_ref` (то, что ушло вперёд на main).
- `feature/update_my_kb` (head `2abf667`), 10 коммитов, затронуты ровно 3 файла:
  - `92941bd` try to update
  - `a823937` Update qwerty.keymap
  - `18627f2` Update qwerty.keymap
  - `63e4908` Update qwerty.keymap
  - `9fd5a69` no sleep
  - `f001fa9` Update qwerty.keymap
  - `bd2274d` Update charybdis.conf
  - `2d82d0b` Update qwerty.keymap
  - `2733d43` Update qwerty.keymap
  - `2abf667` f1 and f2
- `main_ref` (head `82231a1`), 7 коммитов после форка (см. раздел 11):
  - `f4212c8` Feature/prospector display (#40)
  - `0377be6` keymap: make light composite background transparent
  - `338a97e` Fix typo
  - `a62c8ea` Add Prospector display brightness controls to shared keymaps (#42)
  - `4b93987` Update keymap drawings
  - `ecec42e` Fix build failure on second run: set core.fileMode=false before west update (#43)
  - `82231a1` add AGENTS.md

Изменённые в ветке файлы (полный список, `git diff --stat 082ecea origin/feature/update_my_kb`):

| Файл | Что изменилось |
|---|---|
| `config/keymaps/qwerty.keymap` | полная переработка (122 → 434 строки) — ядро эксперимента |
| `config/trackball/charybdis_pointer.dtsi` | перенос обработчиков на другие слои |
| `config/charybdis/charybdis.conf` | только конец файла (добавлен trailing newline) — нефункционально |

Прочие 4 кеймапа (`canary`, `colemak_dh`, `focal`, `graphite`) в ветке **не тронуты** — они
по-прежнему включают общие `keymap_features/*.dtsi` и используют старую схему слоёв.

Ссылки на версии файлов:
- feature: `git show origin/feature/update_my_kb:config/keymaps/qwerty.keymap`
- main_ref: `git show main_ref:config/keymaps/qwerty.keymap`
- база: `git show 082ecea:config/keymaps/qwerty.keymap`

## 2. Структура qwerty.keymap (feature vs main_ref)

| Элемент | main_ref (текущий) | feature |
|---|---|---|
| Upstream-includes | `behaviors.dtsi`, `bt.h`, `keys.h`, `mouse.h`, `outputs.h`, `mouse_keys.dtsi`, **`ext_power.h`** (строки 2–8) | `mouse.h`, `mouse_keys.dtsi`, `behaviors.dtsi`, `bt.h`, `keys.h`, `outputs.h`, **`pointing.h`** (строки 3–9) — `ext_power.h` выброшен |
| Локальные includes | `keymap_features/prospector_brightness.dtsi` (+stub `pbl`, строки 11, 39–47), `macros.dtsi`, `behaviors.dtsi`, `combos.dtsi` (строки 11–14) | **все 4 убраны**; блоки `behaviors {}` и `combos {}` инлайнятся в сам файл (строки 37–291 и 293–341) |
| Макросы (tmux, clipboard и т.д.) | используются в NAV/SYM/EXTRAS | **все ссылки удалены** |

Примечания:
- Удаление `ext_power.h` безопасно: `C_SLEEP`, `C_BRIGHTNESS_INC/DEC` и все consumer/keyboard-ключи
  определены в `keys.h` upstream (проверено: `zmk/app/include/dt-bindings/zmk/keys.h` строки
  894/963/969, `NUMBER_1` 103, `SQT` 282, `BSPC` 211, `LEFT_WIN` 787).
- `pointing.h` в feature фактически не нужен (никаких pointing-символов в кеймапе нет) — можно не
  переносить.
- Инлайн `behaviors {}`/`combos {}` — архитектурное решение: qwerty перестаёт зависеть от общих
  `keymap_features/`, остальные 4 кеймапа остаются на общих файлах. При переносе выбрать: (а)
  инлайн только в qwerty (как в feature) или (б) вернуть includes в общий файл.

## 3. Карта слоёв

| idx | 082ecea = main_ref | feature | комментарий |
|---|---|---|---|
| 0 | BASE | BASE | |
| 1 | NUM | NUM | дизайн слоя полностью переписан |
| 2 | NAV | NAV | убраны tmux-макросы |
| 3 | SYM | **EXTRAS** | |
| 4 | GAME | **SYM** | |
| 5 | EXTRAS | **GAME** | |
| 6 | SLOW | **SCROLL** | SLOW удалён |
| 7 | SCROLL | **mouse** | новый слой (кнопки мыши на thumb-клавишах) |
| 8 | — | **GAME-2** | новый слой |

Порядок узлов `keymap {}` в feature: BASE, NUM, NAV, EXTRAS, SYM, GAME, SCROLL, mouse, GAME-2
(строки 343–433). Блок `#define`-комментариев в feature (строки 24–32) **устарел** — он перечисляет
старую карту плюс «mouse 8»; реальная карта — как в таблице выше.

Воздействие на trackball см. раздел 7 (`charybdis_pointer.dtsi` привязан к номерам слоёв).

## 4. Послойные изменения

Позиции клавиш: 00–11 (верхний ряд), 12–23 (домашний), 24–35 (нижний), 36–40 (thumb-кластер).
«Было» = 082ecea (и main_ref, если не указано иначе).

### 4.1 BASE

| pos | было (082ecea / main_ref) | feature (qwerty.keymap:349–352) |
|---|---|---|
| 00 | `C_AC_SEARCH` (main_ref: то же) | `&kp TAB` |
| 01–10 | Q W E R T Y U I O P | Q W E R T Y U I O P (без изменений) |
| 11 | `&td_layers` | `&kp LCTRL` |
| 12 | `&kp TAB` | `&kp LCTRL` |
| 13 | `&ht_left LCMD A` (main_ref: `&ht_left_tp`) | `&kp A` — plain |
| 14 | `&ht_left LALT S` (main_ref: то же) | `&kp S` |
| 15 | `&ht_left LSHFT D` (main_ref: то же) | `&kp D` |
| 16 | `&ht_left LCTRL F` (main_ref: то же) | `&kp F` |
| 17–18 | G, H | G, H |
| 19 | `&ht_right RCTRL J` (main_ref: то же) | `&lt 7 J` — hold = слой mouse(7), tap = J |
| 20 | `&ht_right RSHFT K` (main_ref: то же) | `&kp K` |
| 21 | `&ht_right LALT L` (main_ref: `&ht_right_tp`) | `&kp L` |
| 22 | `&ht_right RCMD SEMICOLON` (main_ref: `&ht_right_tp`) | `&kp SEMICOLON` |
| 23 | `&kp DEL` | `&kp SQT` |
| 24 | `&clip_hist` | `&none` |
| 25–31 | Z X C V B N M | Z X C V B N; 31: `&lt 7 M` (hold = mouse, tap = M) |
| 32–34 | `&mm_cm_sc`, `&mm_pr_col`, `&mm_qm_ex` | `&kp COMMA`, `&kp PERIOD`, `&kp SLASH` (plain) |
| 35 | `&kp FSLH` | `&sl 3` — **ПРОБЛЕМНО, см. раздел 10 п.1** |
| 36 | `&td_esc_scrl_slow` | `&kp LEFT_WIN` |
| 37 | `&td_clk` | `&lt 1 ESCAPE` — hold = NUM, tap = ESC |
| 38 | `&td_bs` (main_ref: `&bs_num 1 BACKSPACE`) | `&mt LEFT_SHIFT SPACE` |
| 39 | `&kp ENTER` | `&lt 4 ENTER` — hold = SYM(4), tap = ENTER |
| 40 | `&td_space_nav` | `&kp BSPC` |

Суть: home-row hold-taps и mod-morphs убраны (plain QWERTY), J/M — toggle мышиного слоя,
thumb-клавиши — простые (Win / ESC-on-NUM / Shift-Space / ENTER-on-SYM / Backspace).

### 4.2 NUM (полный переписывание)

Было (082ecea/main_ref, qwerty.keymap:66–74): разреженный «numpad»: 06=PLUS, 07–10=`&mm_f7..f10`,
12=trans, 14=BACKSPACE, 15=LSHIFT, 18=KP_MINUS, 19–22=`&mm_f4..f6`, `&mm_f11`, 26=X, 30=EQUAL,
31–34=`&mm_f1..f3`, `&mm_f12`, 35=KP_SLASH, 38=trans, 39=N0, 40=SPACE; остальное none.

Стало (feature, qwerty.keymap:356–364) — полноценный «top-row» слой:

| pos | feature |
|---|---|
| 00 | `&kp TAB` |
| 01–10 | `NUMBER_1 NUMBER_2 NUMBER_3 NUMBER_4 NUMBER_5 NUMBER_6 N7 N8 N9 N0` |
| 11 | `&trans` |
| 12 | `&kp LCTRL` |
| 13–17 | `F1 F2 F8 F4 F5` (простые `&kp`, без mod-morph) |
| 18–21 | `LEFT_ARROW DOWN_ARROW UP_ARROW RIGHT_ARROW` |
| 22–23 | `&none`, `&none` |
| 24 | `&none` |
| 25–34 | `EXCLAMATION AT_SIGN HASH DOLLAR PERCENT CARET AMPERSAND ASTERISK LEFT_PARENTHESIS RIGHT_PARENTHESIS` |
| 35 | `&none` |
| 36–38 | `&trans` ×3 |
| 39 | `&kp LEFT_SHIFT` |
| 40 | `&kp DELETE` |

### 4.3 NAV

Изменены только позиции tmux-макросов — все заменены на `&none` (feature qwerty.keymap:366–374):
06=`detach_session`, 07=`create_or_attach`, 08=`next_pane`, 09=`show_sessions`, 10=`sync_panes`,
18=`h_split`, 30=`v_split`. Всё остальное идентично (1=msc MOVE_LEFT … 12=TAB, 13=MB4, 14=LALT,
15=LSHFT, 16=LCTRL, 17=MB5, 19–22 стрелки, 25–28 mmv, 31=HOME, 32=PG_DN, 33=PG_UP, 34=END,
40=trans).

### 4.4 EXTRAS (idx 5 в старом, 3 в feature)

Было 082ecea (qwerty.keymap:92–100) / main_ref (qwerty.keymap:106–114). Изменения feature
(qwerty.keymap:376–384):

| pos | 082ecea | main_ref | feature |
|---|---|---|---|
| 00 | `&studio_unlock` | то же | то же |
| 01 | `&shrug` | то же | `&sys_reset` |
| 02 | `&lgtm` | то же | `&none` |
| 03 | `&gcm` | то же | `&none` |
| 04 | `&none` | `&pbl PBL_INC` | `&none` |
| 05 | `&kp C_BRIGHTNESS_INC` | то же | то же |
| 06 | `&bt BT_SEL 0` | `&out OUT_TOG` | `&bt BT_SEL 0` (layout 082ecea) |
| 07–10 | `&bt BT_SEL 1..3` | `&bt BT_SEL 0..3` | `&bt BT_SEL 1..3` |
| 11 | `&bt BT_CLR` | `&none` (BT_CLR ушёл в combo `0+11`) | `&bt BT_CLR` (layout 082ecea) |
| 12 | `&kp C_SLEEP` | то же | то же |
| 13 | `&sudo` | то же | `&none` |
| 14 | `&none` | `&none` | `&none` |
| 15 | `&none` | `&pbl PBL_TOG` | `&none` |
| 16 | `&none` | `&pbl PBL_DEC` | `&none` |
| 17 | `&kp C_BRIGHTNESS_DEC` | то же | то же |
| 18 | `&none` | `&none` | `&out OUT_BLE` |
| 19–23 | C_PREVIOUS C_PLAY_PAUSE C_STOP C_NEXT none | то же | то же |
| 24–29 | C_AL_COFFEE C_AC_UNDO C_AC_CUT C_AC_COPY C_AC_PASTE none | то же | то же |
| 30 | `&new_dir` | то же | `&out OUT_USB` |
| 31–35 | K_MUTE C_VOLUME_DOWN C_VOLUME_UP PRINTSCREEN none | то же | то же |
| 36–40 | none ×5 | то же | то же |

Суть: убраны «fun»-макросы (shrug/lgtm/gcm/sudo/new_dir), добавлены `sys_reset` (01),
`OUT_BLE` (18), `OUT_USB` (30). **Конфликт при переносе**: main_ref переложил BT/OUT-клавиши
(OUT_TOG=06, BT_SEL=07–10, BT_CLR=combo) и добавил PBL-яркость (04/15/16); feature следует layout
082ecea. Нужно решение по каждому conflicting-положению (см. раздел 11, P5).

### 4.5 SYM (idx 3 в старом, 4 в feature)

Полный переписывание (feature qwerty.keymap:386–394):

| pos | feature |
|---|---|
| 00, 02, 09–10 | `&none` |
| 01 | `&kp TAB` |
| 03–08 | `LEFT_PARENTHESIS RIGHT_PARENTHESIS TILDE GRAVE SQT DOUBLE_QUOTES` |
| 11 | `&kp BACKSPACE` |
| 12 | `&to 5` — переход в GAME (5 в новой схеме) |
| 13 | `&kp LCTRL` |
| 14, 22–23, 24, 26, 34–35 | `&none` |
| 15–21 | `LEFT_BRACE RIGHT_BRACE PIPE MINUS PLUS EQUAL NON_US_BACKSLASH` |
| 25 | `&kp LSHFT` |
| 27–33 | `LEFT_BRACKET RIGHT_BRACKET AMPS UNDERSCORE LESS_THAN GREATER_THAN SLASH` |
| 36–40 | `&trans` ×5 |

Убраны все 7 editor-макросов (move_line_up/down, mc_add_above/below, delete_line, home_dir,
code_blk). `display-name = "sybol"` — опечатка (в main_ref «Sym»); при переносе поправить.

### 4.6 GAME (idx 4 в старом, 5 в feature)

| pos | было (082ecea/main_ref) | feature (qwerty.keymap:396–404) |
|---|---|---|
| 00 | `&kp N1` | `&kp ESC` |
| 01–05 | TAB Q W E R | то же |
| 06 | `&none` | `&to 0` — возврат в BASE |
| 07–11 | none×4, trans | то же |
| 12 | `&kp N2` | `&kp T` |
| 13 | `&kp LCTRL` | `&kp LSHFT` |
| 14–17 | A S D F | то же |
| 24 | `&kp N3` | `&kp B` |
| 25 | `&kp LSHFT` | `&kp LCTRL` |
| 26 | `&kp Y` | `&kp Z` (исправление раскладки) |
| 27–29 | X C V | то же |
| 36 | `&none` | `&kp G` |
| 37 | `&kp SPACE` | `&lt 8 M` — hold = GAME-2(8), tap = M |
| 38 | `&kp LEFT_ALT` | `&kp SPACE` |
| 39–40 | none | none |

### 4.7 SCROLL (idx 7 в старом, 6 в feature)

Содержимое не изменилось (все `&trans`); слой просто сместился 7→6. У main_ref/082ecea в этом
слое есть ASCII-комментарии-рамки, в feature — нет (косметика).

### 4.8 mouse (новый, idx 7)

qwerty.keymap:416–423: всё `&trans`, кроме thumb-клавиш: **37=`&mkp LCLK`, 38=`&mkp MCLK`,
39=`&mkp RCLK`** (36/40 = `&trans`). Активируется с BASE через J/M (`&lt 7`).

### 4.9 GAME-2 (новый, idx 8)

qwerty.keymap:425–432: всё `&trans`, кроме **12=`&kp I`, 24=`&kp J`, 25–29=`N1 N2 N3 N4 N5`**.
Доступен из GAME через M (`&lt 8`). Экспериментальный (клавиши I/J/N1–N5 в новом месте).

## 5. Combos

feature инлайнит 9 combos (qwerty.keymap:293–341). Сравнение с обоими исходниками:

| combo | binding | 082ecea | main_ref | feature |
|---|---|---|---|---|
| CapsWord | `&caps_word` | 17 18, L0 | то же | то же |
| combo_left_click | `&mkp LCLK` | **25 26** | то же | **28 27** (V+C) |
| combo_middle_click | `&mkp MCLK` | **26 27** | то же | **26 25** (X+Z) |
| combo_right_click | `&mkp RCLK` | **27 28** | то же | **26 27** (X+C) |
| combo_BASE_or_EXTRAS | `&td_bore` | 38 39 | то же | **удалён** |
| alt | `&kp LEFT_ALT` | — | — | **19 20** (J+K) |
| combo_left_click_two | `&mkp LCLK` | — | — | **31 32** (M+COMMA) |
| combo_middle_click_two | `&mkp MCLK` | — | — | **33 34** (PERIOD+SLASH) |
| combo_right_click_two | `&mkp RCLK` | — | — | **32 33** (COMMA+PERIOD) |
| back_to_base_layer | `&to 0` | — | — | **10 11** (P+LCTRL) |
| combo_esc_left | `&kp ESC` | — | 1 2, L0 | — |
| combo_esc_right | `&kp ESC` | — | 9 10, L0 | — |
| combo_bootloader_left | `&bootloader` | — | 0 24, L0 | — |
| combo_bootloader_right | `&bootloader` | — | 11 35, L0 | — |
| combo_bt_clear_safe | `&bt BT_CLR` | — | 0 11, L0 | — |
| combo_prev_desktop | `&desktop_prev` | — | 24 25, L0 | — |
| combo_next_desktop | `&desktop_next` | — | 34 35, L0 | — |
| combo_all_windows | `&ht_combo_all_windows` | — | 32 33, L0 | — |
| combo_app_windows | `&ht_combo_app_windows` | — | 31 32, L0 | — |

Примечания:
- Ни один из 9 combos feature не ограничено `layers` — работают на всех слоях.
- При переносе решить: объединить (union) или заменить. Конфликты позиций: `combo_left_click`
  (25 26 vs 28 27) и т.д. — один набор клавиш не может давать два разных клика; выбор за пользователем.
- Новые combos main_ref (esc/bootloader/desktop/windows) используют behaviors/макросы, которых нет
  в feature-файле (`ht_combo_*`, `desktop_*`) — при инлайне их нужно либо добавить, либо выкинуть.

## 6. Behaviors (инлайн в feature, строки 37–291)

По сравнению с `keymap_features/behaviors.dtsi` на 082ecea:

**Без изменений (24 узла + 3 define):** `mm_f1..mm_f12`, `ht_left`, `ht_right`, `lt_nav`,
`td_space_nav`, `mo_clk`, `mo_kp`, `td_bs`, `httl`, `td_layers`, `mm_qm_ex`, `mm_pr_col`,
`mm_cm_sc`; `KEYS_L`/`KEYS_R`/`THUMBS` (строки 34–36) — те же расширенные списки, что на 082ecea.

**Изменены (4):**

| узел | 082ecea | feature | эффект |
|---|---|---|---|
| `td_esc_scrl_slow` (строка 174) | `<&lt 7 ESC>, <&mo 6>` | `<&lt 5 ESC>, <&mo 3>` | hold-ESC: SCROLL(7)→GAME(5); double-tap: SLOW(6)→EXTRAS(3). **Не просто ренумбер** — смысл изменился |
| `td_clk` (строка 209) | `<&mo_clk 3 LCLK>, <&mkp LCLK>` | `<&mo_clk 0 LCLK>, <&mkp LCLK>` | hold: SYM(3)→BASE(0) (hold-действие фактически отключено) |
| `td_clk_scrl` (строка 253) | `<&mkp RCLK>, <&mo 8>` | `<&mkp RCLK>, <&mo 0>` | double-tap: `&mo 8` (на 082ecea — **выход за диапазон слоёв 0–7, латентный баг**) → BASE(0) |
| `td_bore` (строка 262) | `<&mo 5>, <&to 0>` | `<&mo 0>, <&to 0>` | hold: EXTRAS(5)→BASE(0) no-op |

**Весь инлайн — мёртвый код:** ни один из этих 28 узлов не используется в body кеймапа feature
(в BASE нет ни `&ht_*`, `&mm_*`, `&td_*`, `&bs_*`). Переносить их имеет смысл только если:
(a) вернуть использование (например, HRM в BASE), или (b) оставить как «резерв».

**Конфликты с текущим `keymap_features/behaviors.dtsi` main_ref** (если инлайн всё-таки переносить
в общий файл, а не в qwerty):
- `ht_left`/`ht_right`: feature = значения 082ecea (tapping 280, quick-tap 160, rpi 65/55,
  `retro-tap`, без `hold-trigger-on-release`); main_ref (после #42, behaviors.dtsi:102–125) =
  280/175/150 и 280/175/125, `hold-trigger-on-release`, без `retro-tap` + новые
  `ht_left_tp`/`ht_right_tp` (200/160/120, tap-preferred, `retro-tap`; behaviors.dtsi:128–151),
  используются в BASE main_ref (13, 21, 22).
- `mo_kp`+`td_bs` (feature) против `bs_num` (main_ref, behaviors.dtsi:191–199 — заменяет обе).
- `KEYS_L`/`KEYS_R`: feature = расширенные списки 082ecea (25/24 ключа); main_ref = урезанные
  (21/21, behaviors.dtsi:1–2).
- `td_bore` hold=BASE(0): если затронуть общий файл — `combo_BASE_or_EXTRAS` (38 39) в остальных
  4 кеймапах получит no-op hold.
- `ht_combo_all_windows`/`ht_combo_app_windows` (main_ref:229–244) + `combo_*_hold/tap`,
  `desktop_*` в `macros.dtsi` — нужны только для новых combos main_ref.

## 7. Trackball: `config/trackball/charybdis_pointer.dtsi`

| узел | 082ecea = main_ref | feature | комментарий |
|---|---|---|---|
| `scroller` (строка 10) | `layers = <7>` (SCROLL) | `layers = <1>` (NUM) | комментарий: «layer-8» → «layer-1» |
| `slow_pointer` (строка 20) | `layers = <6>` (SLOW) | `layers = <7>` (mouse) | комментарий: «layer-7» → «**layer-3**» — **неверный** (на самом деле 7) |
| base chain | `&zip_xy_scaler 5 5` | то же | |

Семантика: в feature быстрый скролл (scaler 1/15 + y-invert) активен на **NUM**, точный медленный
указатель (2/6) — на **mouse**; слой SCROLL(6) override'а не получает (base 5/5). В main_ref
классика: SCROLL(7) = скролл, SLOW(6) = медленный.

**Важное связывание:** файл включается из overlay'ов всех трёх trackball-shield'ов
(`boards/shields/charybdis_right_bt/charybdis_right_bt.overlay:3`,
`charybdis_right_dongle/...overlay:3`, `charybdis_dongle/...overlay:75`) — один файл на все
кеймапы. Остальные 4 кеймапа (canary, colemak_dh, focal, graphite) сохраняют старую карту
(SLOW=6, SCROLL=7): на ветке feature их SCROLL(7) получает slow-pointer, а SLOW(6) — base 5/5 —
тихий сбой их trackball-поведения. При переносе в main_ref: либо ренумберим слои во **всех 5**
кеймапах, либо возвращаем pointer-файл как в main_ref, либо делаем per-keymap варианты.

## 8. Мелочи

- `config/charybdis/charybdis.conf`: только добавлен trailing newline (e89a5c6 → 680f0c0).
  Переносить не обязательно.
- Устаревший блок `#define`-комментариев в feature qwerty (строки 24–32) — поправить при переносе.
- Опечатка `display-name = "sybol"` в SYM (feature qwerty.keymap:387).
- В feature NUM/SLOW-подобных слоёв убраны ASCII-рамки-комментарии (косметика).

## 9. Обязательное из main_ref (НЕ потерять при переносе)

Изменения main_ref после форка, которые затронут qwerty.keymap (подробности — статка в разделе 1):

1. **`prospector_brightness`** (#40/#42): include `keymap_features/prospector_brightness.dtsi`
   (qwerty:11), stub `pbl` в `#ifndef CONFIG_SHIELD_PROSPECTOR_ADAPTER` (qwerty:36–47),
   bindings `&pbl PBL_INC/PBL_TOG/PBL_DEC` в EXTRAS (04/15/16, qwerty:109–110). Feature предшествует
   этой фиче — при переносе их **обязательно** добавить обратно.
2. **HRM-тюнинг #42** (behaviors.dtsi:102–151): новые timings `ht_left`/`ht_right` +
   `ht_left_tp`/`ht_right_tp` — актуальные «правильные» значения для home-row; feature-копии
   устарели.
3. **`bs_num`** вместо `mo_kp`+`td_bs` (behaviors.dtsi:191–199) — используется в BASE main_ref (38).
4. **9 новых combos** (combos.dtsi:22–95) и 6 новых макросов (macros.dtsi: `gpush`,
   `combo_all_windows_hold/tap`, `combo_app_windows_hold/tap`, `desktop_prev/next`).
5. **#40**: Prospector dongle-варианты в build.yaml, `config/dongles/dongle_prospector*`,
   west.yml (+prospector-zmk-module), rework keymap-drawer (`draw_keymaps.yml`,
   `keymap-drawer/configs/`, `scripts/make_stacked.py`, `render_stacked_composite.js`).
6. **#43**: фикс `core.fileMode` в build.yml; `82231a1`: AGENTS.md.

## 10. Проблемы и решения до/при переносе

1. **`&sl 3` (BASE, позиция 35, feature qwerty.keymap:351) — undefined reference.** Shift-lock
   (`sl:`) отсутствует в upstream ZMK: в текущем `main` нет `app/dts/behaviors/shift_lock.dtsi`
   (перечень behavior-файлов проверен), в v0.2.0–v0.4.0 его тоже не было, в конфигурации репо
   `sl:` нигде не определяется. Кеймап feature **как есть не скомпилируется** против текущего
   zmk main. Решение: заменить на обычный `&kp`-ключ (например, `&kp LSHFT`/`&kp RSHFT`) либо
   реализовать shift-lock кастомным behavior.
2. **Связывание pointer-файла** (раздел 7) — решить судьбу SLOW/SCROLL у остальных 4 кеймапов.
3. **Layout BT/OUT в EXTRAS** — feature держит layout 082ecea, main_ref переложил
   (OUT_TOG=06, BT_SEL=07–10, BT_CLR=combo 0+11). Выбрать одну раскладку.
4. **`&mo 8` в `td_clk_scrl` на 082ecea** был out-of-range (слои 0–7) — латентный баг; в feature
   заменено на `&mo 0`. Узел мёртвый; при переносе в общий файл использовать значение feature
   (или целевой слой SCROLL=6, если восстанавливаем семантику).
5. **Комментарий slow_pointer «layer-3»** — поправить на реальное значение.
6. **Combos без `layers`** в feature — решают конфликты лигатур на всех слоях; при объединении
   с combos main_ref проверить пересечения позиций (19 20 vs —; 31 32 / 32 33 vs
   `combo_app_windows` 31 32 и `combo_all_windows` 32 33 из main_ref — **прямые конфликты
   позиций!** `combo_left_click_two` 31 32 == `combo_app_windows` 31 32,
   `combo_right_click_two` 32 33 == `combo_all_windows` 32 33).
7. **NUM**: feature-дизайн (цифры/F/стрелки) полностью заменяет numpad — подтвердить, что это
   окончательный вариант.
8. **GAME-2** (12=I, 24=J, 25–29=N1–N5) — эксперимент; переносить или нет?

## 11. Чек-лист переноса в `main_ref`

Статусы: `pending` / `ported` / `skipped` (решено не переносить).

| ID | Пункт | Что сделать | Файлы | Статус |
|---|---|---|---|---|
| P1 | Каркас qwerty | Взять структуру feature (инлайн behaviors/combos или includes — решить); сохранить pbl-stub + include prospector_brightness из main_ref | `config/keymaps/qwerty.keymap:11,36–47` | pending |
| P2 | Карта слоёв | Внедрить 9 слоёв (EXTRAS=3, SYM=4, GAME=5, SCROLL=6, mouse=7, GAME-2=8), удалить SLOW; поправить блок #define-комментариев | qwerty.keymap:24–32, 343–433 | pending |
| P3 | BASE | Plain QWERTY; J/M=`&lt 7`; 37=`&lt 1 ESCAPE`; 38=`&mt LSHFT SPACE`; 39=`&lt 4 ENTER`; 40=BSPC; 36=LEFT_WIN; **позиция 35 — после решения 10.1** | qwerty.keymap:349–352 | pending |
| P4 | NUM | Новый дизайн (цифры/F/стрелы/shift/delete) | qwerty.keymap:356–364 | pending |
| P5 | NAV | Удалить 7 tmux-макросов → `&none` | qwerty.keymap:366–374 | pending |
| P6 | EXTRAS | Merge: sys_reset(01), OUT_BLE(18), OUT_USB(30), убрать fun-макросы; **сохранить** PBL(04/15/16); layout BT/OUT — по решению 10.3 | qwerty.keymap:376–384 | pending |
| P7 | SYM | Новый plain-дизайн; `12=&to 5`; поправить `display-name` («Sym»); при необходимости вернуть editor-макросы, если они нужны | qwerty.keymap:386–394 | pending |
| P8 | GAME | Новый дизайн (ESC, `06=&to 0`, Z вместо Y, `37=&lt 8 M`) | qwerty.keymap:396–404 | pending |
| P9 | mouse + GAME-2 | Добавить новые слои (решение 10.8 по GAME-2) | qwerty.keymap:416–432 | pending |
| P10 | Combos | Union/replace по решению; разрешить конфликты позиций (10.6); обновить общий `combos.dtsi` или инлайн | qwerty.keymap:293–341; `keymap_features/combos.dtsi` | pending |
| P11 | Behaviors | Инлайн: 4 изменённых узла (174, 209, 253, 262); **не затирать** main_ref-значения ht_left/ht_right/ht_*_tp/bs_num, если HRM остаётся в общих кеймапах; dead code можно выкинуть | qwerty.keymap:37–291; `keymap_features/behaviors.dtsi` | pending |
| P12 | Pointer | `scroller: 7→1`, `slow_pointer: 6→7`, поправить комментарии (10.5); учесть 10.2 (другие 4 кеймапа) | `config/trackball/charybdis_pointer.dtsi:10,20` | pending |
| P13 | conf | trailing newline — по желанию (нефункционально) | `config/charybdis/charybdis.conf` | pending |
| P14 | Рисунки | После правок кеймапа: `act -W .github/workflows/draw_keymaps.yml --bind --reuse` (или пуш в main — авто-коммит артефактов) | `keymap-drawer/*` | pending |
| P15 | Сборка/проверка | Полный сборка всех вариантов build.yaml (Docker) + прошивка и ручной тест (см. раздел 12) | — | pending |

## 12. Валидация

- Сборка: `docker-compose -f local-build/docker-compose.yml run --rm builder` (все варианты
  build.yaml). Успех `west build` = минимальная проверка.
- Кеймап: parse через keymap-drawer (workflow выше); при ручном `keymap parse` — symlink
  `config/keymap_features` в `config/keymaps/` (как в AGENTS.md).
- Прошивка (из README): при смене BT↔dongle-режимов сначала прошевать `settings_reset` на все
  устройства; для Prospector dongle-сборок — питание dongle до left, потом right.
- Проверить на устройстве: mouse-слой (J/M), NUM-скролл (trackball на NUM), slow pointer на
  mouse-слое, новые combos (J+K=Alt и т.д.), sys_reset, BT/OUT-переключение, PBL-яркость.
