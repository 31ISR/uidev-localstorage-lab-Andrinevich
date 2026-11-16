# To-Do List с сохранением в `localStorage`

## Срок сдачи работ

Последний коммит и пул реквест должен быть оформлен до ???

## Цель

Разметка и стили To-Do приложения уже готовы. Вам нужно написать **свой JavaScript**, чтобы задачи можно было добавлять, отмечать выполненными и удалять, а главное — **сохранять их в `localStorage`**, чтобы после перезагрузки страницы список не пропадал.

---

## Шаг 0. Подготовка

1. Создайте файл `script.js` рядом с `index.html`.
2. Подключите его в конце `<body>`, чтобы к моменту запуска весь HTML уже существовал.
3. Проверьте, что всё работает:

```js
console.log('Скрипт подключен')
```

---

## Шаг 1. Разберитесь, какие элементы есть в HTML

Откройте `index.html` и найдите id-шники, с которыми вы будете работать:

- `#todo-form` — форма добавления задачи;
- `#todo-input` — поле ввода текста задачи;
- `#todo-list` — пустой `<ul>`, куда JS будет вставлять `<li class="todo">…</li>`;
- `#empty-state` — заглушка «список пуст», показывается, когда задач нет;
- `#counter` — счётчик «Всего / Выполнено»;
- `#clear-btn` — кнопка «Очистить всё».

_В верстке вам даны все элементы сразу. Когда сделаете отрисовку через JavaScript - удалите его из верстки в `index.html`_

- `data-id` — идентификатор задачи;
- класс `todo`;
- вложенные `.todo__checkbox`, `.todo__text`, `.todo__delete`;
- класс-модификатор `todo--done`, если задача выполнена.

---

## Шаг 2. Теория: как устроен `localStorage`

**Web Storage API** — это встроенный в браузер механизм хранения пар «ключ-значение» на стороне пользователя. Есть два хранилища:

|                  | `localStorage`              | `sessionStorage`       |
| ---------------- | --------------------------- | ---------------------- |
| Время жизни      | Пока пользователь не удалит | Пока открыта вкладка   |
| Объём            | ~5–10 МБ                    | ~5 МБ                  |
| Видимость        | Все вкладки одного origin   | Только текущая вкладка |
| Участвует в HTTP | ❌                          | ❌                     |

Нам нужен `localStorage`, потому что задачи должны «переживать» перезагрузку страницы.

**Чем отличается от cookies:** cookies отправляются на сервер при каждом запросе, у них лимит ~4 КБ и сложный API. `localStorage` — чисто клиентский, и именно поэтому он удобен для хранения пользовательских данных.

### Основные методы

```js
localStorage.setItem('key', 'value') // сохранить
localStorage.getItem('key') // получить (или null)
localStorage.removeItem('key') // удалить
localStorage.clear() // очистить всё
```

### Главное ограничение — только строки

`localStorage` **не умеет хранить массивы и объекты**. Если попытаться:

```js
localStorage.setItem('todos', [{ id: 1, text: 'Учиться' }])
// сохранится строка "[object Object]" — данные потеряны ❌
```

Поэтому любое сохранение проходит через **сериализацию в JSON**:

```js
// Записать
localStorage.setItem('todos', JSON.stringify(todos))

// Прочитать
const todos = JSON.parse(localStorage.getItem('todos') ?? '[]')
```

Каждая задача — объект:

```js
{
  id: 1717662400000,
  text: "Сделать лабораторную",
  done: false,
  createdAt: "2025-06-06T10:00:00.000Z"
}
```

Весь список — **массив** таких объектов. Он и будет лежать в `localStorage` под ключом `todos`.

### Ловушки `localStorage`

1. **Синхронность.** Операции блокируют главный поток — не храните туда мегабайты.
2. **Приватный режим и квоты.** `setItem` может бросить исключение, если хранилище заполнено или отключено. Оборачивайте в `try/catch`.
3. **Битые данные.** Пользователь может испортить строку через DevTools — `JSON.parse` упадёт. Тоже `try/catch`.
4. **Origin.** Данные видны только страницам того же origin (протокол + домен + порт).

---

## Шаг 3. Константы и ссылки на DOM

В начале `script.js` заведите константы, чтобы не «плодить» магические строки и `querySelector` по всему файлу:

```js
const STORAGE_KEY = 'todos'

const form = document.getElementById('todo-form')
const input = document.getElementById('todo-input')
const list = document.getElementById('todo-list')
const emptyState = document.getElementById('empty-state')
const counter = document.getElementById('counter')
const clearBtn = document.getElementById('clear-btn')

let todos = [] // текущее состояние в памяти
```

`todos` — единственный источник правды. Все функции работают с этим массивом, а `localStorage` и DOM — лишь его отражения.

---

## Шаг 4. Чтение из `localStorage` — пример

```js
function loadTodos() {
    try {
        const raw = localStorage.getItem(STORAGE_KEY)
        if (!raw) {
            todos = []
            return
        }
        const parsed = JSON.parse(raw)
        todos = Array.isArray(parsed) ? parsed : []
    } catch (err) {
        console.error('Не удалось прочитать задачи из localStorage:', err)
        todos = []
    }
}
```

**Что здесь важно:**

- `try/catch` — потому что и `getItem`, и `JSON.parse` могут бросить исключение (заблокированное хранилище, битая строка).
- `if (!raw)` — если ключа нет, `getItem` вернёт `null`, и `JSON.parse(null)` даст `null` (а не пустой массив). Обрабатываем явно.
- `Array.isArray(parsed)` — защита от того, что пользователь записал в ключ не массив, а число или объект.

---

## Шаг 5. Запись в `localStorage` — сделайте сами

Функция `saveTodos()` делает обратную операцию: берёт текущий массив `todos`, превращает его в строку через `JSON.stringify` и записывает через `localStorage.setItem` под тем же ключом `STORAGE_KEY`.

**Ваша задача:** напишите `saveTodos()` по аналогии с `loadTodos()`:

- объявите функцию без аргументов;
- внутри — `try/catch`;
- в `try` одна строка: `localStorage.setItem(STORAGE_KEY, JSON.stringify(todos))`;
- в `catch` — `console.error` с понятным сообщением.

---

Шаг 6. Рендер списка

Функция render() — единственное место, где меняется DOM списка. Она полностью перерисовывает <ul> из текущего массива todos.
```js
function render() {
  list.innerHTML = "";

  todos.forEach((todo) => {
    const li = document.createElement("li");
    li.className = "todo" + (todo.done ? " todo--done" : "");
    li.dataset.id = todo.id;

    const checkbox = document.createElement("input");
    checkbox.type = "checkbox";
    checkbox.className = "todo__checkbox";
    checkbox.checked = todo.done;

    const text = document.createElement("span");
    text.className = "todo__text";
    text.textContent = todo.text; 

    const del = document.createElement("button");
    del.type = "button";
    del.className = "todo__delete";
    del.textContent = "✕";
    del.setAttribute("aria-label", "Удалить задачу");

    li.append(checkbox, text, del);
    list.append(li);
  });

  emptyState.hidden = todos.length > 0;

  const doneCount = todos.filter((t) => t.done).length;
  counter.textContent = `Всего: ${todos.length} | Выполнено: ${doneCount}`;
}
```

---

## Шаг 7. Действия над задачами

### Пример: `addTodo(text)`

```js
function addTodo(text) {
    todos.push({
        id: Date.now(),
        text: text.trim(),
        done: false,
        createdAt: new Date().toISOString(),
    })
    saveTodos()
    render()
}
```

Любое изменение массива `todos` сопровождается `saveTodos()` → затем `render()`. Сначала «зафиксировали» изменение в хранилище, потом перерисовали интерфейс. Не наоборот.

### `toggleTodo(id)` — сделайте сами

Функция получает `id`, находит в `todos` нужный объект через [`find`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find) (сравнение по полю `id`), и если нашла — инвертирует у него `done`. Затем `saveTodos()` + `render()`.

Подсказки:

- `const todo = todos.find(t => t.id === id);`
- если `todo` не нашёлся (например, элемент уже удалили), сделайте `return`, чтобы не упасть с `Cannot read properties of undefined`;
- `todo.done = !todo.done;` — инверсия.

### `deleteTodo(id)` — сделайте сами

Функция получает `id`, «выбрасывает» из массива элемент с этим `id` через [filter](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter) и сохраняет результат обратно в `todos`. Затем `saveTodos()` + `render()`.

Подсказки:

- удобнее всего через `todos = todos.filter(t => t.id !== id);` — `filter` возвращает **новый** массив, поэтому присваиваем его в `todos`;

---

## Шаг 8. События — один пример, остальное по аналогии

### Пример: отправка формы

```js
form.addEventListener('submit', (e) => {
    e.preventDefault()
    const text = input.value.trim()
    if (!text) return
    addTodo(text)
    input.value = ''
    input.focus()
})
```

**Что здесь важно:**

- `e.preventDefault()` — иначе браузер попытается «отправить форму» и перезагрузит страницу. Задачи «моргнут» и вернутся из `localStorage`.
- `.trim()` и `if (!text) return` — защита от пустых задач и строк из одних пробелов.
- `input.focus()` после добавления — чтобы можно было сразу печатать следующую задачу.

### Список

Кнопки «✕» и чекбоксы создаются динамически в `render()` — если вешать обработчики на каждую, придётся делать это заново после каждой перерисовки. Вместо этого повесьте **один** `click`-обработчик на `<ul id="todo-list">` и внутри разберитесь, куда именно кликнули.

```JS
list.addEventListener("click", (e) => {
  const li = e.target.closest(".todo");
  if (!li) return;
  const id = Number(li.dataset.id);

  if (e.target.classList.contains("todo__checkbox")) {
    toggleTodo(id);
  } else if (e.target.classList.contains("todo__delete")) {
    deleteTodo(id);
  }
});
```

### Кнопка «Очистить всё» — сделайте сами

Повесьте `click` на `clearBtn`. Внутри:

1. Если `todos.length === 0` — ничего не делать (`return`).
2. Спросить подтверждение через `confirm("Удалить все задачи?")` — если пользователь нажал «Отмена», `return`.
3. Обнулить `todos = []`.
4. `saveTodos()` + `render()`.

---

## Шаг 9. Инициализация

В самом конце файла:

```js
loadTodos()
render()
```

Скрипт подключён в конце `<body>`, поэтому DOM уже готов. Если бы он был в `<head>`, пришлось бы оборачивать инициализацию в `DOMContentLoaded`.

---

## Шаг 10. Проверка

**Как посмотреть данные в браузере:** DevTools → **Application → Storage → Local Storage → ваш origin**. Там будет ключ `todos` со строкой JSON. Отредактируйте её вручную и обновите страницу — приложение перерисуется по новым данным.

## Как сдавать

1. Создайте форк репозитория в организации `31ISR` с названием репо.
2. Используя ветку `wip` сделайте задание
3. Зафиксируйте изменения в вашем репозитории
4. Когда документ будет готов - создайте пул реквест из ветки `wip` (вашей) на ветку `main` (тоже вашу) и укажите меня ([ktkv419](https://github.com/ktkv419)) как reviewer
5. Захостите работу на Pages

**Не мержите сами коммит**, это сделаю я после проверки задания
