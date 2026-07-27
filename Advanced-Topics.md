# Advanced Topics — Front-End Engineer Interview Q&A

> Остання актуалізація: 2026-06-15

---

## Зміст
1. [JWT Tokens](#jwt-tokens)
2. [Методи автентифікації та авторизації](#методи-автентифікації-та-авторизації)
3. [BE як ініціатор — Push-архітектури](#be-як-ініціатор--push-архітектури)
4. [Відображення та агрегація даних з BE](#відображення-та-агрегація-даних-з-be)
5. [Робота з великими даними з BE](#робота-з-великими-даними-з-be)

---

## JWT Tokens

**Q: Що таке JWT і з яких частин він складається?**

A: JWT (JSON Web Token) — компактний, самодостатній спосіб передачі інформації між сторонами у вигляді JSON-об'єкта, підписаного цифровим підписом.

Складається з трьох частин, розділених крапкою:
```
header.payload.signature
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMifQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

- **Header** — алгоритм підпису та тип токена:
  ```json
  { "alg": "HS256", "typ": "JWT" }
  ```
- **Payload** — claims (твердження): дані користувача та метадані:
  ```json
  { "sub": "userId123", "role": "admin", "exp": 1718000000, "iat": 1717996400 }
  ```
- **Signature** — `HMACSHA256(base64(header) + "." + base64(payload), secret)`

---

**Q: Які стандартні claims є в Payload?**

A:
| Claim | Опис |
|---|---|
| `sub` | Subject — ідентифікатор користувача |
| `iss` | Issuer — хто видав токен |
| `aud` | Audience — для кого призначений |
| `exp` | Expiration time — коли закінчується |
| `iat` | Issued at — коли виданий |
| `nbf` | Not before — не дійсний до цього часу |
| `jti` | JWT ID — унікальний ідентифікатор токена |

---

**Q: Як працює JWT-аутентифікація (flow)?**

A:
```
1. Клієнт надсилає credentials (login + password) → POST /auth/login
2. Сервер перевіряє credentials
3. Сервер генерує Access Token (короткоживучий, ~15хв) + Refresh Token (~7 днів)
4. Клієнт зберігає токени та надсилає Access Token в кожному запиті:
   Authorization: Bearer <access_token>
5. Сервер валідує підпис токена (без звернення до БД)
6. Коли Access Token протухає → клієнт надсилає Refresh Token → отримує новий Access Token
7. Коли Refresh Token протухає → логаут, потрібна повторна аутентифікація
```

---

**Q: Де зберігати JWT на клієнті і які ризики?**

A:
| Місце | Переваги | Ризики |
|---|---|---|
| `localStorage` | Простота, доступний між вкладками | Вразливий до **XSS** — будь-який JS може прочитати |
| `sessionStorage` | Очищається при закритті вкладки | Також вразливий до XSS |
| `HttpOnly Cookie` | **Недоступний для JS** → захищений від XSS | Вразливий до **CSRF** (вирішується CSRF-токеном або `SameSite=Strict`) |

Рекомендований підхід: **HttpOnly Cookie** для Refresh Token + `Authorization: Bearer` header для Access Token (зберігається в пам'яті).

---

**Q: Чим JWT відрізняється від Session-based автентифікації?**

A:
| | JWT (Stateless) | Session (Stateful) |
|---|---|---|
| Стан на сервері | Немає — сервер не зберігає сесію | Є — сесія зберігається в БД/Redis |
| Масштабованість | Висока — будь-який сервер може валідувати | Потребує спільного сховища сесій |
| Відкликання | Складне — токен валідний до `exp` | Просте — видалити сесію з БД |
| Розмір запиту | Більший — токен у кожному запиті | Менший — тільки session ID в cookie |
| Підходить для | Мікросервіси, API, мобільні клієнти | Монолітні веб-застосунки |

---

**Q: Як відкликати JWT до закінчення його терміну дії?**

A: JWT за своєю природою stateless — сервер не зберігає їх. Способи відкликання:
- **Blacklist** — зберігати відкликані `jti` у Redis до часу `exp`
- **Short-lived tokens** — робити Access Token дуже короткоживучим (1-5 хв)
- **Token versioning** — зберігати `tokenVersion` у БД користувача; при logout — інкрементувати; перевіряти в кожному запиті

---

**Q: Що таке Access Token vs Refresh Token?**

A:
- **Access Token** — короткоживучий (~15 хв), передається в кожному запиті, надає доступ до ресурсів
- **Refresh Token** — довгоживучий (~7-30 днів), зберігається в HttpOnly cookie, використовується лише для отримання нового Access Token

Ротація Refresh Token: при кожному використанні Refresh Token видається новий, старий інвалідується.

---

**Q: Що таке алгоритми підпису JWT? RS256 vs HS256?**

A:
- **HS256** (HMAC + SHA-256) — симетричний: один секретний ключ для підпису і валідації. Простий, але ключ потрібен і на сервері видачі, і на сервері валідації.
- **RS256** (RSA + SHA-256) — асиметричний: приватний ключ підписує, публічний ключ валідує. Ідеально для мікросервісів — сервіси можуть валідувати токени, не маючи приватного ключа.

---

## Методи автентифікації та авторизації

**Q: Чим відрізняється автентифікація від авторизації?**

A:
- **Автентифікація (AuthN)** — хто ти? Перевірка особи (логін/пароль, біометрія, токен)
- **Авторизація (AuthZ)** — що тобі дозволено? Перевірка прав доступу до ресурсів

---

**Q: Що таке OAuth 2.0?**

A: Відкритий протокол авторизації, що дозволяє третім сторонам отримати обмежений доступ до ресурсів від імені користувача без передачі паролів.

Основні ролі:
- **Resource Owner** — користувач
- **Client** — застосунок, що запитує доступ
- **Authorization Server** — видає токени (Google, GitHub тощо)
- **Resource Server** — API з захищеними ресурсами

Основні flows:
- **Authorization Code** (з PKCE) — для веб-застосунків; найбезпечніший
- **Client Credentials** — для machine-to-machine (сервіс до сервісу)
- **Implicit** (застарілий) — токен у URL, не рекомендується
- **Device Code** — для пристроїв без браузера (TV, CLI)

---

**Q: Що таке PKCE і навіщо він?**

A: Proof Key for Code Exchange — розширення Authorization Code flow. Захищає від перехоплення authorization code. Клієнт генерує `code_verifier` (випадковий рядок) → хешує в `code_challenge` → надсилає з запитом. При обміні code на token — надсилає оригінальний `code_verifier`. Сервер перевіряє відповідність.

---

**Q: Що таке OpenID Connect (OIDC)?**

A: Шар ідентифікації поверх OAuth 2.0. OAuth дає авторизацію, OIDC додає автентифікацію — повертає `id_token` (JWT з інформацією про користувача: `name`, `email`, `sub`). Використовується для SSO (Single Sign-On).

---

**Q: Що таке SSO (Single Sign-On)?**

A: Механізм, що дозволяє користувачу аутентифікуватись один раз і отримати доступ до кількох незалежних систем без повторного введення credentials. Реалізується через SAML або OIDC.

---

**Q: Що таке Basic Auth?**

A: Найпростіший метод — `login:password` кодується у Base64 і надсилається в заголовку:
```
Authorization: Basic dXNlcjpwYXNz
```
Небезпечно без HTTPS (Base64 ≠ шифрування). Підходить тільки для внутрішніх API або між сервісами.

---

**Q: Що таке API Key?**

A: Статичний секретний рядок, що видається клієнту для ідентифікації та авторизації запитів. Надсилається в заголовку (`X-API-Key: ...`) або query-параметрі. Простий, але не дозволяє гранулярного контролю прав. Використовується для server-to-server або публічних API.

---

**Q: Що таке MFA (Multi-Factor Authentication)?**

A: Додатковий рівень захисту — вимагає підтвердження особи кількома факторами:
- **Знання** — пароль, PIN
- **Наявність** — OTP (TOTP/HOTP), SMS, апаратний ключ (YubiKey)
- **Біометрія** — відбиток пальця, Face ID

---

**Q: Що таке RBAC та ABAC?**

A:
- **RBAC** (Role-Based Access Control) — права визначаються роллю користувача (`admin`, `editor`, `viewer`). Просто та зрозуміло.
- **ABAC** (Attribute-Based Access Control) — права визначаються атрибутами (час доступу, відділ, геолокація). Гнучкіше, але складніше.

---

**Q: Що таке CSRF і як захиститись?**

A: Cross-Site Request Forgery — атака, при якій шкідливий сайт змушує браузер користувача надіслати запит до іншого сайту, де він аутентифікований (через cookie).

Захист:
- **CSRF Token** — унікальний токен в кожній формі, сервер перевіряє його
- **`SameSite=Strict/Lax`** cookie — браузер не надсилає cookie в cross-site запитах
- **Double Submit Cookie** — CSRF token одночасно в cookie і в заголовку

---

## BE як ініціатор — Push-архітектури

**Q: Які є механізми, коли сервер ініціює передачу даних до клієнта?**

A: Є три основних підходи:
1. **Short/Long Polling** — клієнт регулярно питає сервер
2. **Server-Sent Events (SSE)** — одностороннє з'єднання, сервер пушить події
3. **WebSockets** — двостороннє з'єднання реального часу

---

**Q: Що таке Short Polling і коли використовувати?**

A: Клієнт надсилає HTTP-запити через фіксований інтервал.

```js
setInterval(async () => {
  const data = await fetch('/api/status');
  updateUI(await data.json());
}, 5000);
```

Переваги: простота, сумісність з будь-яким HTTP-сервером.
Недоліки: велике навантаження на сервер, затримка між запитами.
Коли використовувати: рідкісні оновлення, де затримка не критична.

---

**Q: Що таке Long Polling?**

A: Клієнт надсилає запит, сервер тримає з'єднання відкритим до появи нових даних (або timeout), потім відповідає — клієнт одразу надсилає новий запит.

```js
async function longPoll() {
  const res = await fetch('/api/events?lastId=123'); // сервер чекає нових даних
  const data = await res.json();
  processData(data);
  longPoll(); // одразу новий запит
}
```

Переваги: менша затримка ніж short polling, дані надходять майже миттєво.
Недоліки: тримає HTTP-з'єднання відкритим, складніша реалізація на сервері.

---

**Q: Що таке Server-Sent Events (SSE)?**

A: Однонаправлений канал: сервер → клієнт через постійне HTTP-з'єднання. Сервер надсилає текстові події в форматі `data: ...\n\n`.

```js
// Клієнт
const es = new EventSource('/api/stream');
es.onmessage = (event) => {
  const data = JSON.parse(event.data);
  updateUI(data);
};
es.addEventListener('custom-event', (e) => { /* ... */ });
es.onerror = () => es.close();
```

```
// Формат відповіді сервера
Content-Type: text/event-stream

event: price-update
data: {"symbol":"AAPL","price":185.42}

data: {"symbol":"GOOG","price":175.10}
```

Переваги: вбудована підтримка автоматичного перепідключення (`retry:`), легкість реалізації, працює через звичайний HTTP/2.
Недоліки: тільки сервер → клієнт, обмежена кількість одночасних з'єднань в HTTP/1.1 (6 на домен).
Коли використовувати: live feeds, сповіщення, прогрес виконання задач, AI streaming відповіді.

---

**Q: Що таке WebSockets?**

A: Протокол повнодуплексного зв'язку в реальному часі через одне TCP-з'єднання. Починається як HTTP-запит (Upgrade), потім перемикається на WS-протокол.

```js
// Клієнт
const ws = new WebSocket('wss://api.example.com/ws');

ws.onopen = () => ws.send(JSON.stringify({ type: 'subscribe', channel: 'prices' }));
ws.onmessage = (event) => processMessage(JSON.parse(event.data));
ws.onerror = (err) => console.error(err);
ws.onclose = () => reconnect(); // реконнект при розриві

// Надсилання повідомлень
ws.send(JSON.stringify({ type: 'ping' }));
```

Переваги: мінімальна затримка, двосторонній зв'язок, ефективний протокол (менший overhead ніж HTTP).
Недоліки: складніше масштабувати (sticky sessions або pub/sub брокер), потребує обробки реконнекту.
Коли використовувати: чати, мультиплеєрні ігри, торгові термінали, колаборативне редагування.

---

**Q: Що таке WebRTC і коли використовувати?**

A: Web Real-Time Communication — API для peer-to-peer зв'язку між браузерами (аудіо, відео, довільні дані) без проміжного сервера (після handshake). Використовується для відеодзвінків (Zoom, Meet), P2P передачі файлів, screen sharing.

---

**Q: Що таке Push Notifications (Web Push)?**

A: Механізм надсилання сповіщень до браузера навіть коли сайт закритий. Складається з:
1. **Service Worker** — фоновий скрипт, що слухає push-події
2. **Push API** — підписка клієнта на сервер (отримує endpoint)
3. **Notification API** — відображення сповіщення

```js
// Підписка
const sub = await reg.pushManager.subscribe({ userVisibleOnly: true, applicationServerKey });
await fetch('/api/push/subscribe', { method: 'POST', body: JSON.stringify(sub) });

// Service Worker
self.addEventListener('push', (event) => {
  const data = event.data.json();
  self.registration.showNotification(data.title, { body: data.body });
});
```

---

**Q: Що таке GraphQL Subscriptions?**

A: Механізм реального часу в GraphQL — клієнт підписується на події через WebSocket.

```graphql
subscription {
  orderStatusUpdated(orderId: "123") {
    status
    updatedAt
  }
}
```

Сервер надсилає дані кожного разу, коли відбувається підписана подія. Реалізується через `graphql-ws` або `subscriptions-transport-ws`.

---

**Q: Як порівняти SSE, WebSockets та Long Polling?**

A:
| | Long Polling | SSE | WebSockets |
|---|---|---|---|
| Напрямок | Двосторонній (через нові запити) | Сервер → клієнт | Двосторонній |
| Протокол | HTTP | HTTP | WS (окремий протокол) |
| Авто-реконнект | Ручний | Вбудований | Ручний |
| Складність | Низька | Низька | Середня |
| Overhead | Високий (HTTP headers) | Низький | Мінімальний |
| Масштабування | Просте | Просте | Потребує координації |
| Підтримка | Скрізь | Всі сучасні браузери | Всі сучасні браузери |

---

## Відображення та агрегація даних з BE

**Q: Як правильно відображати великі списки даних з BE (virtualization)?**

A: Рендерити лише елементи, що видимі у viewport — **Virtual Scrolling / Windowing**.

Бібліотеки:
- `react-window` — легка, для простих списків
- `react-virtual` (TanStack Virtual) — гнучкіша

```jsx
import { FixedSizeList } from 'react-window';

<FixedSizeList height={600} itemCount={10000} itemSize={50} width="100%">
  {({ index, style }) => (
    <div style={style}>Row {index}</div>
  )}
</FixedSizeList>
```

Рендериться лише ~20 елементів замість 10 000 — різниця в продуктивності кардинальна.

---

**Q: Що таке infinite scroll і як реалізувати?**

A: Підвантаження нових даних при наближенні до кінця списку, замість пагінації.

```js
// Intersection Observer — найефективніший спосіб
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting && hasNextPage) {
    fetchNextPage();
  }
}, { threshold: 0.1 });

observer.observe(sentinelRef.current); // "сторожовий" елемент в кінці списку
```

З TanStack Query:
```js
const { data, fetchNextPage, hasNextPage } = useInfiniteQuery({
  queryKey: ['items'],
  queryFn: ({ pageParam = 0 }) => fetchItems(pageParam),
  getNextPageParam: (lastPage) => lastPage.nextCursor,
});
```

---

**Q: Що таке пагінація на BE і які її типи?**

A:
- **Offset pagination** — `?page=3&limit=20` → `OFFSET 60 LIMIT 20`. Проста, але повільна на великих таблицях (БД сканує всі попередні рядки).
- **Cursor pagination** — `?after=cursor_id` → `WHERE id > cursor_id LIMIT 20`. Ефективна, стабільна при вставці нових записів, але не підтримує перехід на довільну сторінку.
- **Keyset pagination** — різновид cursor, використовує індексований стовпець (`created_at`).

---

**Q: Як агрегувати дані з BE для чартів?**

A: Агрегацію по можливості виконувати на BE (або в БД), не на FE — клієнт отримує вже готові дані.

Паттерни:
```js
// BE повертає готові агреговані дані
GET /api/analytics/revenue?from=2026-01-01&to=2026-06-01&groupBy=month

// Відповідь
[
  { "period": "2026-01", "revenue": 45200, "orders": 312 },
  { "period": "2026-02", "revenue": 51800, "orders": 367 },
  ...
]
```

На FE — мінімальна трансформація:
```js
const chartData = apiData.map(d => ({
  x: d.period,
  y: d.revenue,
}));
```

---

**Q: Які бібліотеки використовувати для чартів в React?**

A:
| Бібліотека | Особливості | Коли використовувати |
|---|---|---|
| **Recharts** | Декларативна, на SVG, легко кастомізується | Більшість задач, вбудована в React |
| **Chart.js + react-chartjs-2** | Canvas-based, велика екосистема, висока продуктивність | Велика кількість точок даних |
| **D3.js** | Максимальна гнучкість, низькорівнева | Нестандартні візуалізації |
| **Plotly** | Наукові графіки, інтерактивність з коробки | Дата-аналіз, фінанси |
| **Victory** | Легка, composable | Прості проекти |
| **Nivo** | Красивий дизайн, SSR-підтримка | Дашборди |

---

**Q: Як реалізувати real-time чарт з даними з BE (WebSocket/SSE)?**

A:
```jsx
import { useState, useEffect, useRef } from 'react';
import { LineChart, Line, XAxis, YAxis, Tooltip } from 'recharts';

const MAX_POINTS = 100; // обмежуємо кількість точок у пам'яті

function RealtimeChart() {
  const [data, setData] = useState([]);

  useEffect(() => {
    const es = new EventSource('/api/stream/prices');
    es.onmessage = (e) => {
      const point = JSON.parse(e.data);
      setData(prev => {
        const next = [...prev, point];
        return next.length > MAX_POINTS ? next.slice(-MAX_POINTS) : next;
      });
    };
    return () => es.close();
  }, []);

  return (
    <LineChart width={800} height={300} data={data}>
      <XAxis dataKey="time" />
      <YAxis />
      <Tooltip />
      <Line type="monotone" dataKey="value" dot={false} isAnimationActive={false} />
    </LineChart>
  );
}
```

`isAnimationActive={false}` — критично для real-time, анімація при кожному оновленні вбиває продуктивність.

---

**Q: Як оптимізувати ре-рендери при частих оновленнях даних?**

A:
- `useMemo` — мемоізувати трансформацію даних для чарту
- `useCallback` — мемоізувати обробники
- Throttling/debouncing оновлень стану при дуже частих подіях (>30fps):
  ```js
  const throttledUpdate = useCallback(
    throttle((point) => setData(prev => [...prev.slice(-99), point]), 100),
    []
  );
  ```
- Розділити стан: зберігати дані в `useRef`, оновлювати DOM безпосередньо (поза React) для критичних по частоті оновлень — `canvas`-based рендеринг (Chart.js)

---

## Робота з великими даними з BE

**Q: Які проблеми виникають при роботі з великими даними на FE?**

A:
- Довгий час завантаження (великий payload)
- Блокування UI thread при парсингу/обробці
- Переповнення пам'яті
- Повільний рендеринг великих списків
- Складність state management

---

**Q: Що таке Web Workers і як вони допомагають?**

A: Web Workers виконують JavaScript у фоновому потоці, не блокуючи UI thread. Ідеальні для важких обчислень над великими даними.

```js
// worker.js
self.onmessage = (e) => {
  const { data } = e;
  const result = heavyComputation(data); // не блокує UI
  self.postMessage(result);
};

// main thread
const worker = new Worker('/worker.js');
worker.postMessage(rawData);
worker.onmessage = (e) => setProcessedData(e.data);
```

Обмеження: немає доступу до DOM, передача даних через копіювання (або `SharedArrayBuffer`).

---

**Q: Що таке Streaming і як отримувати великі дані потоком?**

A: Замість очікування повного завантаження — обробляти дані по частинах через `ReadableStream` / `fetch` streaming.

```js
const response = await fetch('/api/large-dataset');
const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  const chunk = decoder.decode(value, { stream: true });
  processChunk(chunk); // обробляємо кожен chunk одразу
}
```

Використовується для: AI streaming відповідей, великих CSV/JSON файлів, live logs.

---

**Q: Що таке lazy loading даних і як реалізувати?**

A: Завантажувати лише необхідні дані, решту — по запиту. Для таблиць і списків:

```js
// TanStack Query з infinite loading
const { data, fetchNextPage } = useInfiniteQuery({
  queryKey: ['records'],
  queryFn: ({ pageParam }) => api.getRecords({ cursor: pageParam, limit: 50 }),
  getNextPageParam: (last) => last.nextCursor,
});

// Для детальних даних — завантажувати при відкритті
const { data: details } = useQuery({
  queryKey: ['record', id],
  queryFn: () => api.getRecord(id),
  enabled: !!id, // завантажувати лише коли є id
});
```

---

**Q: Що таке data normalization і навіщо вона при великих даних?**

A: Нормалізація — зберігання даних у плоскій структурі з посиланнями замість вкладених об'єктів. Запобігає дублюванню та спрощує оновлення.

```js
// Денормалізовано (проблема: дублювання автора в кожному пості)
const posts = [
  { id: 1, title: '...', author: { id: 5, name: 'John' } },
  { id: 2, title: '...', author: { id: 5, name: 'John' } },
];

// Нормалізовано (Redux Toolkit + EntityAdapter, або Normalizr)
const state = {
  posts: { ids: [1, 2], entities: { 1: { id: 1, authorId: 5 }, 2: { id: 2, authorId: 5 } } },
  users: { ids: [5], entities: { 5: { id: 5, name: 'John' } } },
};
```

---

**Q: Як кешувати дані з BE на FE і яку бібліотеку використати?**

A:
**TanStack Query (React Query)** — де-факто стандарт:
- Автоматичне кешування та інвалідація
- Background refetch при фокусі вкладки
- Оптимістичні оновлення
- Infinite query для пагінації
- Stale-while-revalidate стратегія

```js
const { data, isLoading, error } = useQuery({
  queryKey: ['users', filters], // ключ кешу
  queryFn: () => api.getUsers(filters),
  staleTime: 5 * 60 * 1000, // 5 хв — дані вважаються свіжими
  gcTime: 10 * 60 * 1000,   // 10 хв — після цього видаляються з кешу
});
```

---

**Q: Що таке optimistic updates і коли використовувати?**

A: Оновлення UI одразу, не чекаючи відповіді від сервера. При помилці — відкат до попереднього стану.

```js
const mutation = useMutation({
  mutationFn: api.updateItem,
  onMutate: async (newItem) => {
    await queryClient.cancelQueries({ queryKey: ['items'] });
    const previousItems = queryClient.getQueryData(['items']);
    queryClient.setQueryData(['items'], (old) =>
      old.map(item => item.id === newItem.id ? { ...item, ...newItem } : item)
    );
    return { previousItems }; // для відкату
  },
  onError: (err, newItem, context) => {
    queryClient.setQueryData(['items'], context.previousItems); // відкат
  },
  onSettled: () => queryClient.invalidateQueries({ queryKey: ['items'] }),
});
```

---

**Q: Як обробляти великі таблиці даних (десятки тисяч рядків)?**

A: Комплексний підхід:
1. **Server-side pagination/sorting/filtering** — на BE, не на FE
2. **Virtual scrolling** — `react-window` або `TanStack Virtual`
3. **Canvas-based рендеринг** — для grid: `AG Grid`, `react-data-grid` (рендерять через canvas або оптимізований DOM)
4. **Web Worker** для сортування/фільтрації на FE якщо потрібно
5. **Memoization** — `useMemo` для трансформацій, `React.memo` для рядків таблиці

```jsx
// TanStack Virtual + TanStack Table — золотий стандарт
import { useVirtualizer } from '@tanstack/react-virtual';

const rowVirtualizer = useVirtualizer({
  count: rows.length,
  getScrollElement: () => parentRef.current,
  estimateSize: () => 40,
  overscan: 10,
});
```

---

**Q: Що таке AbortController і навіщо він при роботі з API?**

A: Дозволяє скасувати fetch-запит. Критично для запобігання race conditions та memory leaks при швидкій зміні параметрів (наприклад, пошук при введенні).

```js
useEffect(() => {
  const controller = new AbortController();

  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(res => res.json())
    .then(setResults)
    .catch(err => {
      if (err.name !== 'AbortError') console.error(err);
    });

  return () => controller.abort(); // скасовуємо при unmount або зміні query
}, [query]);
```

---

**Q: Що таке race condition в контексті API-запитів і як уникнути?**

A: Race condition — ситуація, коли відповіді на запити приходять не в тому порядку, в якому були надіслані.

```js
// Проблема: якщо запит 2 відповість раніше за запит 1 — покажемо застарілі дані
setQuery('re');    // запит 1
setQuery('react'); // запит 2 — може відповісти першим!

// Рішення 1: AbortController (скасовуємо попередній запит)
// Рішення 2: ігнорувати застарілі відповіді через флаг
useEffect(() => {
  let isActive = true;
  fetchData(query).then(data => {
    if (isActive) setData(data);
  });
  return () => { isActive = false; };
}, [query]);

// Рішення 3: TanStack Query обробляє це автоматично
```

---

*Файл оновлюється по мірі надходження нових матеріалів.*
