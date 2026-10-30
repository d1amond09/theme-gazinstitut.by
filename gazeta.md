# Плагин «Автоматизация архива газеты БЕЛТОПГАЗ» — инструкция

**Версия плагина:** 1.2  
**WordPress:** 5.5+  
**PHP:** 7.4+  
**Автор:** Maksim Berdnikov (https://github.com/d1amond09)  
**Text Domain:** `gazeta-archive`

---

## Содержание

1. [Что делает плагин](#1-что-делает-плагин)
2. [Установка и активация](#2-установка-и-активация)
3. [Структура плагина](#3-структура-плагина)
4. [Админка: управление выпусками](#4-админка-управление-выпусками)
5. [Шорткод `[gazeta_archive]`](#5-шорткод-gazeta_archive)
6. [Отображение на сайте](#6-отображение-на-сайте)
7. [CSS: где менять стили](#7-css-где-менять-стили)
8. [Сценарии использования](#8-сценарии-использования)
9. [Перенос dev → production](#9-перенос-dev--production)
10. [Чек-лист перед запуском](#10-чек-лист-перед-запуском)
11. [Диагностика проблем](#11-диагностика-проблем)
12. [Хуки и кастомизация](#12-хуки-и-кастомизация)

---

## 1. Что делает плагин

Плагин добавляет в WordPress **отдельный тип записей «Выпуски газеты»** и **шорткод для автоматического вывода архива** PDF-выпусков отраслевой газеты «БЕЛТОПГАЗ. Газоснабжение и торфопереработка».

### Возможности

- Собственный CPT `gazeta_issue` в админке (иконка «документ»).
- Метабокс с полями: номер выпуска, дата, PDF-файл (через медиабиблиотеку), флаг «Свежий выпуск».
- **Автоматическая группировка** выпусков по годам (год извлекается из поля «Дата выпуска» через регулярное выражение).
- **Автоматическая сортировка**: сначала свежий выпуск, затем — по номеру (по убыванию).
- **Единственный «свежий» выпуск**: при установке галочки «Свежий выпуск» у нового выпуска — у старого флаг снимается автоматически.
- Шорткод `[gazeta_archive]` — выводит архив на любой странице (можно задать свой заголовок и картинку через атрибуты).
- Работает **без сторонних плагинов** (медиабиблиотека — стандартная WordPress).

---

## 2. Установка и активация

### 2.1. Установка

**Вариант А — через админку:**

1. Скачать ZIP с плагином.
2. **Плагины → Добавить новый → Загрузить плагин** → выбрать ZIP → **Установить** → **Активировать**.

**Вариант Б — через FTP:**

1. Распаковать архив в `/wp-content/plugins/gazeta-archive/`.
2. Админка → **Плагины** → найти **«Автоматизация архива газеты БЕЛТОПГАЗ»** → **Активировать**.

### 2.2. Что появится после активации

- В левом меню админки — пункт **«Выпуски газеты»** (иконка документа).
- Шорткод `[gazeta_archive]` становится доступным для вставки в любую страницу/запись.

### 2.3. Структура плагина

```
gazeta-archive/
├── gazeta-archive.php     # единственный файл плагина (CPT + метабокс + шорткод)
└── (опционально)
    ├── assets/
    │   ├── css/
    │   │   └── gazeta.css
    │   └── js/
    │       └── gazeta.js
    └── readme.txt
```

> Плагин можно оставить одним файлом — он самодостаточен. CSS для фронтенда можно держать в теме или подключить отдельно (см. [раздел 7](#7-css-где-менять-стили)).

---

## 3. Структура плагина

### 3.1. Основные функции

| Функция | Что делает |
|---|---|
| `gazeta_create_issue_cpt()` | Регистрирует CPT `gazeta_issue` |
| `gazeta_admin_scripts($hook)` | Подключает `wp.media` и inline-скрипт выбора PDF |
| `gazeta_add_meta_boxes()` | Добавляет метабокс «Детали выпуска» |
| `gazeta_issue_meta_box_callback($post)` | Рендер полей метабокса |
| `gazeta_save_meta_box_data($post_id)` | Сохранение метаполей + автоснятие старого «свежего» |
| `gazeta_archive_shortcode($atts)` | Генерация HTML архива (шорткод) |

### 3.2. Метаполя записи

| Мета-ключ | Тип | Что хранит |
|---|---|---|
| `_gazeta_issue_number` | string | Номер выпуска, например `№ 8 (114)` |
| `_gazeta_issue_date` | string | Дата, например `август 2026` |
| `_gazeta_pdf_url` | string (URL) | Ссылка на PDF-файл |
| `_gazeta_pdf_id` | int | ID вложения в медиабиблиотеке |
| `_gazeta_is_latest` | `'1'` или `''` | Флаг «свежий выпуск» |

---

## 4. Админка: управление выпусками

### 4.1. Где найти

**Админка → Выпуски газеты** (иконка документа слева).

### 4.2. Как добавить новый выпуск

**Выпуски газеты → Добавить новый выпуск**:

| Поле | Обязательное | Описание |
|---|---|---|
| **Заголовок** | ✅ | Название выпуска (например `Выпуск № 8 (114)`) |
| **Номер выпуска** | — | Отображается на карточке, например `№ 8 (114)` |
| **Дата выпуска** | — | Например `август 2026`. **Год из этой строки используется для группировки** |
| **Файл выпуска (PDF)** | — | Через кнопку «Выбрать или загрузить PDF» (медиабиблиотека) |
| **Свежий выпуск** | — | Чекбокс — бейдж «Свежий выпуск» + оранжевое выделение карточки |

### 4.3. Как указать дату, чтобы год распознался

Плагин ищет четырёхзначный год в строке через регулярку `\b((?:19|20)\d{2})\b`. Подходят варианты:

- `август 2026`
- `Август 2026`
- `2026`
- `№8 2026`
- `08.2026`

Если год **не найден** — используется год публикации записи (`get_the_date('Y')`).

### 4.4. Как установить «свежий выпуск»

1. Открыть запись **самого нового** выпуска.
2. Поставить галочку **«Свежий выпуск»**.
3. **Обновить**.

При сохранении плагин:
- ставит флаг `_gazeta_is_latest = 1` текущей записи;
- ищет все остальные записи с этим флагом и **снимает его**.

> То есть «свежий» всегда **только один**. Если нужно показать несколько «свежих» — придётся отказаться от галочки и добавить логику через фильтр (см. [раздел 12](#12-хуки-и-кастомизация)).

### 4.5. Редактирование и удаление

Стандартные действия WordPress:
- **Редактировать** — навести на выпуск → «Редактировать».
- **Удалить** — «Удалить» → в корзину → удалить окончательно.
- **Просмотр** — ссылка на архив не ведёт (см. [раздел 6](#6-отображение-на-сайте)), ведёт на одиночную запись.

### 4.6. Порядок в админке

По умолчанию — по дате публикации (новые сверху). На фронтенде порядок **пересобирается** шорткодом по году и номеру выпуска (см. [раздел 5](#5-шорткод-gazeta_archive)).

---

## 5. Шорткод `[gazeta_archive]`

### 5.1. Базовый вызов

```
[gazeta_archive]
```

### 5.2. Атрибуты

| Атрибут | По умолчанию | Описание |
|---|---|---|
| `title` | `Газета «БЕЛТОПГАЗ. Газоснабжение и торфопереработка»` | Заголовок страницы |
| `image` | `https://gazinstitut.by/wp-json/webdav-storage/v1/file/2026/09/IMG_7565-2.png` | Картинка-шапка над архивом |

### 5.3. Примеры

**Свой заголовок и картинка:**

```
[gazeta_archive title="Архив газеты «БЕЛТОПГАЗ»" image="https://gazinstitut.by/wp-content/uploads/2026/01/gazeta-header.jpg"]
```

**Только заголовок (картинка по умолчанию):**

```
[gazeta_archive title="Наши выпуски"]
```

**Вставить в виджет/блок:**

Добавить шорткод в **Виджеты** (через блок «Шорткод») или в **Gutenberg** (блок «Шорткод»).

### 5.4. Что выводится

1. **H1-заголовок** (из атрибута `title`).
2. **Картинка-шапка** (из атрибута `image`).
3. **Блок «О газете»** — захардкожен в плагине (учредители, цели и задачи).
4. **Секция «Архив номеров»** — `<details>` по годам:
   - Заголовок года: `📅 2026 год — N выпусков ▼`
   - Текущий год открыт по умолчанию (`open`).
   - Внутри — сетка карточек выпусков.

### 5.5. Карточка выпуска

- Иконка PDF
- Номер выпуска
- Дата выпуска
- Кнопка **«📖 Читать»** → PDF в новой вкладке
- Кнопка **«Скачать»** (иконка) → скачивание PDF
- Бейдж **«Свежий выпуск»** — если стоит флаг

### 5.6. Логика сортировки

1. Годы — по убыванию (`krsort`).
2. Внутри года: сначала `_gazeta_is_latest = 1`, затем по убыванию номера (регулярка вытаскивает первое число из `№ 8 (114)` → `8`).

---

## 6. Отображение на сайте

### 6.1. Где показывается архив

**Только там, где вставлен шорткод** `[gazeta_archive]`. Обычно — на отдельной странице **«Газета»** (в меню темы есть пункт «Газета» → `/newspaper/`).

### 6.2. Как сделать страницу архива

1. **Страницы → Добавить новую**.
2. Заголовок: `Газета`.
3. Слаг: `newspaper` (или `gazeta`).
4. В содержимое вставить шорткод:
   ```
   [gazeta_archive]
   ```
5. Опубликовать.
6. Если пункт меню «Газета» ведёт не туда — **Внешний вид → Меню** → отредактировать ссылку.

### 6.3. Одиночные страницы выпусков

CPT `public => true` — у каждого выпуска есть своя страница по адресу `/gazeta_issue/название/`. Но по умолчанию шаблон `single.php` темы может **не показывать PDF**. Если нужно:

- либо использовать плагин для просмотра PDF (`PDF Embedder`, `Real3D Flipbook`),
- либо добавить в `single.php` проверку `if (get_post_type() === 'gazeta_issue') { ... }` и вывести `_gazeta_pdf_url` в `<iframe>` / `<embed>`,
- либо скрыть одиночные страницы через фильтр (см. [раздел 12](#12-хуки-и-кастомизация)).

### 6.4. Архив CPT

С `has_archive => true` доступен `/gazeta_issue/` — **стандартный архив WordPress** (список записей без группировки по годам и без шорткода). Скорее всего, он не нужен — его можно отключить фильтром (см. [раздел 12](#12-хуки-и-кастомизация)).

---

## 7. CSS: где менять стили

Плагин **не подключает** собственный CSS — все классы (`.gazeta-wrapper`, `.gazeta-card`, `.year-block` и т.д.) стилизуются **в теме**. Это сделано, чтобы не дублировать стили и не ломать дизайн сайта.

### 7.1. Классы, на которые можно стилизоваться

```
.gazeta-wrapper            — общий контейнер
.page-title                — H1-заголовок
.gazeta-header-image       — картинка-шапка
.gazeta-description        — блок «О газете»
.goals-list                — список целей
.founders                  — строка «Учредители»

.archive-title             — заголовок «Архив номеров»
.year-block                — <details> по году
.year-summary              — заголовок года
.year-icon, .year-label, .year-count, .year-toggle
.year-content              — контент года
.gazeta-grid               — сетка карточек

.gazeta-card               — карточка выпуска
.gazeta-card.latest        — свежий выпуск
.latest-badge              — бейдж «Свежий выпуск»
.card-header               — шапка карточки (иконка + номер + дата)
.pdf-icon                  — SVG-иконка PDF
.card-number               — номер выпуска
.card-date                 — дата выпуска
.card-body                 — тело карточки
.card-actions              — кнопки
.card-btn                  — кнопка «Читать»
.card-download             — кнопка «Скачать» (иконка)
```

### 7.2. Как подключить стили

**Вариант А — в `style.css` темы** (самый простой):

Добавить в конец `style.css` (или в отдельный `gazeta.css` темы):

```css
/* ===== Архив газеты ===== */
.gazeta-wrapper{
    max-width: 1200px;
    margin: 0 auto;
    padding: 30px 20px 40px;
}
.gazeta-wrapper .page-title{
    font-size: 2rem;
    font-weight: 800;
    color: var(--ink, #0b2b44);
    margin: 0 0 24px;
    border-left: 6px solid var(--accent, #2a7bcb);
    padding-left: 18px;
}
.gazeta-header-image{
    margin: 0 0 30px;
    border-radius: 20px;
    overflow: hidden;
    box-shadow: 0 12px 32px -14px rgba(0,20,60,.25);
}
.gazeta-header-image img{
    width: 100%;
    height: auto;
    display: block;
}
.gazeta-description{
    background: linear-gradient(145deg, #fff, #f4f8fc);
    border: 1px solid #e9eef3;
    border-radius: 20px;
    padding: 24px 26px;
    margin: 0 0 30px;
    line-height: 1.7;
    color: #334155;
}
.goals-list{
    margin: 10px 0 14px;
    padding-left: 22px;
}
.goals-list li{
    margin: 6px 0;
    line-height: 1.55;
}
.founders{
    display: inline-block;
    margin-top: 12px;
    padding: 8px 16px;
    background: #e6f0fa;
    color: var(--accent-dark, #0f3a5f);
    border-radius: 20px;
    font-weight: 700;
    font-size: .9rem;
}
.archive-title{
    font-size: 1.5rem;
    font-weight: 800;
    color: var(--ink, #0b2b44);
    margin: 30px 0 20px;
    display: flex;
    align-items: center;
    gap: 10px;
}
.year-block{
    background: #fff;
    border: 1px solid #e9eef3;
    border-radius: 18px;
    margin: 0 0 14px;
    overflow: hidden;
    box-shadow: 0 6px 18px -10px rgba(0,20,60,.12);
}
.year-summary{
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 16px 20px;
    background: linear-gradient(145deg, #f8fbfe, #eef4fa);
    cursor: pointer;
    font-weight: 700;
    color: var(--ink, #0b2b44);
    list-style: none;
    transition: background .2s;
}
.year-summary:hover{background: #eef4fa;}
.year-summary::-webkit-details-marker{display: none;}
.year-icon{font-size: 1.2rem;}
.year-label{flex: 1;font-size: 1.05rem;}
.year-count{
    font-size: .82rem;
    font-weight: 600;
    color: var(--muted, #64748b);
    background: #fff;
    padding: 4px 10px;
    border-radius: 20px;
    border: 1px solid #e9eef3;
}
.year-toggle{transition: transform .25s;color: var(--accent);}
.year-block[open] .year-toggle{transform: rotate(180deg);}
.year-content{padding: 20px;}
.gazeta-grid{
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(260px, 1fr));
    gap: 18px;
}
.gazeta-card{
    position: relative;
    background: #fff;
    border: 1px solid #e9eef3;
    border-top: 4px solid var(--accent, #2a7bcb);
    border-radius: 16px;
    padding: 18px;
    display: flex;
    flex-direction: column;
    transition: transform .25s, box-shadow .3s, border-top-color .25s;
}
.gazeta-card:hover{
    transform: translateY(-4px);
    box-shadow: 0 16px 32px -14px rgba(0,20,60,.22);
    border-top-color: var(--accent-dark, #0f3a5f);
}
.gazeta-card.latest{
    border-top-color: #d97706;
    background: linear-gradient(180deg, #fffbf3 0%, #fff 40%);
}
.latest-badge{
    position: absolute;
    top: -1px;
    right: 14px;
    background: linear-gradient(135deg, #f59e0b, #d97706);
    color: #fff;
    font-size: .7rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: .5px;
    padding: 5px 12px;
    border-radius: 0 0 10px 10px;
    box-shadow: 0 6px 14px -6px rgba(217,119,6,.7);
}
.card-header{
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding-bottom: 14px;
    border-bottom: 1px dashed #e2e8f0;
    margin-bottom: 14px;
}
.pdf-icon svg{
    width: 32px;
    height: 32px;
    fill: var(--accent, #2a7bcb);
}
.gazeta-card.latest .pdf-icon svg{fill: #d97706;}
.card-number{
    font-size: 1.1rem;
    font-weight: 800;
    color: var(--ink, #0b2b44);
    line-height: 1.3;
}
.card-date{
    font-size: .85rem;
    color: var(--muted, #64748b);
    font-weight: 500;
}
.card-actions{
    display: flex;
    align-items: center;
    gap: 10px;
    margin-top: auto;
}
.card-btn{
    flex: 1;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;
    padding: 10px 16px;
    background: linear-gradient(135deg, #2a7bcb, #1a4c7a);
    color: #fff;
    border-radius: 24px;
    font-weight: 700;
    font-size: .88rem;
    text-decoration: none;
    transition: transform .2s, box-shadow .25s;
    box-shadow: 0 6px 16px -8px rgba(42,123,203,.65);
}
.card-btn:hover{
    transform: translateY(-2px);
    box-shadow: 0 12px 22px -8px rgba(42,123,203,.85);
}
.gazeta-card.latest .card-btn{
    background: linear-gradient(135deg, #f59e0b, #d97706);
    box-shadow: 0 6px 16px -8px rgba(217,119,6,.65);
}
.card-download{
    display: inline-flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    border-radius: 50%;
    background: #f0f4f8;
    color: var(--accent, #2a7bcb);
    transition: background .2s, color .2s, transform .2s;
}
.card-download svg{width: 18px; height: 18px; fill: currentColor;}
.card-download:hover{
    background: var(--accent, #2a7bcb);
    color: #fff;
    transform: scale(1.08);
}
@media (max-width: 640px){
    .gazeta-wrapper .page-title{font-size: 1.5rem;}
    .gazeta-grid{grid-template-columns: 1fr;}
    .year-summary{padding: 14px 16px;}
    .year-content{padding: 14px;}
}
```

**Вариант Б — отдельный файл, подключаемый из плагина:**

Создать в плагине папку `assets/css/gazeta.css`, добавить в основной файл плагина:

```php
add_action('wp_enqueue_scripts', function () {
    if (is_singular() && has_shortcode(get_post()->post_content, 'gazeta_archive')) {
        wp_enqueue_style(
            'gazeta-archive',
            plugin_dir_url(__FILE__) . 'assets/css/gazeta.css',
            [],
            '1.2'
        );
    }
});
```

Это подгрузит CSS **только** на страницах, где реально есть шорткод.

**Вариант В — через `wp_add_inline_style`:**

Если не хочется отдельного файла:

```php
add_action('wp_enqueue_scripts', function () {
    if (is_singular() && has_shortcode(get_post()->post_content, 'gazeta_archive')) {
        wp_register_style('gazeta-archive-inline', false);
        wp_enqueue_style('gazeta-archive-inline');
        wp_add_inline_style('gazeta-archive-inline', '/* ваш CSS */');
    }
});
```

---

## 8. Сценарии использования

### 8.1. Добавление нового выпуска (ежемесячно)

1. **Выпуски газеты → Добавить новый выпуск**.
2. Заголовок: `Выпуск № 8 (114)`.
3. Номер выпуска: `№ 8 (114)`.
4. Дата: `август 2026`.
5. PDF: выбрать из медиабиблиотеки.
6. Галочка **«Свежий выпуск»** — поставить.
7. **Опубликовать**.

Предыдущий «свежий» автоматически перестанет быть свежим. На сайте архив пересоберётся: новый выпуск попадёт в группу «2026», будет первым.

### 8.2. Перенос выпусков со старого сайта

1. Загрузить все PDF в медиабиблиотеку (**Медиафайлы → Добавить**).
2. Для каждого — **Добавить новый выпуск**:
   - Заголовок = номер
   - Номер = номер
   - Дата = `месяц год`
   - PDF — выбрать из библиотеки
   - «Свежий выпуск» — только у самого нового
3. Опубликовать.

> Если выпусков много (десятки) — можно написать скрипт-импортёр, который читает список PDF из папки и создаёт записи через `wp_insert_post` + `update_post_meta`. Плагин не мешает — все метаполя стандартные.

### 8.3. Сезонная смена «свежего»

Если «свежий» нужно снять, но новый ещё не готов:

1. Открыть текущий «свежий».
2. Снять галочку.
3. Сохранить.

Бейдж исчезнет, оранжевое выделение снимется.

### 8.4. Скрыть архив на время

Временно снять страницу с публикации (**Страницы → Газета → Черновик**) или убрать шорткод с содержимого.

---

## 9. Перенос dev → production

### 9.1. Перенос самого плагина

Плагин — один PHP-файл (+ CSS, если добавляли). Переносится вместе с темой через FTP:

```
/wp-content/plugins/gazeta-archive/
```

**Важно:** активировать плагин на production **после** переноса БД и файлов темы.

### 9.2. Перенос контента (записи + метаполя + PDF)

**Вариант А — через экспорт/импорт WordPress:**

1. На dev: **Инструменты → Экспорт → Выпуски газеты** (или всё содержимое).
2. Скачать XML.
3. На prod: **Инструменты → Импорт → WordPress** → загрузить XML.
4. **Медиафайлы** (PDF) импортируются автоматически, если стоит галочка «Скачать и импортировать вложения».
5. После импорта проверить, что метаполя на месте: открыть любой выпуск в админке — номер, дата, PDF должны подтянуться.

**Вариант Б — полный перенос БД:**

1. `mysqldump -u devuser -p devdb > dev.sql`
2. `sed -i 's/dev\.gazinstitut\.by/gazinstitut.by/g' dev.sql`
3. На prod: `mysql -u produser -p proddb < dev.sql`
4. На prod: **Настройки → Постоянные ссылки → Сохранить** (перегенерирует rewrite для CPT).

**Вариант В — только записи CPT через WP-CLI:**

```bash
# На dev
wp post list --post_type=gazeta_issue --fields=ID,post_title --format=ids > issues.txt
wp export --post_type=gazeta_issue --dir=/tmp --filename_format=gazeta.xml

# На prod
wp import /tmp/gazeta.xml --authors=create
```

### 9.3. Перегенерация ссылок

После переноса **обязательно**:

**Настройки → Постоянные ссылки → Сохранить** (просто нажать, ничего не менять).

Это пересоздаст `.htaccess`-правила и зарегистрирует rewrite для `/gazeta_issue/`.

### 9.4. Проверка после переноса

- [ ] В админке **«Выпуски газеты»** содержит те же записи, что на dev.
- [ ] У каждой записи заполнены поля (номер, дата, PDF, флаг).
- [ ] PDF-файлы открываются (ссылки ведут на prod-домен).
- [ ] Страница `/newspaper/` (или другая с шорткодом) отображает архив.
- [ ] Группировка по годам корректна.
- [ ] «Свежий выпуск» — один, у самого нового.

### 9.5. Что делать с dev-версией

- **Отключить индексацию**: `robots.txt` с `Disallow: /`.
- **Отключить плагин**, если dev больше не нужен.
- Или удалить полностью (`wp plugin delete gazeta-archive`).

---

## 10. Чек-лист перед запуском

### Плагин

- [ ] Плагин активирован на production.
- [ ] CPT «Выпуски газеты» зарегистрирован (пункт в админке есть).
- [ ] Метабокс «Детали выпуска» отображается.
- [ ] Кнопка «Выбрать или загрузить PDF» работает (открывается медиабиблиотека).
- [ ] PDF после выбора отображается в превью.
- [ ] «Свежий выпуск» сохраняется и снимается у старого.
- [ ] Шорткод `[gazeta_archive]` рендерится на странице.

### Контент

- [ ] Все выпуски добавлены.
- [ ] У всех заполнены номер, дата, PDF.
- [ ] Группировка по годам правильная (год извлекается из даты).
- [ ] Внутри года сортировка по номеру корректна.
- [ ] «Свежий» — ровно один и это действительно самый новый.

### Фронтенд

- [ ] Страница архива открывается.
- [ ] Годы свёрнуты, кроме текущего (или все — в зависимости от настройки `<details>`).
- [ ] Карточки кликабельны.
- [ ] «Читать» открывает PDF в новой вкладке.
- [ ] «Скачать» скачивает PDF.
- [ ] Бейдж «Свежий выпуск» на нужной карточке.
- [ ] На мобильных сетка в одну колонку.

### Безопасность

- [ ] PDF-файлы доступны только с вашего домена (или — по решению — публично).
- [ ] Все поля метабокса санитизируются при сохранении (`sanitize_text_field`, `esc_url_raw`).

---

## 11. Диагностика проблем

### 11.1. Пункта «Выпуски газеты» нет в админке

- Проверить, что плагин **активен** (**Плагины**).
- Проверить, что файл `gazeta-archive.php` лежит в `/wp-content/plugins/gazeta-archive/`.
- Проверить `wp-content/debug.log` на фатальные ошибки.
- Пересохранить постоянные ссылки (иногда нужно для CPT).

### 11.2. Шорткод выводится как текст `[gazeta_archive]`

**Причина:** плагин не активен или шорткод вставлен в блок, который не обрабатывает шорткоды.

**Решение:**

1. **Плагины** → проверить, что **«Автоматизация архива газеты БЕЛТОПГАЗ»** активен.
2. Проверить, что шорткод вставлен в **Страницу/Запись** (а не в виджет без обработки шорткодов).
3. Если в **Gutenberg** — использовать блок **«Шорткод»**, а не «Параграф».
4. Если в **Elementor / WPBakery** — использовать виджет «Шорткод» / «Text Block» с обработкой.

### 11.3. PDF не выбирается

- **F12 → Console** — есть ошибка `wp.media is not defined`?
- Проверить, что на странице редактирования `gazeta_issue` подключается `wp_enqueue_media()`. В `functions.php` темы **не должно быть** кода, отключающего медиабиблиотеку.

### 11.4. Выпуски не группируются по годам

**Причина:** год не извлекается из поля «Дата выпуска».

**Проверка:** откройте любой выпуск, посмотрите значение поля «Дата выпуска». Должен быть год в формате `2026`. Плагин использует регулярку `\b((?:19|20)\d{2})\b`.

**Решение:** привести дату к виду `август 2026` (или `2026`). Если год не указан — используется год публикации записи.

### 11.5. «Свежий выпуск» не снимается у старого

**Причина:** флаг был выставлен вручную в БД или плагин `save_post` не сработал.

**Решение:**

1. Открыть старый «свежий» → снять галочку → сохранить.
2. Или проверить в БД: `SELECT post_id FROM wp_postmeta WHERE meta_key = '_gazeta_is_latest' AND meta_value = '1'`. Должна быть максимум одна запись.

### 11.6. Архив пуст, хотя выпуски есть

- Проверить, что записи **опубликованы** (не черновики, не в корзине).
- Проверить, что у записей заполнено поле `_gazeta_pdf_url` — без PDF карточка не выводится (в коде `href` берётся из этого поля).
- Открыть `wp-content/debug.log`.

### 11.7. 404 на `/gazeta_issue/...` или `/gazeta_issue/` (архив)

**Причина:** rewrite не пересобрался.

**Решение:** **Настройки → Постоянные ссылки → Сохранить**.

### 11.8. Ошибка «The link you followed has expired» при загрузке PDF

- Увеличить `upload_max_filesize` и `post_max_size` в `php.ini` (для больших PDF).
- Проверить `WP_MEMORY_LIMIT` в `wp-config.php` (рекомендуется 256M+).
- Проверить права на `/wp-content/uploads/`.

### 11.9. На странице архива применяются стили, но не все

- Проверить, что CSS для `.gazeta-*` классов подключён (см. [раздел 7](#7-css-где-менять-стили)).
- **F12 → Elements** → клик на `.gazeta-card` → справа должны быть правила из вашего CSS.
- Если правил нет — файл не подключён, или селектор не совпадает.

### 11.10. Сайт «поехал» после активации плагина

- Плагин **не должен** влиять на что-либо кроме CPT `gazeta_issue` и шорткода.
- Если сломалась вёрстка — вероятно, конфликт CSS-классов (например, ваш CSS использует `.page-title` глобально, а этот класс есть и в теме). Решение — обернуть все стили в `.gazeta-wrapper`:

```css
.gazeta-wrapper .page-title { /* ... */ }
```

---

## 12. Хуки и кастомизация

Плагин **не предоставляет** своих фильтров и действий в текущей версии (1.2). Всё меняется через **стандартные хуки WordPress**.

### 12.1. Отключить одиночные страницы выпусков

```php
add_filter('register_post_type_args', function ($args, $post_type) {
    if ($post_type === 'gazeta_issue') {
        $args['publicly_queryable'] = false;
        $args['rewrite'] = false;
    }
    return $args;
}, 10, 2);
```

### 12.2. Отключить архив CPT `/gazeta_issue/`

```php
add_filter('register_post_type_args', function ($args, $post_type) {
    if ($post_type === 'gazeta_issue') {
        $args['has_archive'] = false;
    }
    return $args;
}, 10, 2);
```

### 12.3. Добавить своё поле в метабокс

**Вариант А — через `add_meta_box`:**

```php
add_action('add_meta_boxes', function () {
    add_meta_box(
        'gazeta_extra',
        'Дополнительно',
        function ($post) {
            $value = get_post_meta($post->ID, '_gazeta_extra', true);
            echo '<input type="text" name="gazeta_extra" value="' . esc_attr($value) . '" style="width:100%">';
        },
        'gazeta_issue',
        'normal',
        'default'
    );
});

add_action('save_post_gazeta_issue', function ($post_id) {
    if (isset($_POST['gazeta_extra'])) {
        update_post_meta($post_id, '_gazeta_extra', sanitize_text_field($_POST['gazeta_extra']));
    }
});
```

### 12.4. Разрешить несколько «свежих» выпусков

Плагин **принудительно** снимает флаг у всех остальных при сохранении. Чтобы разрешить несколько — нужно убрать этот блок из `gazeta_save_meta_box_data`. Проще всего — переопределить поведение через `save_post` с более высоким приоритетом:

```php
// Отключить автоснятие флага (сохранить только за текущей записью)
add_action('save_post_gazeta_issue', function ($post_id) {
    if (defined('DOING_AUTOSAVE') && DOING_AUTOSAVE) return;
    if (!current_user_can('edit_post', $post_id)) return;

    // Восстанавливаем только что сохранённый флаг
    if (isset($_POST['gazeta_is_latest']) && $_POST['gazeta_is_latest'] === '1') {
        update_post_meta($post_id, '_gazeta_is_latest', '1');
    }
}, 20);
```

> Но плагин при сохранении всё равно пробежится и сбросит — приоритет 20 выполнится **после** плагина, поэтому флаг восстановится. Это «костыль», но рабочий.

### 12.5. Изменить заголовок по умолчанию для шорткода

```php
add_filter('shortcode_atts_gazeta_archive', function ($out, $pairs, $atts) {
    if (empty($atts['title'])) {
        $out['title'] = 'Наш архив';
    }
    return $out;
}, 10, 3);
```

### 12.6. Кастомный шаблон вывода шорткода

Шорткод собирает HTML внутри `gazeta_archive_shortcode()`. Чтобы использовать **свой шаблон**, скопируйте функцию в свой плагин/тему и переопределите шорткод с более поздним приоритетом:

```php
add_action('init', function () {
    remove_shortcode('gazeta_archive');
    add_shortcode('gazeta_archive', 'my_gazeta_archive_shortcode');
});

function my_gazeta_archive_shortcode($atts) {
    // ... ваш HTML ...
}
```

### 12.7. Экспорт архива в JSON (для интеграций)

```php
add_action('rest_api_init', function () {
    register_rest_route('gazeta/v1', '/issues', [
        'methods'  => 'GET',
        'callback' => function () {
            $issues = get_posts([
                'post_type'      => 'gazeta_issue',
                'posts_per_page' => -1,
                'orderby'        => 'date',
                'order'          => 'DESC',
            ]);
            return array_map(function ($p) {
                return [
                    'id'       => $p->ID,
                    'title'    => $p->post_title,
                    'number'   => get_post_meta($p->ID, '_gazeta_issue_number', true),
                    'date'     => get_post_meta($p->ID, '_gazeta_issue_date', true),
                    'pdf'      => get_post_meta($p->ID, '_gazeta_pdf_url', true),
                    'is_latest'=> (bool) get_post_meta($p->ID, '_gazeta_is_latest', true),
                ];
            }, $issues);
        },
        'permission_callback' => '__return_true',
    ]);
});
```

Затем: `GET /wp-json/gazeta/v1/issues` — массив выпусков в JSON.

---

## Контакты

- **Автор:** Maksim Berdnikov (https://github.com/d1amond09)
- **Тема:** ГИПК «ГАЗ-ИНСТИТУТ»
- **Сайт:** https://gazinstitut.by/
- **Документация темы:** см. README.md темы
```

---
