# Junior / Basic Front-End — Interview Q&A

> Матеріали постійно поповнюються. Остання актуалізація: 2026-07-27

---

## Зміст
1. [JavaScript — Типи даних та змінні](#javascript--типи-даних-та-змінні)
2. [Оператори та порівняння](#оператори-та-порівняння)
3. [Hoisting та Scope](#hoisting-та-scope)
4. [Функції](#функції)
5. [Масиви](#масиви)
6. [Об'єкти та деструктуризація](#обєкти-та-деструктуризація)
7. [DOM — основи](#dom--основи)
8. [Події (Events)](#події-events)
9. [JSON та інше](#json-та-інше)

---

## JavaScript — Типи даних та змінні

**Q: Які типи даних є в JavaScript?**

A: 7 примітивних типів:
- `String`, `Number`, `Boolean`, `undefined`, `null`, `Symbol`, `BigInt`

І один непримітивний (посилальний) тип:
- `Object` (включно з масивами, функціями, датами тощо)

---

**Q: Чим відрізняються примітивні типи від посилальних (reference types)?**

A: Примітиви зберігаються **за значенням** — при копіюванні створюється незалежна копія. Об'єкти зберігаються **за посиланням** — змінна містить адресу в пам'яті, тому копія посилання вказує на той самий об'єкт.

```js
let a = 5;
let b = a;
b = 10;
console.log(a); // 5 — не змінився

const obj1 = { x: 1 };
const obj2 = obj1;
obj2.x = 100;
console.log(obj1.x); // 100 — той самий об'єкт
```

---

**Q: Чим відрізняються `var`, `let` та `const`?**

A:
| | `var` | `let` | `const` |
|---|---|---|---|
| Scope | функціональний | блочний | блочний |
| Redeclare | так | ні | ні |
| Reassign | так | так | ні |
| Hoisting | так, з `undefined` | так, в TDZ | так, в TDZ |

`const` забороняє **переприсвоєння змінної**, але не робить об'єкт незмінним (властивості всередині можна змінювати).

```js
const user = { name: 'Alex' };
user.name = 'Bob'; // ОК
user = {}; // TypeError
```

---

**Q: Що таке `undefined` і чим воно відрізняється від `null`?**

A: `undefined` — значення змінної, яка оголошена, але не ініціалізована (або результат звернення до неіснуючої властивості). `null` — навмисне "порожнє" значення, яке присвоює розробник. `typeof undefined === 'undefined'`, а `typeof null === 'object'` (історичний баг мови).

```js
let x;
console.log(x); // undefined

let y = null;
console.log(y); // null
```

---

**Q: Що таке `NaN` і як правильно перевірити значення на `NaN`?**

A: `NaN` (Not-a-Number) — спеціальне значення типу `Number`, результат некоректної математичної операції. Особливість: `NaN !== NaN`. Правильна перевірка — `Number.isNaN(x)`, а не глобальна `isNaN(x)` (вона спершу приводить аргумент до числа і може давати хибні спрацювання).

```js
console.log(NaN === NaN);        // false
console.log(Number.isNaN(NaN));  // true
console.log(isNaN('foo'));       // true (бо приводить до Number → NaN)
console.log(Number.isNaN('foo'));// false (не число, але й не NaN-приведення)
```

---

**Q: Що таке truthy та falsy значення?**

A: Falsy-значень усього 8: `false`, `0`, `-0`, `""`, `null`, `undefined`, `NaN`, `0n`. Все інше — truthy (включно з `"0"`, `"false"`, `[]`, `{}`).

```js
if ([]) console.log('truthy!'); // виведеться, бо [] — truthy
```

---

## Оператори та порівняння

**Q: Чим відрізняється `==` від `===`?**

A: `===` (strict equality) порівнює значення **без** приведення типів. `==` (loose equality) спершу приводить операнди до одного типу, потім порівнює. Рекомендація — завжди використовувати `===`, щоб уникнути неочевидної поведінки.

```js
console.log(1 == '1');   // true  — '1' приводиться до Number
console.log(1 === '1');  // false — різні типи
console.log(null == undefined);  // true
console.log(null === undefined); // false
```

---

**Q: Що виведе `[] + []`, `[] + {}` та `{} + []`?**

A: Оператор `+` для об'єктів приводить операнди до примітивів (через `toString()`):
```js
console.log([] + []); // "" (порожній рядок + порожній рядок)
console.log([] + {}); // "[object Object]"
console.log({} + []); // 0 (в консолі браузера) — {} інтерпретується як блок коду, а не вираз
```

---

**Q: Що таке нестрогий та строгий режими приведення типу в порівняннях `<`, `>`?**

A: Оператори порівняння завжди приводять операнди до примітивів/чисел (окрім порівняння двох рядків — там йде лексикографічне порівняння).

```js
console.log('2' > '10');  // true  — порівняння рядків посимвольно ('2' > '1')
console.log(2 > 10);      // false — числове порівняння
```

---

**Q: Як працює оператор `??` (nullish coalescing) і чим він відрізняється від `||`?**

A: `??` повертає праву частину, тільки якщо ліва — `null` або `undefined`. `||` повертає праву частину для будь-якого falsy-значення зліва (включно з `0`, `""`, `false`).

```js
const count = 0;
console.log(count || 10); // 10 — 0 вважається falsy
console.log(count ?? 10); // 0  — 0 не є null/undefined
```

---

**Q: Що таке optional chaining `?.`?**

A: Дозволяє безпечно звертатись до вкладених властивостей, повертаючи `undefined` замість помилки, якщо проміжна ланка — `null`/`undefined`.

```js
const user = { profile: null };
console.log(user.profile.age);   // TypeError
console.log(user.profile?.age);  // undefined
```

---

## Hoisting та Scope

**Q: Що таке hoisting (підняття)?**

A: Механізм, при якому оголошення змінних (`var`, `function`) та функцій "піднімаються" вгору області видимості під час фази компіляції, ще до виконання коду. Ініціалізація (присвоєння значення) при цьому **не** піднімається.

```js
console.log(a); // undefined (оголошення підняте, значення — ні)
var a = 5;

sayHi(); // "Hi!" — function declaration підіймається повністю
function sayHi() { console.log('Hi!'); }
```

---

**Q: Що таке Temporal Dead Zone (TDZ)?**

A: Проміжок між початком блоку та рядком, де оголошено змінну через `let`/`const`. У цій зоні звернення до змінної кидає `ReferenceError`, хоча технічно оголошення вже "підняте".

```js
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 5;
```

---

**Q: Які види scope існують в JavaScript?**

A:
- **Global scope** — доступний з будь-якого місця програми
- **Function scope** — змінні, оголошені у функції, доступні тільки всередині неї
- **Block scope** — змінні `let`/`const`, оголошені у `{ }`, недоступні за його межами
- **Lexical scope** — вкладена функція має доступ до змінних зовнішньої функції, виходячи з місця **написання** коду, а не виклику

```js
{
  let blockVar = 1;
  var funcVar = 2;
}
console.log(funcVar); // 2
console.log(blockVar); // ReferenceError
```

---

**Q: Класична задача — що виведе цей код і чому?**

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

A: Виведе `3 3 3`. `var` не має блокового scope — усі callback-и посилаються на **одну й ту саму** змінну `i`, яка на момент виконання `setTimeout` вже дорівнює 3. Якщо замінити `var` на `let`, виведе `0 1 2` — бо `let` створює нову прив'язку `i` на кожній ітерації циклу.

---

**Q: Що таке замикання (closure) простими словами?**

A: Функція "запам'ятовує" змінні з оточення, в якому вона була створена, навіть після того, як зовнішня функція завершила виконання.

```js
function counter() {
  let count = 0;
  return function () {
    count++;
    return count;
  };
}
const increment = counter();
console.log(increment()); // 1
console.log(increment()); // 2
```

---

## Функції

**Q: Чим відрізняється function declaration від function expression?**

A: Function declaration підіймається повністю (можна викликати до оголошення). Function expression підіймається лише як змінна (без значення), тому виклик до присвоєння призведе до помилки.

```js
sayHi(); // працює
function sayHi() { console.log('Hi'); }

sayBye(); // TypeError: sayBye is not a function
var sayBye = function () { console.log('Bye'); };
```

---

**Q: Чим стрілкова функція (arrow function) відрізняється від звичайної?**

A:
- Не має власного `this` — бере `this` з лексичного (зовнішнього) контексту
- Не має об'єкта `arguments`
- Не може використовуватись як конструктор (`new`)
- Не має власного `prototype`
- Не можна змінити `this` через `call`/`apply`/`bind`

```js
const obj = {
  name: 'Alex',
  regular: function () { console.log(this.name); }, // 'Alex'
  arrow: () => { console.log(this.name); }           // undefined (this з глобального scope)
};
```

---

**Q: Що таке параметри за замовчуванням (default parameters)?**

A: Дозволяють задати значення параметра, якщо аргумент не передано (або передано `undefined`).

```js
function greet(name = 'Guest') {
  console.log(`Hello, ${name}`);
}
greet();        // Hello, Guest
greet('Alex');  // Hello, Alex
```

---

**Q: Чим `arguments` відрізняється від rest-параметрів (`...args`)?**

A: `arguments` — псевдо-масив (array-like), доступний тільки у звичайних функціях, не має методів масиву (`map`, `filter`). `...args` — справжній масив, доступний і в стрілкових функціях.

```js
function sum(...args) {
  return args.reduce((a, b) => a + b, 0);
}
console.log(sum(1, 2, 3)); // 6
```

---

**Q: Що таке IIFE (Immediately Invoked Function Expression)?**

A: Функція, яка викликається одразу після оголошення. Використовується для створення ізольованого scope, щоб не забруднювати глобальний простір імен.

```js
(function () {
  const secret = 42;
  console.log(secret);
})();
```

---

**Q: Що таке `this` у звичайній функції і як воно визначається?**

A: `this` визначається способом **виклику** функції (а не місцем оголошення):
- Виклик як метод об'єкта: `obj.fn()` → `this === obj`
- Простий виклик: `fn()` → `this === undefined` (strict mode) або глобальний об'єкт
- `new fn()` → `this` — новий створюваний об'єкт
- `fn.call(ctx)`/`fn.apply(ctx)`/`fn.bind(ctx)` → `this === ctx`

---

## Масиви

**Q: Як створити масив і перевірити, що змінна — масив?**

A:
```js
const arr = [1, 2, 3];
const arr2 = new Array(1, 2, 3);

Array.isArray(arr); // true
typeof arr;          // 'object' — не дає відрізнити масив від об'єкта
```

---

**Q: Чим відрізняються `slice` та `splice`?**

A: `slice(start, end)` — **не змінює** оригінальний масив, повертає новий підмасив. `splice(start, deleteCount, ...items)` — **мутує** оригінальний масив: видаляє та/або вставляє елементи.

```js
const arr = [1, 2, 3, 4, 5];
console.log(arr.slice(1, 3)); // [2, 3], arr не змінився

arr.splice(1, 2, 'a', 'b');
console.log(arr); // [1, 'a', 'b', 4, 5] — arr змінено
```

---

**Q: Чим відрізняються `map`, `filter`, `forEach` і `reduce`?**

A:
- **`map`** — трансформує кожен елемент, повертає **новий масив** тієї ж довжини
- **`filter`** — повертає новий масив із елементами, що пройшли перевірку (predicate)
- **`forEach`** — просто ітерує масив, **нічого не повертає** (`undefined`)
- **`reduce`** — "згортає" масив в одне значення (число, об'єкт, масив тощо)

```js
const nums = [1, 2, 3, 4];
console.log(nums.map(n => n * 2));            // [2, 4, 6, 8]
console.log(nums.filter(n => n % 2 === 0));    // [2, 4]
console.log(nums.reduce((sum, n) => sum + n, 0)); // 10
```

---

**Q: Як додати/видалити елементи на початку/в кінці масиву?**

A:
| Метод | Дія | Змінює масив |
|---|---|---|
| `push(x)` | додає в кінець | так |
| `pop()` | видаляє останній | так |
| `unshift(x)` | додає на початок | так |
| `shift()` | видаляє перший | так |

---

**Q: Як перевірити, чи містить масив елемент?**

A:
```js
const arr = [1, 2, 3];
arr.includes(2);      // true — просто перевірка наявності
arr.indexOf(2);        // 1 — повертає індекс (-1, якщо немає)
arr.find(n => n > 1);  // 2 — повертає перший елемент, що пройшов умову
arr.some(n => n > 2);  // true — чи є хоч один елемент, що проходить умову
arr.every(n => n > 0); // true — чи всі елементи проходять умову
```

---

**Q: Як об'єднати/розгорнути масиви за допомогою spread-оператора?**

A:
```js
const a = [1, 2];
const b = [3, 4];
const merged = [...a, ...b]; // [1, 2, 3, 4]

const copy = [...a]; // поверхнева копія (shallow copy)
```

---

## Об'єкти та деструктуризація

**Q: Як отримати ключі, значення та пари [ключ, значення] об'єкта?**

A:
```js
const obj = { a: 1, b: 2 };
Object.keys(obj);    // ['a', 'b']
Object.values(obj);  // [1, 2]
Object.entries(obj);  // [['a', 1], ['b', 2]]
```

---

**Q: Що таке деструктуризація (destructuring)?**

A: Синтаксис для "розпаковування" значень з масивів або властивостей з об'єктів у окремі змінні.

```js
const { name, age = 18 } = { name: 'Alex' };
console.log(name, age); // Alex 18

const [first, second] = [10, 20];
console.log(first, second); // 10 20

// Перейменування та вкладена деструктуризація
const { profile: { city } } = { profile: { city: 'Kyiv' } };
console.log(city); // Kyiv
```

---

**Q: Як зробити поверхневу копію об'єкта (shallow copy)?**

A:
```js
const original = { a: 1, b: { c: 2 } };
const copy1 = { ...original };
const copy2 = Object.assign({}, original);

copy1.b.c = 99;
console.log(original.b.c); // 99 — вкладені об'єкти копіюються за посиланням!
```

---

**Q: Що таке шаблонні рядки (template literals)?**

A: Рядки в зворотних лапках (`` ` ``), що підтримують інтерполяцію змінних (`${}`) та багаторядковість без символів переносу.

```js
const name = 'Alex';
console.log(`Hello, ${name}! Сьогодні ${new Date().getFullYear()}`);
```

---

**Q: Як перевірити, чи має об'єкт власну (не успадковану) властивість?**

A:
```js
const obj = { a: 1 };
console.log(obj.hasOwnProperty('a'));      // true
console.log('toString' in obj);            // true (успадкована з прототипу)
console.log(obj.hasOwnProperty('toString')); // false
```

---

## DOM — основи

**Q: Що таке DOM?**

A: Document Object Model — деревоподібне представлення HTML-документа у вигляді об'єктів, з якими JavaScript може взаємодіяти (читати, змінювати, видаляти елементи).

---

**Q: Якими способами можна отримати елемент(и) зі сторінки?**

A:
| Метод | Повертає | Live/Static |
|---|---|---|
| `getElementById(id)` | один елемент | — |
| `getElementsByClassName(cls)` | HTMLCollection | live |
| `getElementsByTagName(tag)` | HTMLCollection | live |
| `querySelector(css)` | перший відповідний елемент | — |
| `querySelectorAll(css)` | NodeList всіх відповідних | static |

`querySelector`/`querySelectorAll` приймають будь-який CSS-селектор і зазвичай зручніші за старіші методи.

---

**Q: Чим відрізняються `innerHTML`, `textContent` та `innerText`?**

A:
- **`innerHTML`** — читає/записує HTML-розмітку всередині елемента (небезпечно з користувацьким вводом — ризик XSS)
- **`textContent`** — читає/записує весь текстовий вміст, включно з прихованими елементами, без парсингу HTML
- **`innerText`** — враховує CSS-стилі (не поверне текст `display: none` елементів), викликає reflow — повільніший

---

**Q: Як створити та додати новий елемент у DOM?**

A:
```js
const div = document.createElement('div');
div.textContent = 'Hello';
div.classList.add('box');
document.body.appendChild(div);
```

---

**Q: Як видалити елемент з DOM?**

A:
```js
element.remove();               // сучасний спосіб
element.parentNode.removeChild(element); // старий спосіб
```

---

## Події (Events)

**Q: Як додати обробник події елементу?**

A:
```js
button.addEventListener('click', function (event) {
  console.log('Клікнуто!', event.target);
});

// Видалення обробника (функція має бути іменована/збережена в змінній)
button.removeEventListener('click', handler);
```

---

**Q: Що таке спливання подій (event bubbling)?**

A: Подія, що виникла на дочірньому елементі, послідовно "спливає" вгору до батьківських елементів (від найглибшого до `document`). Протилежний механізм — **capturing** (занурення від `document` до цільового елемента), вмикається третім аргументом `addEventListener(type, fn, true)`.

```html
<div id="parent"><button id="child">Click</button></div>
```
```js
parent.addEventListener('click', () => console.log('parent'));
child.addEventListener('click', () => console.log('child'));
// Клік по button виведе: "child", потім "parent"
```

---

**Q: Чим відрізняються `event.preventDefault()` і `event.stopPropagation()`?**

A: `preventDefault()` скасовує **стандартну поведінку браузера** (наприклад, перехід за посиланням, відправку форми). `stopPropagation()` зупиняє **подальше спливання/занурення** події до інших елементів — вони не пов'язані одне з одним.

---

**Q: Що таке делегування подій (event delegation)?**

A: Техніка, коли замість навішування обробника на кожен дочірній елемент, обробник вішається на **спільного батька**, а конкретний елемент визначається через `event.target`. Ефективно для динамічних списків.

```js
list.addEventListener('click', (e) => {
  if (e.target.tagName === 'LI') {
    console.log('Клікнуто по:', e.target.textContent);
  }
});
```

---

## JSON та інше

**Q: Що таке JSON і як з ним працювати в JS?**

A: JSON (JavaScript Object Notation) — текстовий формат обміну даними. `JSON.stringify(obj)` перетворює JS-об'єкт у рядок, `JSON.parse(str)` — рядок назад у об'єкт.

```js
const obj = { name: 'Alex', age: 25 };
const str = JSON.stringify(obj); // '{"name":"Alex","age":25}'
const parsed = JSON.parse(str);  // { name: 'Alex', age: 25 }
```

Важливо: `JSON.stringify` пропускає функції, `undefined` та `Symbol`, не підтримує циклічні посилання.

---

**Q: Чим відрізняються `setTimeout` та `setInterval`?**

A: `setTimeout(fn, delay)` виконує функцію **один раз** через вказаний час. `setInterval(fn, delay)` виконує функцію **повторно** з інтервалом, доки не викликано `clearInterval`.

```js
const id = setInterval(() => console.log('tick'), 1000);
setTimeout(() => clearInterval(id), 5000); // зупинити через 5с
```

---

**Q: Що таке `strict mode` (`'use strict'`)?**

A: Директива, яка вмикає суворіший режим виконання JS: забороняє неявне оголошення глобальних змінних, робить помилки видимими (замість мовчазного ігнорування), забороняє дублювання параметрів функції тощо.

```js
'use strict';
x = 5; // ReferenceError: x is not defined (без 'use strict' створило б глобальну змінну)
```

---

**Q: У чому різниця між компіляцією та інтерпретацією стосовно JS?**

A: JavaScript традиційно вважають інтерпретованою мовою, але сучасні рушії (наприклад, V8) використовують **JIT-компіляцію** (Just-In-Time) — код спершу парситься та частково компілюється в машинний код перед виконанням, для підвищення продуктивності.

---

*Файл оновлюється по мірі надходження нових матеріалів.*
