Формат задач: **Анализ Pull Request**. В каждом задании содержится фрагмент кода с неочевидной инженерной ошибкой, описание сбоя в продакшене и техническое задание на исправление.

---

## Блок 1. Архитектура SPA и роутинг (Кейсы 1–3)

### Кейс 01. Зацикливание истории браузера при программном редиректе

- **Контекст:** Разработчик реализовал защиту приватного роута: если пользователь не авторизован, его перенаправляет на страницу логина.
- **Проблемный код:**

```javascript
function checkAuthAndNavigate(isAuthenticated, currentPath) {
  if (!isAuthenticated && currentPath !== "/login") {
    window.history.pushState({}, "", "/login");
    renderLoginPage();
  }
}
```

- **Симптом в продакшене:** Пользователь нажимает кнопку «Назад» в браузере, но страница не возвращается назад — URL на секунду меняется и мгновенно возвращается на `/login`. Кнопка «Назад» оказывается заблокированной.
- **Задание:** Объясните разницу между мутациями стека истории через `pushState` и `replaceState`. Исправьте код так, чтобы редирект не засорял историю сессии.

---

### Кейс 02. Утечка памяти в долгоживущей вкладке SPA

- **Контекст:** В SPA реализована шапка с отображением ширины экрана для адаптивного меню.
- **Проблемный код:**

```javascript
function mountHeader() {
  const header = document.getElementById("header");

  window.addEventListener("resize", () => {
    const isMobile = window.innerWidth < 768;
    header.className = isMobile ? "header-mobile" : "header-desktop";
  });

  return header;
}
```

- **Симптом в продакшене:** После нескольких часов активного серфинга по вкладкам SPA без перезагрузки браузер начинает потреблять более 1.5 ГБ RAM, а интерфейс начинает тормозить при изменении размера окна.
- **Задание:** Найдите причину утечки памяти (Memory Leak) в замыкании Event Listener. Напишите функцию жизненного цикла с методом очистки (`cleanup / unmount`).

---

### Кейс 03. Hard Reload вместо SPA перехода

- **Контекст:** Реализован компонент карточки товара с кнопкой перехода в корзину.
- **Проблемный код:**

```javascript
function ProductCard({ product, router }) {
  return `
    <div class="card">
      <h3>${product.title}</h3>
      <a href="/cart" onclick="router.navigate('/cart')">В корзину</a>
    </div>
  `;
}
```

- **Симптом в продакшене:** При клике на ссылку страница на доли секунды монтирует корзину, после чего происходит белый экран и полная перезагрузка страницы браузером (сброс всего стейта приложения).
- **Задание:** Объясните физику поведения браузера при обработке событий ссылок. Исправьте обработчик.

---

## Блок 2. Окружение, зависимости и бандлинг (Кейсы 4–6)

### Кейс 04. Поломка продакшен-сборки из-за devDependencies

- **Контекст:** Разработчик подключил библиотеку для форматирования дат `date-fns` и утилиту сборки `vite`.
- **Фрагмент `package.json`:**

```json
{
  "dependencies": {
    "vite": "^5.4.0"
  },
  "devDependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "date-fns": "^3.6.0"
  }
}
```

- **Симптом в продакшене:** Локально команда `npm run dev` работает штатно, но на CI/CD сервере команда сборки `npm run build` падает с ошибкой или собирает контейнер огромного размера.
- **Задание:** Найдите логическую ошибку в классификации зависимостей. Перепишите секции `dependencies` и `devDependencies` в соответствии со стандартами продакшена.

---

### Кейс 05. Разрушение детерминизма через `npm install` на сервере

- **Контекст:** В пайплайне деплоя на хостинг настроен скрипт сборки:

```bash
git pull origin main
npm install
npm run build
```

- **Симптом в продакшене:** В пятницу вечером проект собирался корректно, а в понедельник утром автоматический билд упал с ошибкой сборки, хотя в репозиторий не было отправлено ни единого коммита.
- **Задание:** Объясните, как оператор каретки `^` в сочетании с `npm install` ломает детерминизм сборки. Какую команду пакетного менеджера необходимо использовать на сервере?

---

### Кейс 06. Импорт серверных Node.js модулей в клиентский код Vite

- **Контекст:** Разработчик попытался прочитать версию приложения из `package.json` внутри React-компонента:

```javascript
// src/components/Footer.jsx
import fs from "node:fs";
import path from "node:path";

export function Footer() {
  const pkgPath = path.resolve("package.json");
  const pkg = JSON.parse(fs.readFileSync(pkgPath, "utf-8"));
  return `<footer>Версия: ${pkg.version}</footer>`;
}
```

- **Симптом в продакшене:** Vite выбрасывает критическую ошибку в консоли браузера: `Module "fs" has been externalized for browser compatibility. Cannot access "fs.readFileSync" in client code`.
- **Задание:** Объясните фундаментальное различие сред исполнения (Node.js runtime vs Browser Sandbox). Как корректно прокинуть метаданные проекта в клиентский код через `vite.config.js`?

---

## Блок 3. Модель памяти, ссылки и мутации (Кейсы 7–10)

### Кейс 07. Мутация данных в общем кэше через присвоение ссылки

- **Контекст:** Функция фильтрует массив пользователей по статусу активности и сбрасывает им счетчик попыток входа:
- **Проблемный код:**

```javascript
function resetAttemptsForActiveUsers(usersList) {
  const activeUsers = usersList.filter((user) => user.isActive);

  for (let user of activeUsers) {
    user.loginAttempts = 0; // Сброс
  }

  return activeUsers;
}
```

- **Симптом в продакшене:** После вызова этой функции пользователи в _исходном_ списке `usersList` тоже обнулили свои счетчики, хотя функция должна была вернуть изолированный результат.
- **Задание:** Объясните, как работает передача ссылочных типов в куче (Heap). Перепишите функцию так, чтобы она была абсолютно чистой (Pure Function) и не мутировала исходный массив и его объекты.

---

### Кейс 08. Ловушка `Object.assign` и Spread на вложенных структурах

- **Контекст:** Разработчик клонирует профиль пользователя с помощью Spread-оператора:
- **Проблемный код:**

```javascript
const userProfile = {
  id: 101,
  name: "Сергей",
  security: {
    roles: ["admin", "editor"],
    twoFactor: true,
  },
};

const userCopy = { ...userProfile };
userCopy.security.roles.push("superadmin");
```

- **Симптом в продакшене:** В `userProfile.security.roles` добавилась роль `superadmin`. Нарушена безопасность сессии.
- **Задание:** Объясните физику Shallow Copy (поверхностного копирования). Напишите функцию глубокого клонирования или безопасного иммутабельного обновления вложенного массива.

---

### Кейс 09. Ловушка вызова стрелочных функций в циклах (Замыкание `var`)

- **Контекст:** Создается массив колбэков для динамических кнопок пагинатора:
- **Проблемный код:**

```javascript
function createPaginationButtons(pagesCount) {
  var buttons = [];

  for (var i = 1; i <= pagesCount; i++) {
    buttons.push(() => {
      console.log(`Переход на страницу: ${i}`);
    });
  }

  return buttons;
}

const pageHandlers = createPaginationButtons(5);
pageHandlers[0](); // Ожидалось: 1
pageHandlers[1](); // Ожидалось: 2
```

- **Симптом в продакшене:** Все кнопки без исключения выводят: `Переход на страницу: 6`.
- **Задание:** Объясните поведение лексического окружения, область видимости `var` vs `let` и замыкание переменной цикла. Исправьте баг двумя разными способами.

---

### Кейс 10. Потеря контекста при передаче методов объекта в Event Handler

- **Контекст:** Сервис аналитики отслеживает клики по элементам интерфейса:
- **Проблемный код:**

```javascript
const analyticsTracker = {
  prefix: "ANALYTICS_EVENT",
  sendEvent(eventName) {
    console.log(`[${this.prefix}]: ${eventName}`);
  },
};

document
  .getElementById("checkout-btn")
  .addEventListener("click", analyticsTracker.sendEvent);
```

- **Симптом в продакшене:** В лог падает: `[undefined]: [object MouseEvent]`.
- **Задание:** Объясните, как работает динамический контекст `this` при потере вызова через точку. Исправьте привязку контекста (покажите 3 разных варианта решения).

---

## Блок 4. Методы массивов: map, filter, find, reduce (Кейсы 11–14)

### Кейс 11. Мутирующий метод `.sort()` внутри пайплайна

- **Контекст:** Функция отображает топ-3 самых дорогих товаров каталога:
- **Проблемный код:**

```javascript
function getTopExpensiveProducts(products) {
  return products.sort((a, b) => b.price - a.price).slice(0, 3);
}
```

- **Симптом в продакшене:** После каждого вызова этой функции общий каталог товаров на главной странице спонтанно меняет свой порядок и перемешивается.
- **Задание:** Какие методы массивов мутируют массив in-place, а какие являются чистыми? Перепишите код безопасно для использования в React-приложении.

---

### Кейс 12. Некорректный выбор метода: `.forEach()` вместо `.map()` с мутацией

- **Контекст:** Разработчик преобразует массив строк с ID в массив объектов задач:
- **Проблемный код:**

```javascript
function mapIdsToTasks(ids) {
  let result = [];
  ids.forEach((id, index) => {
    result[index] = { taskId: id, isDone: false };
  });
  return result;
}
```

- **Симптом в продакшене:** Избыточный императивный код, создание внешней переменной и ручное индексирование.
- **Задание:** Проведите рефакторинг в одну строку через декларативный `.map()`. Объясните, почему в React функциональный стиль предпочтительнее императивных циклов.

---

### Кейс 13. Логическая ошибка при поиске falsy-значений через `.find()`

- **Контекст:** Поиск статуса выполнения этапа по списку кодов завершения:
- **Проблемный код:**

```javascript
const exitCodes =;[0][5][8][12]

const targetCode = exitCodes.find(code => code === 0);

if (!targetCode) {
  console.log("Код 0 не найден в списке!");
} else {
  console.log("Код успешно найден:", targetCode);
}
```

- **Симптом в продакшене:** В консоль всегда выводится сообщение `"Код 0 не найден в списке!"`, несмотря на то, что ноль присутствует в массиве.
- **Задание:** Объясните ловушку проверки истинности (truthy/falsy) при возврате числа `0` методом `.find()`. Напишите правильную проверку наличия элемента.

---

### Кейс 14. Неинициализированный аккумулятор в `.reduce()`

- **Контекст:** Функция подсчитывает общую сумму заказов в корзине:
- **Проблемный код:**

```javascript
const cartItems = [
  { id: 1, title: "Мышь", price: 2000 },
  { id: 2, title: "Коврик", price: 1000 },
];

const totalCost = cartItems.reduce((acc, item) => acc + item.price);
```

- **Симптом в продакшене:** Вместо числа `3000` в `totalCost` оказывается строка `"[object Object]1000"`.
- **Задание:** Объясните механику выбора первого значения аккумулятора в методе `.reduce()`, если `initialValue` не передан. Исправьте ошибку.

---

## Блок 5. Деструктуризация, Rest и вычисляемые свойства (Кейсы 15–17)

### Кейс 15. Баг динамического обновления формы с несколькими полями

- **Контекст:** Обработчик формы профиля:
- **Проблемный код:**

```javascript
let formState = {
  name: "Иван",
  email: "ivan@test.com",
};

function onFieldChange(fieldName, value) {
  formState = {
    ...formState,
    fieldName: value,
  };
}

onFieldChange("email", "new-email@test.com");
```

- **Симптом в продакшене:** `formState.email` не изменился, но в объекте появилось новое поле `{ fieldName: "new-email@test.com" }`.
- **Задание:** Объясните синтаксис статических ключей объекта и вычисляемых свойств (Computed Property Names). Исправьте функцию.

---

### Кейс 16. Падение при деструктуризации `null` в ответах API

- **Контекст:** Получение данных пользователя с бэкенда:
- **Проблемный код:**

```javascript
function renderUserStats(serverData) {
  // serverData может прийти как null при сбое
  const { stats = { views: 0, likes: 0 } } = serverData;
  return `Просмотры: ${stats.views}`;
}

renderUserStats(null);
```

- **Симптом в продакшене:** Фатальная ошибка: `Uncaught TypeError: Cannot destructure property 'stats' of 'serverData' as it is null.`
- **Задание:** Объясните, почему значение по умолчанию не сработало для `null`. Перепишите деструктуризацию для абсолютно безопасной работы.

---

### Кейс 17. Случайное стирание пропсов из-за неправильного порядка Spread

- **Контекст:** Универсальный компонент инпута с дефолтным типом:
- **Проблемный код:**

```javascript
function TextInput(props) {
  const mergedProps = {
    ...props,
    type: "text",
    className: "form-input",
  };

  return mergedProps;
}

// Вызов разработчиком:
const passwordConfig = TextInput({ type: "password", id: "pass" });
```

- **Симптом в продакшене:** Инпут пароля отображает вводимые символы открытым текстом (`type="text"`).
- **Задание:** Объясните правило переопределения ключей в объектах (Key Overriding). Исправьте компонент так, чтобы внешние пропсы могли переопределять дефолтные, но `className` объединялись.

---

## Блок 6. Асинхронный JS, Promise и Race Conditions (Кейсы 18–20)

### Кейс 18. «Проглоченная» ошибка внутри `new Promise()`

- **Контекст:** Обертка над асинхронным событием:
- **Проблемный код:**

```javascript
function loadCustomScript(src) {
  return new Promise((resolve, reject) => {
    const script = document.createElement("script");
    script.src = src;

    script.onload = () => {
      // Имитация скрытой ошибки парсинга
      const data = JSON.parse("НЕВАЛИДНЫЙ_JSON");
      resolve(data);
    };

    document.head.appendChild(script);
  });
}
```

- **Симптом в продакшене:** В консоли падает `Uncaught SyntaxError`, но возвращенный промис навсегда зависает в состоянии `pending`. Блок `.catch()` внешнего кода никогда не вызывается, спиннер загрузки крутится бесконечно.
- **Задание:** Объясните, почему ошибки внутри асинхронных колбэков не ловятся конструктором `new Promise`. Исправьте обработчик.

---

### Кейс 19. Race Condition при быстром вводе в строку поиска (Search Autocomplete)

- **Контекст:** Пользователь вводит текст в поле живого поиска. На каждый символ отправляется сетевой запрос:
- **Проблемный код:**

```javascript
let currentSearchResult = [];

async function onSearchInput(query) {
  const response = await fetch(`/api/search?q=${query}`);
  const data = await response.json();
  currentSearchResult = data;
  renderDropdown(currentSearchResult);
}
```

- **Симптом в продакшене:** Пользователь ввел `"react"`, а затем быстро стер и ввел `"vue"`. Первым вернулся ответ по `"vue"`, а через секунду пришел задержавшийся медленный ответ по `"react"` и перезаписал экран. В инпуте написано `"vue"`, а в выпадающем списке лежат результаты для `"react"`.
- **Задание:** Объясните природу состояния гонки (Race Condition). Перепишите сервис с использованием стандарта `AbortController`.

---

### Кейс 20. Падение всего `Promise.all` из-за одного второстепенного виджета

- **Контекст:** Загрузка страницы профиля: данные пользователя, баланс и второстепенный рекламный баннер:
- **Проблемный код:**

```javascript
async function loadFullProfilePage(userId) {
  const [user, balance, promo] = await Promise.all([
    fetchUserData(userId),
    fetchUserBalance(userId),
    fetchPromoBanner(), // Сторонний рекламный сервис, часто дает 500
  ]);

  renderProfile({ user, balance, promo });
}
```

- **Симптом в продакшене:** Если сторонний рекламный сервер `fetchPromoBanner()` падает с ошибкой, страница профиля не отображается вовсе — весь `Promise.all` аварийно падает, лишая пользователя доступа к балансу и профилю.
- **Задание:** Объясните поведение Fail-Fast в `Promise.all`. Отрефакторите код с помощью `Promise.allSettled`, чтобы ошибки второстепенных виджетов не ломали критические данные.

---

## Подробный инженерный разбор кейса №19: Race Condition в поиске

### 1. Архитектурный анализ проблемы

Сетевые запросы асинхронны и не гарантируют соблюдения порядка возврата ответов (FIFO — First In, First Out).
Ввод "rea" ──(HTTP Запрос 1: Задержка 1200мс)─────────────> [Ответ 1 пришел ПОЗЖЕ!]
Ввод "react" ──(HTTP Запрос 2: Быстрый кэш 200мс) ──> [Ответ 2]

text

Если не управлять отменой предыдущих сетевых вызовов:

1. Запрос №1 был отправлен в $t=0$, но сервер ответил через $1.2$ секунды.
2. Запрос №2 был отправлен в $t=200$, но сервер ответил мгновенно через $0.2$ секунды ($t=400$).
3. В момент $t=400$ на экране отобразились актуальные данные запроса №2.
4. В момент $t=1200$ приходит задержавшийся ответ запроса №1 и затирает актуальный экран устаревшими данными.

### 2. Эталонное решение через AbortController

Для предотвращения гонки запросов применяется браузерный API **`AbortController`**. Перед каждым новым запросом предыдущий активный запрос физически отменяется на уровне сетевого стека браузера.

```javascript
/**
 * Фабрика безопасного сервиса поиска с отменой устаревших запросов
 */
function createSearchService() {
  // Храним ссылку на текущий активный контроллер отмены в замыкании
  let activeAbortController = null;

  return async function search(query) {
    // 1. Если есть активный незавершенный запрос — отменяем его
    if (activeAbortController) {
      activeAbortController.abort();
    }

    // 2. Создаем новый экземпляр контроллера для текущего вызова
    activeAbortController = new AbortController();
    const { signal } = activeAbortController;

    try {
      const response = await fetch(
        `/api/search?q=${encodeURIComponent(query)}`,
        {
          signal, // Передаем сигнал отмены в fetch
        },
      );

      if (!response.ok) {
        throw new Error(`HTTP Error: ${response.status}`);
      }

      const results = await response.json();

      // Запрос успешно завершился — сбрасываем ссылку
      activeAbortController = null;
      return results;
    } catch (error) {
      // 3. Отмена запроса выбрасывает исключение AbortError
      // Это штатное поведение, такую ошибку НЕЛЬЗЯ выводить пользователю как сбой
      if (error.name === "AbortError") {
        console.log(
          `[Search Aborted]: Запрос для "${query}" отменен в пользу более нового`,
        );
        return null; // Или возвращаем специальный маркер отмены
      }

      // Все остальные ошибки (сеть, 500) являются реальными сбоями
      activeAbortController = null;
      throw error;
    }
  };
}
```

### 3. Интеграция с React-компонентом через useEffect

В React этот же паттерн реализуется через функцию очистки (Cleanup function):

```jsx
import { useState, useEffect } from "react";

function UserSearchDropdown() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [isLoading, setIsLoading] = useState(false);

  useEffect(() => {
    // Если строка поиска пуста — очищаем список без отправки запроса
    if (!query.trim()) {
      setResults([]);
      setIsLoading(false);
      return;
    }

    const controller = new AbortController();

    async function executeSearch() {
      setIsLoading(true);
      try {
        const res = await fetch(
          `/api/users?search=${encodeURIComponent(query)}`,
          {
            signal: controller.signal,
          },
        );

        if (!res.ok) throw new Error("Search failed");
        const data = await res.json();

        setResults(data);
      } catch (err) {
        if (err.name !== "AbortError") {
          console.error("Ошибка поиска:", err);
        }
      } finally {
        // Выключаем индикатор только если данный запрос не был прерван
        if (!controller.signal.aborted) {
          setIsLoading(false);
        }
      }
    }

    executeSearch();

    // Cleanup: срабатывает при каждом новом вводе символа или размонтировании
    return () => {
      controller.abort();
    };
  }, [query]);

  return (
    <div className="search-container">
      <input
        type="text"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Поиск пользователей..."
      />
      {isLoading && <span className="spinner">Загрузка...</span>}
      <ul className="results-list">
        {results.map((user) => (
          <li key={user.id}>{user.name}</li>
        ))}
      </ul>
    </div>
  );
}
```
