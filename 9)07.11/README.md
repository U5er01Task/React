Формат задач: **Lead Architecture Review**. В каждом кейсе дан фрагмент кода уровня production-сервиса, описаны аппаратные метрики сбоя (Memory Profiler, Performance Tab, Layout Metrics) и поставлена инженерная задача на перепроектирование.

---

## Блок 1. Производительность рендеринга и браузерный конвейер (Кейсы 1–3)

### Кейс 01. Принудительный сброс геометрии (Layout Thrashing / Forced Reflow)

- **Контекст:** Разработчик написал утилиту для автоматического выравнивания высоты карточек каталога под самый высокий элемент в сетке.
- **Проблемный код:**

```javascript
function alignCardHeights(cardsContainer) {
  const cards = cardsContainer.querySelectorAll(".product-card");

  cards.forEach((card) => {
    // Чтение высоты текущей карточки
    const currentHeight = card.getBoundingClientRect().height;

    // Если карточка меньше 300px, принудительно задаем новую высоту
    if (currentHeight < 300) {
      card.style.height = "300px";
    }
  });
}
```

- **Аппаратные метрики (Performance Profiler):** На списке из 200 карточек кадровая частота падает с 60 fps до 9 fps. В DevTools горит предупреждение _Forced Reflow is a likely bottleneck (Layout Thrashing)_. Вкладка замирает на 280 мс при скролле.
- **Задание:** Объясните физику работы браузерного конвейера (_Style -> Layout -> Paint_). Почему чередование чтения геометрии (`getBoundingClientRect`) и записи стилей (`style.height`) в цикле ломает оптимизацию пакетной отрисовки браузера? Напишите решение, разделяющее фазы чтения и записи (Read-Write Batching).

---

### Кейс 02. Блокировка главного потока (Long Tasks) при трансформации массива 100k элементов

- **Контекст:** Аналитический дэшборд получает с сервера сырой массив из 100 000 транзакций и выполняет сложную фильтрацию, дедупликацию и расчет агрегатов прямо в главном потоке:
- **Проблемный код:**

```javascript
function processAnalytics(rawTransactions, filterCriteria) {
  // Выполняется синхронно прямо при клике на вкладку "Аналитика"
  return rawTransactions
    .filter((t) => t.amount > filterCriteria.min && t.status === "completed")
    .map((t) => ({
      ...t,
      normalizedDate: new Date(t.timestamp).toLocaleDateString(),
      tax: t.amount * 0.2,
    }))
    .reduce(
      (acc, t) => {
        acc.totalRevenue += t.amount;
        acc.totalTax += t.tax;
        return acc;
      },
      { totalRevenue: 0, totalTax: 0 },
    );
}
```

- **Аппаратные метрики:** Метрика **INP (Interaction to Next Paint)** подскакивает до 1400 мс (красная зона Web Vitals). Пользователь кликает на чекбокс или инпут, но страница полностью «зависает» на 1.5 секунды, игнорируя любые нажатия.
- **Задание:** Объясните, почему задачи длительностью более 50 мс (_Long Tasks_) блокируют обработку очереди пользовательского ввода в Event Loop. Предложите архитектурное решение: разбиение вычислений на чанки времени через `scheduler.yield() / setTimeout` или перенос расчетов в фоновый **Web Worker**.

---

### Кейс 03. Сетевой DDoS бэкенда через `Promise.all` (Неконтролируемая конкурентность)

- **Контекст:** Пакетный загрузчик файлов: пользователь выбирает 150 фотографий для загрузки в облачное хранилище.
- **Проблемный код:**

```javascript
async function uploadAllFiles(fileList) {
  const uploadPromises = fileList.map((file) => {
    const formData = new FormData();
    formData.append("file", file);
    return fetch("/api/upload", { method: "POST", body: formData });
  });

  // Запуск 150 запросов одновременно
  return await Promise.all(uploadPromises);
}
```

- **Аппаратные метрики:** Браузер исчерпывает лимит одновременных TCP-соединений к одному домену (в HTTP/1.1 это ровно 6 соединений). В сети возникает эффект _Queueing / Stalled_. Бэкенд получает лавину одновременных соединений и отвечает ошибкой `429 Too Many Requests` или `502 Bad Gateway`.
- **Задание:** Спроектируйте утилиту **Concurrency Pool (Ограничитель параллелизма)**, которая принимает массив асинхронных задач и запускает строго не более $K$ запросов одновременно (например, Concurrency Limit = 4), подхватывая следующую задачу по мере завершения любой из активных.

---

## Блок 2. Утечки памяти (Memory Leaks) и сборщик мусора V8 (Кейсы 4–7)

### Кейс 04. Скрытая утечка памяти через глобальный кэш на объектах (Ключи без слабой связи)

- **Контекст:** Разработчик написал универсальный кэш метаданных для DOM-узлов компонентов SPA:
- **Проблемный код:**

```javascript
const nodeMetadataCache = new Map();

function attachComponentMetadata(domNode, metadata) {
  // Сохраняем в кэш ссылку на DOM-узел как ключ
  nodeMetadataCache.set(domNode, {
    ...metadata,
    attachedAt: Date.now(),
  });
}
```

- **Аппаратные метрики:** По снимкам памяти (Heap Snapshot) видно, что после удаления сотен карточек из DOM дерево отсоединенных узлов (_Detached HTMLElement_) не очищается сборщиком мусора (Garbage Collector). Объем памяти растет линейно до краша вкладки.
- **Задание:** Объясните, как работает алгоритм сборки мусора **Mark-and-Sweep** в движке V8. Почему ссылка на DOM-узел в качестве ключа стандартного `Map` удерживает весь объект в памяти? Какая структура данных стандарта ES6 решает эту проблему за счет слабых ссылок?

---

### Кейс 05. Забытый AbortSignal и утечка сетевого сокета

- **Контекст:** В долгоживущем SPA реализована подписка на серверные события (Server-Sent Events) или длинные опросы (Long Polling):
- **Проблемный код:**

```javascript
function pollServerStatus(onStatusUpdate) {
  let isAlive = true;

  async function loop() {
    while (isAlive) {
      try {
        const res = await fetch("/api/health");
        const data = await res.json();
        onStatusUpdate(data);
      } catch (err) {
        console.error("Poll error", err);
      }
      await new Promise((r) => setTimeout(r, 5000));
    }
  }

  loop();

  // Функция демонтирования, вызываемая при уходе со страницы
  return function unsubscribe() {
    isAlive = false;
  };
}
```

- **Аппаратные метрики:** Пользователь уходит со страницы в момент, когда запрос `/api/health` висит в фазе ожидания ответа (Pending). Переменная `isAlive` переводится в `false`, но сам сетевой HTTP-сокет продолжает висеть в фоновом режиме браузера до таймаута (до 60 секунд), удерживая в куче колбэк `onStatusUpdate` и все замкнутые в нем переменные страницы.
- **Задание:** Отрефакторите код с использованием `AbortController`, чтобы вызов `unsubscribe()` мгновенно обрывал сетевое соединение на транспортном уровне браузера.

---

### Кейс 06. Утечка через замыкание на тяжелом контексте (The Accidental Retention)

- **Контекст:** Функция создает обработчик клика для маленькой иконки, но захватывает в замыкание весь массив данных страницы:
- **Проблемный код:**

```javascript
function setupIconClickHandler(hugeDataPayload) {
  const iconId = hugeDataPayload.meta.iconId;

  return function handleIconClick() {
    // Нам нужен только iconId!
    console.log("Clicked icon:", iconId);
    // Но hugeDataPayload (20 МБ JSON) остается в лексическом окружении!
  };
}
```

- **Аппаратные метрики:** Heap Snapshot показывает миллионы удерживаемых объектов в скоупе `Closure`. Массив `hugeDataPayload` не освобождается, пока жив обработчик клика.
- **Задание:** Объясните, как движок V8 формирует объекты `Context` для замыканий. Напишите безопасный рефакторинг, гарантирующий, что тяжелый объект станет доступен для сборщика мусора сразу после выполнения внешней функции.

---

### Кейс 07. Зомби-слушатель глобальной шины (Event Bus Zombie)

- **Контекст:** Компонент корзины подписывается на кастомную шину событий приложения:
- **Проблемный код:**

```javascript
class CartComponent {
  constructor(eventBus) {
    this.eventBus = eventBus;
    this.items = [];

    // Привязываем метод к контексту экземпляра
    this.eventBus.on("ITEM_ADDED", this.handleItemAdded.bind(this));
  }

  handleItemAdded(item) {
    this.items.push(item);
  }

  destroy() {
    // Разработчик пытается отписаться
    this.eventBus.off("ITEM_ADDED", this.handleItemAdded.bind(this));
  }
}
```

- **Аппаратные метрики:** Компонент корзины открывается и закрывается пользователем 10 раз. При добавлении одного товара обработчик вызывается 10 раз, а все 10 старых экземпляров `CartComponent` остаются в оперативной памяти.
- **Задание:** Объясните, почему `this.fn.bind(this) !== this.fn.bind(this)`. Как некорректная ссылка на функцию делает метод `.off()` неработоспособным? Напишите 2 правильных паттерна отписки.

---

## Блок 3. Логические асинхронные аварии и стейт (Кейсы 8–11)

### Кейс 08. Устаревшее замыкание (Stale Closure) в интервалах

- **Контекст:** Таймер обратного отсчета в интерфейсе оформления заказа:
- **Проблемный код:**

```javascript
function startOrderCountdown() {
  let secondsLeft = 60;

  const timerId = setInterval(() => {
    if (secondsLeft <= 0) {
      clearInterval(timerId);
      console.log("Время истекло!");
      return;
    }

    console.log(`Осталось: ${secondsLeft} сек`);
    secondsLeft = secondsLeft - 1;
  }, 1000);
}
```

- **Симптом в React:** Если перенести эту логику в хук `useEffect` без функционального обновления или зависимостей:

```javascript
useEffect(() => {
  const id = setInterval(() => {
    // В замыкании навсегда зафиксировано secondsLeft = 60
    setSecondsLeft(secondsLeft - 1);
  }, 1000);
  return () => clearInterval(id);
}, []); // Пустой массив зависимостей
```

- **Аппаратные метрики:** На экране таймер переключается с 60 на 59 и замирает на значении 59 навсегда.
- **Задание:** Разберите механику **Stale Closure** (устаревшего замыкания) в React Fiber. Покажите решение через функциональный сеттер `setState(prev => ...)` и через паттерн `useRef`.

---

### Кейс 09. Silent Failure при проглатывании исключений в асинхронной цепочке

- **Контекст:** Процессинг оплаты заказа в интернет-магазине:
- **Проблемный код:**

```javascript
async function processOrderPayment(paymentData) {
  try {
    const token = await generatePaymentToken(paymentData);
    const result = await chargeCard(token);
    return result;
  } catch (error) {
    console.error("Payment failed", error);
    // Точка сбоя: функция ничего не возвращает явно (undefined)
  }
}

// Вызывающий код в UI:
async function onCheckoutClick() {
  const result = await processOrderPayment(data);
  // Проверка: разработчик проверяет успешность вот так:
  if (result) {
    showSuccessModal();
  } else {
    // Пусто! Ошибка не обработана
  }
}
```

- **Симптом в продакшене:** При сетевой ошибке банк отклоняет операцию, в консоль пишется лог, но пользователь видит зависшую кнопку оплаты, с которой просто пропал спиннер. Никаких сообщений об ошибке не выводится.
- **Задание:** Спроектируйте архитектурный паттерн **Result Tuple Pattern** (по аналогии с языками Go/Rust): функция возвращает `[error, data]`, заставляя вызывающий код принудительно проверять наличие ошибки.

---

### Кейс 10. Гонка мутаций общего состояния (Shared State Race Condition)

- **Контекст:** Модуль кэширования профиля пользователя с дедупликацией сетевых запросов:
- **Проблемный код:**

```javascript
let cachedUser = null;
let isFetching = false;

async function getAuthenticatedUser() {
  if (cachedUser) return cachedUser;

  if (!isFetching) {
    isFetching = true;
    const res = await fetch("/api/me");
    cachedUser = await res.json();
    isFetching = false;
    return cachedUser;
  }

  // Если запрос уже идет, ждем...
  while (isFetching) {
    await new Promise((r) => setTimeout(r, 50));
  }
  return cachedUser;
}
```

- **Симптом в продакшене:** При одновременном вызове функции из 5 разных независимых компонентов страницы (Header, Sidebar, Analytics, Chat, Cart) проверка `while(isFetching)` превращается в бесконечный опрос, либо несколько запросов проскальзывают одновременно при смене флагов.
- **Задание:** Устраните антипаттерн `while-polling`. Реализуйте паттерн **Promise Memoization (In-Flight Request Deduplication)**: кэшировать нужно не данные, а сам работающий экземпляр `Promise`.

---

### Кейс 11. Неконтролируемая рекурсия асинхронных повторов (Stack Overflow / Infinite Loop)

- **Контекст:** Функция автоматического переподключения к WebSocket или повтора сетевого запроса:
- **Проблемный код:**

```javascript
async function fetchWithInfiniteRetry(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) throw new Error("HTTP error");
    return await response.json();
  } catch (err) {
    // Рекурсивный перезапуск без пауз и лимитов
    return fetchWithInfiniteRetry(url);
  }
}
```

- **Симптом в продакшене:** Если у пользователя отключился интернет или сервер упал в 500, браузер генерирует 20 000 HTTP-запросов в секунду. Вкладка браузера мгновенно забивает сетевой стек, процессор нагружается на 100%, аккумулятор ноутбука садится за 15 минут.
- **Задание:** Реализуйте алгоритм **Exponential Backoff с фактором случайности (Jitter)** и строгим лимитом максимального числа попыток `maxRetries`.

---

## Блок 4. Иммутабельность, Нормализация и Архитектура Структур (Кейсы 12–15)

### Кейс 12. Разрушение ссылочной целостности при использовании `JSON.parse(JSON.stringify())`

- **Контекст:** Разработчик решил сделать «надежный глубокий клон» стейта, чтобы избежать мутаций:
- **Проблемный код:**

```javascript
function cloneEntireApplicationState(state) {
  return JSON.parse(JSON.stringify(state));
}

const state = {
  activeDate: new Date(),
  pattern: /^[a-z]+$/i,
  calculateTax: () => 100,
  balance: 1000n, // BigInt
  history: undefined,
};

const clonedState = cloneEntireApplicationState(state);
```

- **Симптом в продакшене:**
  1. `activeDate` превратилась из объекта `Date` в простую строку.
  2. Регулярное выражение `pattern` превратилось в пустой объект `{}`.
  3. Функция `calculateTax` и поле `history: undefined` бесследно стерлись.
  4. При попытке сериализации `BigInt` приложение выбросило фатальную ошибку `TypeError: Do not know how to serialize a BigInt`.
- **Задание:** Сформулируйте ограничения JSON-сериализации. В каких случаях применим нативный `structuredClone()`, а когда необходим строгий структурный Spread на уровне редьюсеров?

---

### Кейс 13. Каскадное уничтожение кэша мемоизации из-за генерации новых литералов

- **Контекст:** Высоконагруженная таблица на 500 строк. Каждая строка обернута в `React.memo`:
- **Проблемный код:**

```jsx
function Table({ rows }) {
  return (
    <div>
      {rows.map((row) => (
        <TableRow
          key={row.id}
          data={row}
          // Передача стилей и фильтров инлайн-литералами:
          style={{ padding: "8px", borderBottom: "1px solid #eee" }}
          options={{ canSelect: true, canEdit: false }}
          onDelete={() => console.log("Delete", row.id)}
        />
      ))}
    </div>
  );
}
```

- **Симптом в продакшене:** При изменении поля всего в одной строке все 500 строк таблицы полностью перерисовываются заново. `React.memo` не помогает.
- **Задание:** Проанализируйте работу алгоритма `shallowEqual`. Почему создание литералов объектов `{}` и стрелочных функций `() => {}` прямо в JSX ломает оптимизацию? Отрефакторите код.

---

### Кейс 14. Неправильное слияние списков через Spread при пагинации (Duplication Bug)

- **Контекст:** Бесконечная лента новостей (Infinite Scroll):
- **Проблемный код:**

```javascript
let feedState = {
  posts: [],
};

function appendNextPage(newPostsFromApi) {
  feedState = {
    ...feedState,
    posts: [...feedState.posts, ...newPostsFromApi],
  };
}
```

- **Симптом в продакшене:** Пользователь скроллит ленту. Если за время скролла на сервере появилась новая статья, пагинация со сдвигом `limit/offset` присылает статью, которая уже была в конце предыдущей страницы. В ленте появляются дубликаты, React выбрасывает ошибку: `Warning: Encountered two children with the same key`.
- **Задание:** Напишите функцию иммутабельного объединения списков со строгой дедупликацией по уникальному идентификатору `id` с сохранением производительности $O(N)$ через структуру `Set`.

---

### Кейс 15. Деструктуризация с переопределением дефолтов при `null` значениях из базы данных

- **Контекст:** В базе данных PostgreSQL/MySQL поля, не имеющие значений, хранятся строго как `NULL`.
- **Проблемный код:**

```javascript
function renderBadgeConfig({
  color = "blue",
  text = "Новинка",
  icon = "star",
}) {
  return `<span class="badge ${color}">${icon} ${text}</span>`;
}

// Ответ из реальной SQL базы данных:
const dbRow = {
  color: null,
  text: null,
  icon: null,
};

const resultHtml = renderBadgeConfig(dbRow);
```

- **Симптом в продакшене:** На экране отображается: `<span class="badge null">null null</span>`. Дефолтные значения `"blue"`, `"Новинка"` и `"star"` полностью проигнорированы.
- **Задание:** Объясните спецификацию ECMAScript относительно значений по умолчанию в деструктуризации: при каких типах данных срабатывает default value (`undefined` vs `null`). Напишите отказоустойчивую функцию нормализации объекта из базы.

---

## Подробный инженерный разбор кейса №03: Concurrency Pool (Ограничение параллелизма сетевых запросов)

### 1. Архитектурная проблема

Когда код выполняет `Promise.all(files.map(upload))`, рантайм мгновенно инициирует все сетевые вызовы.
[150 файлов] ──(Promise.all)──> 150 одновременных сетевых сокетов:
├── Сокет 1..6: [ACTIVE] (Браузер держит максимум 6 соединений к хосту)
├── Сокет 7..150: [STALLED / QUEUED] (Зависают в очереди ОС)
└── Backend: Получает ударный всплеск нагрузки ──> 429 Too Many Requests / 502 Bad Gateway

text

Если один запрос из 150 падает, `Promise.all` мгновенно прерывается по ошибке (_Fail-Fast_), бросая остальные 149 запросов выполняться неуправляемо в фоне.

### 2. Инженерное решение

Необходимо реализовать алгоритм **Пула рабочих потоков (Worker Pool / Concurrency Limiter)**:

1. Запускается строго фиксированное количество воркеров $K$ (например, $K=4$).
2. Каждый воркер берет задачу из общей очереди (Queue).
3. По завершении задачи воркер немедленно берет следующую задачу из очереди, пока очередь не опустеет.
4. Общий промис завершается только тогда, когда выполнены абсолютно все элементы.

### 3. Эталонная реализация на чистом JavaScript (ES6+)

```javascript
/**
 * Concurrency Runner: Запуск массива асинхронных задач с лимитом параллелизма
 * @param {Array<() => Promise<any>>} taskFactories - Массив функций, возвращающих промисы
 * @param {number} limit - Максимальное число одновременно выполняемых задач
 * @returns {Promise<Array<any>>} - Массив результатов в исходном порядке
 */
async function asyncPool(taskFactories, limit = 4) {
  if (!Array.isArray(taskFactories)) {
    throw new TypeError("taskFactories must be an array of functions");
  }

  // Массив для хранения финальных результатов в строгом порядке индексов
  const results = new Array(taskFactories.length);

  // Указатель на следующий индекс задачи в очереди
  let currentIndex = 0;

  // Функция-воркер: крутится в цикле, пока есть задачи в очереди
  async function worker() {
    while (currentIndex < taskFactories.length) {
      // Атомарно забираем текущий индекс и сдвигаем указатель вперед
      const taskIndex = currentIndex++;
      const taskFn = taskFactories[taskIndex];

      try {
        // Дожидаемся выполнения конкретной задачи
        const result = await taskFn();
        results[taskIndex] = { status: "fulfilled", value: result };
      } catch (error) {
        results[taskIndex] = { status: "rejected", reason: error };
      }
    }
  }

  // Запускаем пул из K независимых параллельных воркеров
  const workerCount = Math.min(limit, taskFactories.length);
  const workers = [];

  for (let i = 0; i < workerCount; i++) {
    workers.push(worker());
  }

  // Дожидаемся, пока ВСЕ воркеры исчерпают очередь
  await Promise.all(workers);

  return results;
}
```

### 4. Верификация и проверка работы под нагрузкой

```javascript
// Эмуляция сетевой задержки от 100 до 500 мс
const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

// Генерируем 12 тяжелых задач
let activeRequestsCount = 0;
let peakRequestsCount = 0;

const mockTasks = Array.from({ length: 12 }, (_, i) => async () => {
  activeRequestsCount++;
  peakRequestsCount = Math.max(peakRequestsCount, activeRequestsCount);

  console.log(
    `[START] Задача #${i + 1} | Сейчас активно: ${activeRequestsCount}`,
  );

  // Эмулируем сетевой ответ
  await sleep(200);

  activeRequestsCount--;
  console.log(
    `[DONE]  Задача #${i + 1} | Сейчас активно: ${activeRequestsCount}`,
  );

  return `Result payload #${i + 1}`;
});

// Запускаем с лимитом ровно 3 задачи одновременно:
console.log("=== СТАРТ ПУЛА С КОНКУРЕНТНОСТЬЮ = 3 ===");

asyncPool(mockTasks, 3).then((finalResults) => {
  console.log("=== ВСЕ ЗАДАЧИ ЗАВЕРШЕНЫ ===");
  console.log("Пиковое количество одновременных сокетов:", peakRequestsCount);
  // Строго 3! Лимит ни разу не был превышен.

  console.log(
    "Результаты сохранены в исходном порядке:",
    finalResults.length === 12,
  );
});
```
