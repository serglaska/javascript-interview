# Senior+ Front-End Engineer — Interview Q&A

> Остання актуалізація: 2026-06-15

---

## Зміст
1. [Browser Internals — як браузер рендерить сторінку](#browser-internals)
2. [Performance — Core Web Vitals та оптимізація](#performance)
3. [Security — XSS, CSP, CORS та інше](#security)
4. [TypeScript — Advanced](#typescript--advanced)
5. [Testing — Стратегії та підходи](#testing)
6. [Architecture — Патерни та підходи](#architecture)
7. [SSR / SSG / Hydration](#ssr--ssg--hydration)
8. [Build Tools — Webpack, Vite, оптимізація бандлу](#build-tools)
9. [Advanced React — Concurrent, Suspense, Context](#advanced-react)
10. [Accessibility (a11y)](#accessibility-a11y)
11. [System Design для Front-End](#system-design-для-front-end)
12. [SOLID та принципи чистого коду](#solid-та-принципи-чистого-коду)

---

## Browser Internals

**Q: Опиши критичний шлях рендерингу (Critical Rendering Path).**

A: Послідовність кроків, яку браузер виконує від отримання HTML до відображення пікселів:

```
HTML → DOM → ┐
             ├→ Render Tree → Layout → Paint → Composite
CSS  → CSSOM → ┘
```

1. **Parse HTML → DOM** — байти HTML парсяться в токени, токени в вузли, вузли в DOM-дерево
2. **Parse CSS → CSSOM** — аналогічно для CSS; рендеринг блокується до побудови CSSOM
3. **JavaScript** — блокує парсинг HTML (якщо без `async`/`defer`); може змінювати DOM і CSSOM
4. **Render Tree** — об'єднання DOM + CSSOM; невидимі елементи (`display: none`) не включаються
5. **Layout (Reflow)** — обчислення геометрії кожного вузла (позиція, розмір)
6. **Paint** — заповнення пікселів (колір, тіні, зображення)
7. **Composite** — об'єднання шарів (layers) на GPU

---

**Q: Що таке Reflow та Repaint? Чим небезпечні?**

A:
- **Reflow** — перерахунок геометрії елементів. Найдорожча операція. Тригери: зміна розміру/позиції елемента, зміна DOM, зміна шрифту, `window.resize`.
- **Repaint** — перемальовування пікселів без зміни геометрії. Тригери: зміна `color`, `background`, `opacity`, `visibility`.

Небезпека: **Layout thrashing** — почергове читання і запис геометричних властивостей у циклі змушує браузер робити reflow на кожній ітерації.

```js
// Погано — layout thrashing
elements.forEach(el => {
  const height = el.offsetHeight; // читання → примусовий reflow
  el.style.height = height * 2 + 'px'; // запис
});

// Добре — спочатку всі читання, потім всі записи
const heights = elements.map(el => el.offsetHeight); // всі читання
elements.forEach((el, i) => el.style.height = heights[i] * 2 + 'px'); // всі записи
```

---

**Q: Що таке Composite Layer і як їх використовувати?**

A: Браузер може виділити елемент на окремий GPU-шар (composite layer), змінення якого не потребує reflow і repaint — тільки compositing.

Тригери створення нового шару:
- `transform: translateZ(0)` або `will-change: transform`
- `opacity < 1` з анімацією
- `position: fixed/sticky`
- `filter`, `backdrop-filter`

```css
/* Оптимізована CSS-анімація — тільки compositing, без reflow/repaint */
.animated {
  will-change: transform, opacity;
  animation: slide 0.3s ease;
}

/* Уникати анімації властивостей, що тригерять reflow */
/* Погано */ animation: width 0.3s; /* reflow на кожному кадрі */
/* Добре */  animation: transform 0.3s; /* тільки compositing */
```

---

**Q: Що таке requestAnimationFrame і коли використовувати?**

A: API для виконання коду перед наступним малюванням кадру браузером. Гарантує 60fps (16.67ms на кадр), синхронізований з дисплеєм.

```js
function animate(timestamp) {
  const progress = (timestamp - start) / duration;
  element.style.transform = `translateX(${progress * 300}px)`;
  if (progress < 1) requestAnimationFrame(animate);
}
requestAnimationFrame(animate);
```

Використовувати замість `setInterval` для плавних анімацій. Браузер призупиняє rAF при прихованій вкладці — економить батарею.

---

**Q: Чим відрізняється `async` від `defer` для тегу `<script>`?**

A:
| | Звичайний | `async` | `defer` |
|---|---|---|---|
| Парсинг HTML | Блокується | Не блокується | Не блокується |
| Завантаження | Синхронне | Паралельне | Паралельне |
| Виконання | Одразу | Одразу після завантаження | Після завершення парсингу HTML |
| Порядок | Збережений | Не гарантований | Збережений |

`defer` — для скриптів, що залежать від DOM або один від одного. `async` — для незалежних скриптів (аналітика, реклама).

---

**Q: Що відбувається при введенні URL в браузер і натисканні Enter?**

A:
1. **DNS lookup** — резолвить домен в IP (кеш браузера → кеш ОС → DNS-сервер)
2. **TCP handshake** — встановлення з'єднання (SYN → SYN-ACK → ACK)
3. **TLS handshake** — для HTTPS: обмін сертифікатами, узгодження шифрування
4. **HTTP запит** — браузер надсилає GET-запит
5. **Сервер відповідає** — HTML, заголовки (включно з кешуванням)
6. **Critical Rendering Path** — парсинг HTML, CSS, JS, побудова DOM/CSSOM
7. **Рендеринг** — Layout → Paint → Composite

---

## Performance

**Q: Що таке Core Web Vitals і які їх порогові значення?**

A: Метрики Google для оцінки UX:

| Метрика | Опис | Добре | Потребує покращення | Погано |
|---|---|---|---|---|
| **LCP** (Largest Contentful Paint) | Час рендерингу найбільшого елемента | ≤ 2.5s | 2.5–4s | > 4s |
| **INP** (Interaction to Next Paint) | Час реакції на взаємодію | ≤ 200ms | 200–500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | Несподівані зміщення контенту | ≤ 0.1 | 0.1–0.25 | > 0.25 |

---

**Q: Як покращити LCP?**

A:
- **Preload** найбільший елемент: `<link rel="preload" as="image" href="hero.webp">`
- Використовувати **WebP/AVIF** замість PNG/JPEG
- **Server-side rendering** — HTML одразу містить контент
- Усунути **render-blocking ресурси** (CSS, JS)
- **CDN** — зменшити TTFB (Time to First Byte)
- `fetchpriority="high"` для hero-зображення
- Уникати `lazy` loading для above-the-fold зображень

---

**Q: Як усунути CLS (Layout Shift)?**

A:
- Завжди вказувати `width` і `height` для `<img>` і `<video>` — браузер резервує місце
- Резервувати місце для динамічного контенту (скелетони, `min-height`)
- Уникати вставки контенту над існуючим (банери, сповіщення)
- `font-display: swap` з `size-adjust` для кастомних шрифтів

---

**Q: Що таке Code Splitting і як реалізувати?**

A: Поділ бандлу на менші частини, завантаження яких відбувається за потреби (lazy loading).

```js
// Route-based splitting — стандартний підхід
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

<Suspense fallback={<Spinner />}>
  <Routes>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/settings" element={<Settings />} />
  </Routes>
</Suspense>

// Component-based splitting — для важких компонентів
const HeavyChart = lazy(() => import('./components/HeavyChart'));

// Prefetch — завантажити наперед при hover
const prefetch = () => import('./pages/Settings'); // без lazy
button.addEventListener('mouseover', prefetch);
```

---

**Q: Що таке Tree Shaking?**

A: Видалення мертвого коду (unused exports) при збірці. Працює тільки з ES Modules (`import/export`), не з CommonJS (`require`). Webpack і Vite виконують tree shaking автоматично в production.

```js
// math.js
export const add = (a, b) => a + b;
export const multiply = (a, b) => a * b; // не використовується → буде видалено

// main.js
import { add } from './math'; // multiply не імпортується → tree-shaken
```

Важливо: `sideEffects: false` в `package.json` дозволяє Webpack агресивніше видаляти код.

---

**Q: Що таке мемоізація в React і коли її застосовувати?**

A: Кешування результату обчислення або компонента для уникнення зайвих ре-рендерів.

- `React.memo(Component)` — не ре-рендерить компонент, якщо пропси не змінились (shallow compare)
- `useMemo(fn, deps)` — кешує обчислене значення
- `useCallback(fn, deps)` — кешує функцію (стабільне посилання)

```js
// Антипатерн: надмірна мемоізація
const value = useMemo(() => a + b, [a, b]); // просте обчислення — зайва мемоізація

// Правильно: мемоізувати важкі обчислення або стабілізувати референції
const sortedList = useMemo(() => expensiveSort(items), [items]);
const handleClick = useCallback((id) => dispatch(remove(id)), [dispatch]);
```

---

**Q: Що таке bundle analyzer і як зменшити розмір бандлу?**

A: `webpack-bundle-analyzer` або `rollup-plugin-visualizer` — показують, що займає найбільше місця.

Техніки зменшення:
- Tree shaking + `sideEffects: false`
- Code splitting (lazy imports)
- Замінити важкі бібліотеки легшими: `moment.js` → `date-fns`, `lodash` → нативний JS
- `externals` — винести React/ReactDOM у CDN
- Стиснення: gzip або Brotli на сервері
- Dynamic imports для рідко використовуваних фіч

---

**Q: Що таке Service Worker і як реалізувати офлайн-режим?**

A: Service Worker — скрипт, що працює у фоновому потоці між браузером і мережею. Може перехоплювати запити, кешувати відповіді, працювати офлайн.

```js
// sw.js — стратегія Cache First
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      return cached || fetch(event.request).then(response => {
        const clone = response.clone();
        caches.open('v1').then(cache => cache.put(event.request, clone));
        return response;
      });
    })
  );
});
```

Стратегії кешування:
- **Cache First** — офлайн-застосунки, статичні ресурси
- **Network First** — актуальні дані важливіші за кеш
- **Stale While Revalidate** — показати кеш, оновити у фоні

---

## Security

**Q: Що таке XSS (Cross-Site Scripting) і як захиститись?**

A: Атака, при якій зловмисник впроваджує шкідливий JS-код на сторінку, що виконується у браузері жертви.

Типи:
- **Stored XSS** — шкідливий код збережений у БД (коментарі, пости)
- **Reflected XSS** — код у URL-параметрі, що відображається у відповіді
- **DOM-based XSS** — маніпуляція DOM через небезпечні API

Захист:
```js
// 1. Ніколи не вставляти невалідовані дані в innerHTML
element.innerHTML = userInput; // НЕБЕЗПЕЧНО
element.textContent = userInput; // БЕЗПЕЧНО

// 2. React за замовчуванням екранує JSX
<div>{userInput}</div> // безпечно — React екранує
<div dangerouslySetInnerHTML={{ __html: input }} /> // небезпечно — тільки якщо потрібно

// 3. DOMPurify для sanitization HTML
import DOMPurify from 'dompurify';
element.innerHTML = DOMPurify.sanitize(userHTML);
```

---

**Q: Що таке CSP (Content Security Policy)?**

A: HTTP-заголовок, що визначає, які ресурси (скрипти, стилі, зображення) браузер може завантажити. Найефективніший захист від XSS.

```
Content-Security-Policy:
  default-src 'self';
  script-src 'self' https://cdn.example.com;
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
```

`'self'` — тільки власний домен. `'unsafe-inline'` — дозволяє inline-скрипти (небажано). `nonce` — дозволяє конкретний inline-скрипт за унікальним токеном.

---

**Q: Що таке CORS і як він працює?**

A: Cross-Origin Resource Sharing — механізм браузера, що обмежує HTTP-запити між різними origins (протокол + домен + порт).

**Простий запит** (`GET/POST` з простими заголовками) — браузер відправляє запит, перевіряє `Access-Control-Allow-Origin` у відповіді.

**Preflight** — для складних запитів (`PUT/DELETE`, кастомні заголовки) браузер спочатку надсилає `OPTIONS`:
```
OPTIONS /api/data HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: PUT
```

Сервер відповідає:
```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

CORS — браузерний механізм. Server-to-server запити не підпадають під CORS.

---

**Q: Що таке Subresource Integrity (SRI)?**

A: Механізм перевірки цілісності ресурсів (CDN), завантажених з зовнішніх джерел. Браузер порівнює hash завантаженого файлу з очікуваним.

```html
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC"
  crossorigin="anonymous">
</script>
```

---

**Q: Що таке `X-Frame-Options` та `Clickjacking`?**

A: Clickjacking — атака, коли шкідливий сайт вбудовує цільову сторінку в прозорий `iframe` і обманює користувача натискати на прихований UI.

Захист:
```
X-Frame-Options: DENY                    # застарілий
Content-Security-Policy: frame-ancestors 'none'  # сучасний підхід
```

---

**Q: Що таке HTTPS / TLS і що він захищає?**

A: TLS (Transport Layer Security) шифрує передачу даних між клієнтом і сервером. Захищає від:
- **Перехоплення** (eavesdropping) — дані зашифровані
- **Підробки** (tampering) — MAC (Message Authentication Code) перевіряє цілісність
- **Підміни сервера** (spoofing) — сертифікат підтверджує ідентичність сервера

TLS handshake: Client Hello → Server Hello + Certificate → Key Exchange → Finished. HTTP/2 та HTTP/3 вимагають TLS.

---

## TypeScript — Advanced

**Q: Що таке Generic Types? Коли використовувати?**

A: Дозволяють писати компоненти, що працюють з будь-яким типом, зберігаючи type safety.

```ts
// Без generics — втрачаємо тип
function first(arr: any[]): any { return arr[0]; }

// З generics — тип зберігається
function first<T>(arr: T[]): T { return arr[0]; }

const num = first([1, 2, 3]);       // number
const str = first(['a', 'b', 'c']); // string

// Constraints — обмежити тип
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

---

**Q: Що таке Utility Types? Назви основні.**

A:
```ts
type User = { id: number; name: string; email: string; age?: number };

Partial<User>          // всі поля опціональні
Required<User>         // всі поля обов'язкові
Readonly<User>         // всі поля read-only
Pick<User, 'id' | 'name'>      // вибрати підмножину полів
Omit<User, 'email'>            // виключити поля
Record<string, User>           // { [key: string]: User }
Exclude<'a' | 'b' | 'c', 'a'> // 'b' | 'c'
Extract<'a' | 'b', 'b' | 'c'> // 'b'
NonNullable<string | null | undefined> // string
ReturnType<typeof fn>          // тип, який повертає функція
Parameters<typeof fn>          // tuple типів параметрів функції
```

---

**Q: Що таке Conditional Types?**

A:
```ts
type IsString<T> = T extends string ? 'yes' : 'no';

type A = IsString<string>; // 'yes'
type B = IsString<number>; // 'no'

// Практичний приклад — Flatten масиву
type Flatten<T> = T extends Array<infer Item> ? Item : T;
type Str = Flatten<string[]>; // string
type Num = Flatten<number>;   // number

// infer — виведення типу зсередини
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
```

---

**Q: Що таке `never`, `unknown`, `any` — різниця?**

A:
- **`any`** — вимикає type checking. Уникати — втрачається сенс TypeScript.
- **`unknown`** — як `any`, але безпечний: не можна використовувати без звуження типу (`typeof`, `instanceof`).
- **`never`** — тип, якого не існує. Функція, що завжди кидає помилку або нескінченний цикл. Використовується для exhaustive checks:

```ts
// unknown — безпечна альтернатива any
function process(input: unknown) {
  if (typeof input === 'string') {
    console.log(input.toUpperCase()); // OK — тип звужений
  }
}

// never — exhaustive check
type Shape = 'circle' | 'square';
function area(shape: Shape): number {
  switch (shape) {
    case 'circle': return Math.PI;
    case 'square': return 1;
    default:
      const _exhaustive: never = shape; // помилка якщо додали новий тип і не обробили
      throw new Error(`Unknown: ${_exhaustive}`);
  }
}
```

---

**Q: Що таке Declaration Merging?**

A: TypeScript дозволяє об'єднувати кілька оголошень з однаковим ім'ям.

```ts
// Розширення сторонніх типів (наприклад, Express Request)
declare namespace Express {
  interface Request {
    user?: User;
  }
}

// Розширення Window
interface Window {
  myAnalytics: Analytics;
}
```

---

**Q: `type` vs `interface` — коли що використовувати?**

A:
- `interface` — для опису форми об'єктів; підтримує Declaration Merging; кращий для публічних API бібліотек
- `type` — для union types, intersection types, умовних типів, mapped types, псевдонімів примітивів

```ts
// type — для складних типів
type ID = string | number;
type Status = 'active' | 'inactive';
type UserOrAdmin = User & Admin;

// interface — для об'єктів
interface Repository<T> {
  findById(id: number): Promise<T>;
  save(entity: T): Promise<T>;
}
```

---

## Testing

**Q: Що таке Testing Pyramid?**

A: Концепція балансу типів тестів:

```
        /\
       /E2E\         <- мало, повільні, дорогі
      /------\
     /Integrat\      <- середня кількість
    /----------\
   /  Unit Tests \   <- багато, швидкі, дешеві
  /--------------\
```

- **Unit** — тестують окремі функції/компоненти ізольовано. Інструменти: Jest, Vitest
- **Integration** — тестують взаємодію між модулями (компонент + хук + API). Інструмент: React Testing Library
- **E2E** — тестують повний user flow у браузері. Інструменти: Cypress, Playwright

---

**Q: Що таке React Testing Library і яка її філософія?**

A: Бібліотека для тестування React-компонентів так, як їх використовує реальний користувач — через DOM, не через implementation details.

```js
import { render, screen, userEvent } from '@testing-library/react';

test('submits login form', async () => {
  const onLogin = jest.fn();
  render(<LoginForm onLogin={onLogin} />);

  await userEvent.type(screen.getByLabelText('Email'), 'test@test.com');
  await userEvent.type(screen.getByLabelText('Password'), 'secret');
  await userEvent.click(screen.getByRole('button', { name: /sign in/i }));

  expect(onLogin).toHaveBeenCalledWith({ email: 'test@test.com', password: 'secret' });
});
```

Принцип: "Test behavior, not implementation." Уникати тестування state, methods, props безпосередньо.

---

**Q: Що таке мокування (mocking) і коли використовувати?**

A: Заміна реальних залежностей на контрольовані підробки.

```js
// Mock модуля
jest.mock('./api', () => ({
  fetchUser: jest.fn().mockResolvedValue({ id: 1, name: 'John' }),
}));

// Mock fetch
global.fetch = jest.fn().mockResolvedValue({
  ok: true,
  json: () => Promise.resolve({ data: [] }),
});

// Spies — відстежувати виклики реальних функцій
const spy = jest.spyOn(console, 'error').mockImplementation(() => {});
// ... тест ...
expect(spy).not.toHaveBeenCalled();
spy.mockRestore();
```

---

**Q: Що таке TDD (Test-Driven Development)?**

A: Підхід: спочатку пишеться тест (red) → мінімальний код для проходження (green) → рефакторинг (refactor). **Red → Green → Refactor**.

Переваги: продуманий API, висока покриваність тестами, безпечний рефакторинг.

---

**Q: Як тестувати кастомні хуки?**

A:
```js
import { renderHook, act } from '@testing-library/react';

test('useCounter increments', () => {
  const { result } = renderHook(() => useCounter(0));

  expect(result.current.count).toBe(0);
  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});
```

---

## Architecture

**Q: Що таке Micro-Frontends?**

A: Архітектурний підхід, при якому фронтенд застосунок ділиться на незалежні частини (мікрофронтенди), які розробляються, тестуються та деплояться окремо різними командами.

Підходи реалізації:
- **Module Federation** (Webpack 5) — завантаження JavaScript-модулів з різних origins в runtime
- **iframes** — повна ізоляція, але погана UX-інтеграція
- **Web Components** — нативна ізоляція через Custom Elements та Shadow DOM
- **Single-SPA** — фреймворк для оркестрації мікрофронтендів

```js
// webpack.config.js — Module Federation
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    catalog: 'catalog@https://catalog.example.com/remoteEntry.js',
    checkout: 'checkout@https://checkout.example.com/remoteEntry.js',
  },
});
```

---

**Q: Що таке Monorepo і які інструменти для нього?**

A: Один репозиторій для кількох пакетів/застосунків. Дозволяє спільно використовувати код, єдина система CI/CD, атомарні коміти.

Інструменти: **Turborepo**, **Nx**, pnpm workspaces, Yarn workspaces.

```
monorepo/
├── apps/
│   ├── web/         # Next.js app
│   └── mobile/      # React Native
├── packages/
│   ├── ui/          # Shared components
│   ├── utils/       # Shared utilities
│   └── api-client/  # Shared API client
└── turbo.json
```

---

**Q: Що таке Feature Flags і навіщо вони?**

A: Механізм вмикання/вимикання фіч без деплою — через конфігурацію (GrowthBook, LaunchDarkly, Unleash).

Використання:
- **Canary releases** — поступове відкриття для % користувачів
- **A/B тестування** — різні варіанти UI для різних сегментів
- **Kill switch** — швидко вимкнути проблемну фічу
- **Trunk-based development** — незавершені фічі в `main`, але приховані за флагом

```js
if (featureFlags.isEnabled('new-checkout')) {
  return <NewCheckout />;
}
return <OldCheckout />;
```

---

**Q: Що таке BFF (Backend for Frontend)?**

A: Окремий backend-сервіс, написаний і підтримуваний FE-командою, що агрегує дані з кількох мікросервісів та адаптує їх під потреби конкретного клієнта (web, mobile, TV).

```
Mobile App  ───→ BFF Mobile ───→ User Service
Web App     ───→ BFF Web    ───→ Order Service
                             └──→ Payment Service
```

Переваги: FE-оптимізований API, менше over-fetching, ізоляція від змін в мікросервісах.

---

## SSR / SSG / Hydration

**Q: Чим відрізняються CSR, SSR, SSG, ISR?**

A:
| | CSR | SSR | SSG | ISR |
|---|---|---|---|---|
| HTML генерується | На клієнті | На сервері per-request | При збірці | При збірці + реvalidation |
| TTFB | Швидкий | Повільніший | Дуже швидкий | Дуже швидкий |
| SEO | Погане | Відмінне | Відмінне | Відмінне |
| Актуальність даних | Завжди свіжі | Завжди свіжі | На момент build | Можна налаштувати |
| Навантаження сервера | Немає | Висока | Немає | Мінімальна |
| Приклад | Create React App | Next.js `getServerSideProps` | Next.js `getStaticProps` | Next.js `revalidate` |

---

**Q: Що таке Hydration і які проблеми з нею?**

A: Hydration — процес, при якому React бере статичний HTML, згенерований на сервері, та "оживляє" його — прикріплює обробники подій та синхронізує стан.

Проблеми:
- **Hydration mismatch** — HTML від сервера відрізняється від того, що React рендерить на клієнті → помилки та мерехтіння
- **Blocking hydration** — весь JS повинен завантажитись і виконатись перед тим, як сторінка стане інтерактивною

Рішення — **Partial Hydration** (Astro) та **Selective Hydration** (React 18 Suspense + SSR):
```jsx
// React 18 — Selective Hydration
// Компоненти в Suspense гідратуються незалежно
<Suspense fallback={<Spinner />}>
  <Comments /> {/* гідратується пізніше, не блокує решту */}
</Suspense>
```

---

**Q: Що таке Streaming SSR (React 18)?**

A: Сервер надсилає HTML по частинах (`chunks`) через HTTP streaming, не чекаючи повної генерації. Браузер починає рендерити HTML та завантажувати ресурси раніше.

```jsx
// Next.js App Router — streaming з React Suspense
export default function Page() {
  return (
    <main>
      <Header /> {/* надсилається одразу */}
      <Suspense fallback={<Skeleton />}>
        <SlowComponent /> {/* надсилається коли готово */}
      </Suspense>
    </main>
  );
}
```

---

## Build Tools

**Q: Чим відрізняється Vite від Webpack?**

A:
| | Webpack | Vite |
|---|---|---|
| Dev server | Збирає весь бандл при старті | ESM-based: браузер імпортує модулі напряму |
| HMR | Повільний на великих проектах | Блискавичний (оновлює тільки змінені модулі) |
| Production build | Webpack bundling | Rollup під капотом |
| Конфігурація | Складна | Мінімальна з коробки |
| Екосистема | Величезна, зріла | Сучасна, швидко зростає |

---

**Q: Що таке Source Maps і навіщо вони?**

A: Файли (`.map`), що відображають мінімізований/транспільований код назад на оригінальний. Дозволяють дебажити production-код у браузері або читати stack traces. В production — зберігати на сервері помилок (Sentry), не публікувати публічно.

---

**Q: Що таке tree shaking і які умови для його роботи?**

A: Видалення невикористаного коду. Умови:
1. ES Modules (`import/export`) — не CommonJS
2. `sideEffects: false` або перелік файлів з side effects у `package.json`
3. Production mode (мінімізатор видаляє мертвий код)

```json
// package.json
{
  "sideEffects": ["*.css", "src/polyfills.js"]
}
```

---

## Advanced React

**Q: Що таке React Concurrent Features?**

A: Набір можливостей React 18, що дозволяють розбивати рендеринг на перервані одиниці роботи — важливі оновлення (взаємодія з користувачем) мають пріоритет над менш важливими.

```js
// startTransition — позначити оновлення як низькопріоритетне
import { startTransition, useTransition } from 'react';

const [isPending, startTransition] = useTransition();

startTransition(() => {
  setSearchQuery(input); // не блокує введення тексту
});

// useDeferredValue — дефер значення для важких ре-рендерів
const deferredQuery = useDeferredValue(query);
// expensiveComponent перерендеривається з deferredQuery — не блокує UI
```

---

**Q: Що таке React Suspense?**

A: Механізм декларативного очікування — компонент може "призупинити" рендеринг, поки дані або код не завантажаться. Fallback показується автоматично.

```jsx
// Для lazy-loaded компонентів
<Suspense fallback={<Spinner />}>
  <LazyComponent />
</Suspense>

// Для data fetching (React 19 / frameworks)
<Suspense fallback={<Skeleton />}>
  <UserProfile userId={id} /> {/* кидає Promise → Suspense показує fallback */}
</Suspense>
```

---

**Q: Як оптимізувати React Context щоб уникнути зайвих ре-рендерів?**

A: Кожен споживач Context ре-рендериться при будь-якій зміні value. Рішення:

```jsx
// 1. Розділити контекст на дрібніші
const UserContext = createContext();   // рідко змінюється
const ThemeContext = createContext();  // змінюється окремо

// 2. Мемоізувати value
const value = useMemo(() => ({ user, updateUser }), [user]);
<UserContext.Provider value={value}>

// 3. Context + useReducer — для складного стану
const [state, dispatch] = useReducer(reducer, initialState);
// Передати dispatch окремо від state — dispatch стабільний
<DispatchContext.Provider value={dispatch}>
  <StateContext.Provider value={state}>
```

---

**Q: Що таке Portals і навіщо вони?**

A: `ReactDOM.createPortal(children, container)` — рендерить дочірній компонент поза DOM-ієрархією батьківського, але в межах React-дерева (bubbling подій працює).

```jsx
// Модальне вікно поза #root — уникаємо проблем з z-index та overflow: hidden
function Modal({ children }) {
  return ReactDOM.createPortal(
    <div className="modal">{children}</div>,
    document.getElementById('modal-root')
  );
}
```

Типові використання: модалки, tooltips, dropdowns, notifications.

---

## Accessibility (a11y)

**Q: Що таке WCAG і які рівні відповідності?**

A: Web Content Accessibility Guidelines — міжнародний стандарт доступності.

- **A** — базовий рівень
- **AA** — стандарт для більшості комерційних продуктів (вимагається законодавством у багатьох країнах)
- **AAA** — найвищий рівень, не завжди досяжний

Чотири принципи (POUR): Perceivable, Operable, Understandable, Robust.

---

**Q: Що таке ARIA і коли використовувати?**

A: Accessible Rich Internet Applications — HTML-атрибути для покращення доступності. Правило №1: **не використовуй ARIA якщо є нативний HTML-елемент**.

```html
<!-- Погано: div замість button + ARIA -->
<div role="button" tabindex="0" aria-label="Close">✕</div>

<!-- Добре: нативний button -->
<button aria-label="Close dialog">✕</button>

<!-- ARIA потрібна для кастомних компонентів -->
<div role="combobox" aria-expanded="true" aria-haspopup="listbox" aria-owns="list-id">
  <input aria-autocomplete="list" aria-controls="list-id" />
</div>
<ul id="list-id" role="listbox">
  <li role="option" aria-selected="false">Option 1</li>
</ul>
```

---

**Q: Що таке focus management і навіщо він?**

A: Програмне управління фокусом клавіатури — критично для модальних вікон, динамічного контенту, SPA-навігації.

```js
// При відкритті модалки — перемістити фокус всередину
useEffect(() => {
  if (isOpen) modalRef.current?.focus();
}, [isOpen]);

// Focus trap — не дати фокусу вийти з модалки
// При закритті — повернути фокус на тригер
useEffect(() => {
  if (!isOpen) triggerRef.current?.focus();
}, [isOpen]);
```

---

## System Design для Front-End

**Q: Як спроектувати Feed (стрічку новин) на зразок Twitter/Facebook?**

A:
- **API**: cursor-based pagination (`GET /feed?after=cursor&limit=20`)
- **Real-time**: WebSocket або SSE для нових постів
- **Rendering**: Virtual scrolling (react-window) для великих списків
- **State**: нові пости показувати через banner "Show 5 new posts" — не підвантажувати автоматично (збережемо позицію скролу)
- **Caching**: TanStack Query, infinite query
- **Images**: lazy loading, WebP, srcset для різних розмірів
- **Optimistic updates**: like/repost — одразу оновити UI

---

**Q: Як спроектувати Autocomplete (пошукові підказки)?**

A:
- **Debounce** введення (300ms) — не надсилати запит на кожен символ
- **AbortController** — скасовувати попередній запит при новому введенні
- **Caching** — зберігати результати для однакових запитів
- **Keyboard navigation** — стрілки, Enter, Escape; ARIA roles: `combobox`, `listbox`, `option`
- **Virtualization** — якщо результатів > 50-100
- **Highlight** — підкреслювати збіг в результатах

```js
const debouncedSearch = useMemo(() =>
  debounce(async (q, signal) => {
    const res = await fetch(`/api/search?q=${q}`, { signal });
    setSuggestions(await res.json());
  }, 300),
[]);

useEffect(() => {
  const controller = new AbortController();
  if (query.length > 1) debouncedSearch(query, controller.signal);
  return () => controller.abort();
}, [query]);
```

---

**Q: Як спроектувати форму з 50+ полями?**

A:
- **React Hook Form** (не контрольовані інпути) — набагато швидше ніж контрольовані при великих формах
- **Схема валідації** — Zod або Yup
- **Multi-step wizard** — поділити на секції, валідувати per-step
- **Збереження прогресу** — `localStorage` або backend draft
- **Conditional fields** — watchField → showField через `watch`
- **Performance**: `useFormContext` + `Controller` для складних UI-компонентів; `shouldUnregister: false` щоб зберігати значення прихованих полів

---

## SOLID та принципи чистого коду

**Q: Розкрий принципи SOLID стосовно FE-розробки.**

A:

**S — Single Responsibility**: Компонент робить одну річ. `UserCard` — відображає, не завантажує дані.

**O — Open/Closed**: Відкритий для розширення, закритий для зміни. Компонент `Button` з `variant` prop замість окремих `PrimaryButton`, `DangerButton`.

**L — Liskov Substitution**: Підкомпонент можна замінити батьківським без зміни поведінки. Специфічний `IconButton` має весь API `Button`.

**I — Interface Segregation**: Не змушувати компонент приймати props, які він не використовує. Розбити великий `UserProfileProps` на `UserAvatarProps`, `UserInfoProps`.

**D — Dependency Inversion**: Залежати від абстракцій, не від конкретних реалізацій. Хук `useRepository` замість прямого виклику `fetch`.

---

**Q: Що таке DRY, KISS, YAGNI?**

A:
- **DRY** (Don't Repeat Yourself) — виносити повторювану логіку в хуки, утиліти, компоненти
- **KISS** (Keep It Simple, Stupid) — прості рішення кращі за складні; не переускладнювати
- **YAGNI** (You Aren't Gonna Need It) — не додавати функціонал "на майбутнє"; реалізовувати лише те, що потрібно зараз

---

**Q: Що таке Separation of Concerns у контексті React?**

A: Розділення логіки, стану та UI:

```
hooks/useUserData.ts    ← логіка отримання та трансформації даних
stores/userStore.ts     ← глобальний стан
components/UserCard.tsx ← тільки UI, отримує дані через props
pages/UserPage.tsx      ← оркестрація: використовує хук, передає дані в компонент
```

Компонент не повинен знати, звідки приходять дані — тільки як їх відобразити.

---

**Q: Що таке колокація (Colocation)?**

A: Розміщення пов'язаних файлів поруч — тести, стилі, типи, хуки поряд з компонентом, який їх використовує. Протилежне до організації "за типом файлу".

```
// Погано — за типом
src/components/Button.tsx
src/tests/Button.test.tsx
src/styles/Button.module.css

// Добре — колокація
src/components/Button/
├── Button.tsx
├── Button.test.tsx
├── Button.module.css
└── index.ts
```

---

*Файл оновлюється по мірі надходження нових матеріалів.*
