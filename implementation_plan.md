# Оптимизация MaxMiniApp — мгновенная загрузка

## Текущее состояние

| Файл | Размер | Строк |
|------|--------|-------|
| [index.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/%D0%9F%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5/MaxMiniApp/index.html) | 8 KB | 228 |
| [calculator.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/%D0%9F%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5/MaxMiniApp/calculator.html) | 29 KB | 883 |

Проект — два standalone HTML-файла с inline CSS/JS. Хостинг на GitHub Pages.

---

## 🔍 Найденные проблемы быстродействия

### 🔴 Критические (блокируют загрузку)

| # | Проблема | Влияние |
|---|----------|---------|
| 1 | **Google Fonts загружается синхронно** — `<link href="fonts.googleapis.com/...">` блокирует рендеринг до полной загрузки шрифта | **+300-800ms** к загрузке |
| 2 | **Изображения с внешнего хостинга** (`i.ibb.co`) — DNS-резолв + TLS + скачивание с чужого CDN | **+200-500ms** на каждое изображение |
| 3 | **Нет `<link rel="preconnect">`** — браузер начинает DNS/TLS только при встрече ресурса | **+100-200ms** задержки |

### 🟡 Средние (замедляют отрисовку)

| # | Проблема | Влияние |
|---|----------|---------|
| 4 | **`backdrop-filter: blur(12px)`** на каждой карточке — очень тяжёлая GPU-операция, особенно на слабых Android | Подтормаживание скролла |
| 5 | **`background-attachment: fixed`** — принудительная перерисовка при скролле | Дёрганный скролл на мобильных |
| 6 | **Анимации с задержками до 1.3s** — контент невидим (`opacity: 0`) до секунды+ после загрузки | Страница кажется пустой |
| 7 | **Пустой `touchstart` listener** — `document.addEventListener("touchstart", function(){}, true)` — ненужная нагрузка | Незначительно |

### 🟢 Улучшения структуры и кода

| # | Проблема | Решение |
|---|----------|---------|
| 8 | Дублирование CSS между файлами | Общие стили повторяются в обоих файлах |
| 9 | `innerHTML` + шаблонные строки для рендера | Работает, но подвержено XSS |
| 10 | Нет `meta description` и `og:` тегов | Плохо для SEO и шаринга |
| 11 | Нет favicon | Нет иконки во вкладке |

---

## Предлагаемые изменения

### 1. Мгновенная загрузка шрифтов

#### [MODIFY] [index.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/Програмирование/MaxMiniApp/index.html)
#### [MODIFY] [calculator.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/Програмирование/MaxMiniApp/calculator.html)

```diff
-<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
+<link rel="preconnect" href="https://fonts.googleapis.com">
+<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
+<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap"
+      rel="stylesheet" media="print" onload="this.media='all'">
```

Шрифт загружается **асинхронно** — страница рисуется системным шрифтом, потом плавно переключается на Inter.

---

### 2. Preconnect к хостингу изображений

В оба файла перед загрузкой шрифтов:
```html
<link rel="preconnect" href="https://i.ibb.co">
```

---

### 3. Оптимизация CSS-производительности

```diff
-backdrop-filter: blur(12px);
--webkit-backdrop-filter: blur(12px);
+/* Убран backdrop-filter для лучшей производительности */
```

```diff
-background-attachment: fixed;
+/* Фон фиксирован через псевдоэлемент с will-change для GPU */
```

Заменить `background-attachment: fixed` на `::before` псевдоэлемент с `position: fixed` — GPU-ускорение без перерисовки.

---

### 4. Ускорение анимаций появления

```diff
-animation: fadeInUp 0.8s cubic-bezier(0.16, 1, 0.3, 1) forwards 1.3s;
+animation: fadeInUp 0.4s cubic-bezier(0.16, 1, 0.3, 1) forwards 0.1s;
```

- Сократить все `animation-delay` в **2-3 раза**
- Сократить `duration` с `0.8s` → `0.4-0.5s`
- Максимальная задержка: **0.5s** вместо 1.3s

---

### 5. Критический CSS + структурные улучшения

- Добавить `<meta name="description">` и `og:image`
- Убрать пустой `touchstart` listener
- Добавить `loading="lazy"` к hero-изображению в калькуляторе (оно большое, но ниже fold)

---

## Verification Plan

### Визуальная проверка в браузере
1. Открыть [index.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/%D0%9F%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5/MaxMiniApp/index.html) в браузере и убедиться:
   - Страница появляется мгновенно (без белого экрана)
   - Шрифт подгружается без рывков
   - Анимации быстрые и плавные
   - Визуальный стиль не изменился
2. Открыть [calculator.html](file:///Users/ivanreckalov/Library/Mobile%20Documents/com~apple~CloudDocs/%D0%9F%D1%80%D0%BE%D0%B3%D1%80%D0%B0%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5/MaxMiniApp/calculator.html):
   - Калькулятор работает корректно (кнопки +/−, подсчёт итога)
   - Скролл плавный (нет подтормаживаний на карточках)
   - Кнопка «Назад» работает

### Ручное тестирование пользователем
- После изменений деплой на GitHub Pages и проверка на реальном телефоне через MAX мессенджер

> [!IMPORTANT]
> Все изменения чисто оптимизационные — визуальный дизайн и функциональность не меняются.
