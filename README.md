# Шпаргалка по паттернам в TypeScript

Как типы выбирают и извлекают
- [Conditional types](#conditional-types)
- [infer](#infer)
- [Object values](#objectvalues--key-access)
- [Tuple to union](#tuple-to-union)

Как типы преобразуют структуру
- [Distributive conditional types](#distributive-conditional-types)
- [Union-to-intersection](#union-to-intersection)
- [Intersection-to-union](#intersection-to-union)
- [Mapped types и keyof](#mapped-types)
- [Key remapping](#key-remapping)
- [template/string literal types](#template-literal-types)
- [Recursion](#recursion)
- [Simplify](#simplify)

Как типы описывают протокол использования API
- [variadic tuples](#variadic-tuples)
- [branded types](#branded-types)
- [state machine / typed builder](#builder--state-machine)
- [path generation](#path-generation)

Математика на TypeScript
- [Как представить число](#numbers)
- [Как складывать и вычитать](#math-operations)
- [Как сравнивать числа](#compare-numbers)


## Глоссарий
Иногда я в тексте буду использовать либо английскую версию, либо русскую - как пойдет.
Поэтому важно понимать, что
- union - юнион - объединение типов - синтаксис `string | number` - это когда ИЛИ тот тип, ИЛИ другой
- intersection - пересечение типов - синтаксис `{ id: string } & { total: number }`
- tuple - кортеж - конечный массив определенных значений по определенным индексам

---

## Conditional types

Условные типы работают как обычный `if`, чтобы выбрать один из двух вариантов по условию.
Это базовый паттерн для написания сложных типов в TypeScript: мы программируем условие "если тип похож на такой, то вернуть одно, иначе вернуть другое".
На этом паттерне строится буквально всё - в программировании без if мы бы до сих пор хз чо делали.

### Пример использования Conditional types

```typescript
type TIsString<TValue> = TValue extends string ? true : false;

type TRequestConfig<TMethod extends "GET" | "POST"> =
  TMethod extends "GET"
    ? { query?: Record<string, string> }
    : { body: unknown };

type TExample1 = TIsString<"hello">;
type TExample2 = TIsString<42>;

type TGetConfig = TRequestConfig<"GET">;
type TPostConfig = TRequestConfig<"POST">;
```

### Где применим паттерн Conditional types

- Когда тип результата зависит от входного типа
- Когда нужно разрешить или запретить часть API в зависимости от конфигурации - можно возвращать более широкий или узкий интерфейс
- Когда получаемый интерфейс зависит от режима работы, например `GET`-запрос и `POST`-запрос имеют разный конфиг
- Когда нужно типизировать что-то с несколькими ветками поведения

### Объяснение паттерна Conditional types

Это, по сути, тернарный оператор: если левый тип совместим с правым, берется ветка после `?`, если нет, то ветка после `:`.
Отличие от тернарника и условий в JS в том, что TypeScript умеет сравнивать именно типы - один тип с другим через `extends`.
Extends - это не совсем равенство в нашем понимании. Тут вам стоит посмотреть на иерархию базовых типов в TS.
Круто, что это происходит не в рантайме, а во время вычисления типа - рантайм еще не родился, а типы уже ветвятся.

### Лайфхак
Мякотка, которая помогает и вам, и нейросетям. Наверняка вы встречали ошибку "Type '...' is not assignable to type 'never'". 
Попробуй пойми, откуда там never, особенно если много вложенных тернарников...
В случае тупиковых ситуаций описывайте не never, а нормальное сообщение об ошибке.
```typescript
type TExampleIf<TCondition, TData> = TCondition extends true ? { data: TData } : "Invalid condition";

type TResult = TExampleIf<false, string>;
const result: TResult = { data: 'hello, data!' }; // Error: Invalid condition - теперь все понятно
```
Но! Это меняет модель типа. Вместо невозможного состояния(never) теперь валидная строка, и в каких-то случаях она может утечь дальше по реализации типов.
Поэтому иногда такой лайфхак удобно применять при разработке сложных типов, но после того, как типы отлажены - возможно стоит вернуться к never.


## infer

`infer` позволяет вытащить __часть__ типа, сохранить ее в переменную и вернуть.
Проще всего думать о нем как о маске: мы описываем форму, которую хотим распознать, и если тип в эту форму попадает, TS достает нужный кусок наружу.

### Пример использования infer

```typescript
type TReturnTypeOf<TFunction> =
  TFunction extends (...args: any[]) => infer TResult
    ? TResult
    : never;

type TPayloadOf<THandler> =
  THandler extends (payload: infer TPayload) => void
    ? TPayload
    : never;

type TExampleHandler = (payload: { id: string; total: number }) => void;

type TExampleReturn = TReturnTypeOf<() => Promise<number>>;
type TExamplePayload = TPayloadOf<TExampleHandler>;

type TPromiseOf<TPromiseGeneric> = TPromiseGeneric extends Promise<infer TPromiseResult> ? TPromiseResult : "Not promise passed";
type TExamplePromise = TPromiseOf<Promise<number>>; // number
```

### Где применим паттерн infer

- Когда нужно достать тип результата функции, аргументов функции или payload обработчика
- Когда нужно извлечь внутреннее значение из `Promise`, массива, колбэка или обертки. В общем, из любого дженерика
- Когда вы пишете middleware, decorators или wrapper-функции и хотите не потерять сигнатуру
- Когда тип должен автоматически выводиться из структуры, а не задаваться вручную - infer тут сработает для получения начальной "картины"

### Объяснение паттерна infer

Важная штука: `infer` работает только внутри conditional types, которые описаны выше.
TypeScript пытается сопоставить ваш тип с шаблоном слева от `?`. Если шаблон подходит, все места, где стоит `infer`, заполняются значениями типов из исходного типа.
Шаблон - это валидное выражение на типах TS - тип, дженерик или функция.
Разберем такой пример:
```typescript
type TReturnTypeOf<TFunction> =
  TFunction extends (...args: any[]) => infer TResult
    ? TResult
    : never;

type TExampleHandler1 = (payload: { id: string; total: number }) => void;
type TExampleHandler2 = () => Promise<number>;

type TExampleReturn1 = TReturnTypeOf<TExampleHandler1>; // void
type TExampleReturn2 = TReturnTypeOf<TExampleHandler2>; // Promise<number>
```
Здесь происходит следующее:
- TExampleHandler попадает в TReturnTypeOf и TS начинает сравнивать:
- похоже вообще переданное значение на функцию? Похоже, идем дальше
- похожи аргументы: (payload: { id: string; total: number }) на шаблон (...args: any[])? Похожи, круто, идем дальше
- похожестей не осталось, остался infer. Что там в переданном типе на месте infer? А там void. Ок, вот его в infer и положим.
- срабатывает тернарник, похожести соблюдены, в TResult уже лежит значение, возвращаем его

## Object.values & key access

Иногда бывает так, что у вас есть четко описанный тип глубокого объекта, или вы построили тип с помощью маппинга, как-то преобразовали его значения, и теперь вам нужны только значения, а не ключи.
Здесь поможет аналог Object.values из JS

### Пример использования Object.values
```typescript
type TApi = {
    users: {
        method: "GET";
        path: "/users";
    };
    createOrder: {
        method: "POST";
        path: "/orders";
    };
    getOrderById: {
        method: "GET";
        path: "/orders/:id";
    };
};

type TEndpoint = TApi[keyof TApi];

type TEndpointPath = TApi[keyof TApi]["path"]; // все доступные пути
type TEndpointMethod = TApi[keyof TApi]["method"]; // все доступные методы
```
или
```typescript
// Разные значения могут повляться тут - в источнике истины
type TRolePermissions = {
  admin: "read" | "write" | "delete";
  manager: "read" | "write";
  guest: "read";
};

// А вот этот тип извлечет все из источника истины
type TPermission = TRolePermissions[keyof TRolePermissions]; // read | write | delete
```

### Где применим паттерн Object.values
- Когда у вас есть набор значений, и нужно получить union всех значений
- Когда вы описали API, события, permissions или handlers в виде объекта, а дальше хотите работать только со значениями
- Когда объект нужен как удобный способ объявления, а дальнейшая логика строится уже на union значений
- Когда из объекта-источника-истины вы строите например аргументы для методов

### Объяснение паттерна Object.values
Простой `keyof TSomeObject` возвращает union всех ключей объекта, например "users" | "createOrder" | "getOrderById".
Когда мы пишем `TSomeObject[keyof TSomeObject]`, TypeScript берет объект и говорит: "дай мне тип значения по каждому из этих ключей".
Поскольку ключей несколько, результатом становится union всех значений объекта. Поэтому этот паттерн и похож на Object.values: мы как будто выбрасываем ключи и оставляем только содержимое. А дальше этот union уже удобно фильтровать, преобразовывать и использовать в других паттернах.

Если в вашей структуре известны ключи заранее - вы можете копать вглубь таким образом.
В TS, как и в JS есть способ добраться вглубь объекта по ключам. Только не через точку, а через квадратные скобки:
```typescript
TApi['users']['method'] // 'GET'
```
Так же это копание вглубь полезно, если вы хотите чтобы какой-то тип проходил сквозь всю систему.
```typescript
// КОГДА ТИП НЕ ПРОХОДИТ НАСКВОЗЬ
type TUser = {
  // Если тут сменим number на string - развалится функция getUserById, и надо там будет править руками
  id: number;
}

function getUserById(userId: number): TUser {}

// КОГДА ТИП ПРОХОДИТ НАСКВОЗЬ
// теперь сама функция не развалится, а развалятся конкретные её вызовы в коде - минус одна ручная правка
function getUserById(userId: TUser['id']): TUser {}
```

## Tuple to union

Кортеж - это как массив, но конечный и со строгой длиной.
Иногда набор значений удобнее сначала описать как кортеж(tuple): можно использовать его в рантайме и получать точные литеральные типы.
Но в TS часто нужен не сам кортеж, а union всех его элементов. Для этого в TypeScript есть очень короткий, но полезный прием: TTuple[number].
Этот паттерн поможет превратить кортеж в union.

### Пример использования tuple to union
```typescript
// Вот так тип ROLES будет Array<string>
// const ROLES = ["admin", "user", "guest"];
// А вот так четкий тип readonly ["admin", "user", "guest"]
const ROLES = ["admin", "user", "guest"] as const;
type TRoles = typeof ROLES; // readonly ["admin", "user", "guest"]

type TRole = TRoles[number];

declare function isFeatureAvailableForRole(role: TRole): boolean;
```

### Где применим паттерн Tuple to union

- Когда набор допустимых строк или чисел удобно сначала объявить как tuple, а потом использовать как union
- Когда одни и те же значения нужны и в рантайме, и в типах
- Когда вы описываете роли, статусы, фича-флаги, события или список поддерживаемых команд
- Когда нужно получить строгий union из фиксированного списка значений без ручного перечисления через |

### Объяснение паттерна Tuple to union
Во-первых, видим новое слово - typeof. Эта штука позволяет из рантаймового значения получить тип. Удобно, чтобы не дублировать. Т.е. рантаймовое значение первично.
Во-вторых, кортежи. Кортежи - это частный случай массива, у которого известны точные элементы по индексам.
Когда мы пишем TRoles[number], мы говорим TS: "дай мне тип значения, которое лежит в этом tuple по любому числовому индексу".
Поскольку в tuple по разным индексам лежат разные литералы, TypeScript собирает их в union: "admin" | "user" | "guest".
Поэтому этот паттерн удобен как мост между списком значений и union-типом, с которым уже можно работать дальше.

---

## Distributive conditional types

Дистрибутивность в TS применяют условие к каждому элементу union по отдельности. Казалось бы, тот же COnditional type, но с черной магией для юнионов.
Это очень мощный механизм, потому что позволяет фильтровать union, преобразовывать его или разбирать на группы. Если хотите фильтровать объединения типов - вам точно сюда!

### Пример использования Distributive conditional types

```typescript
type TOnlyStrings<TValue> = TValue extends string ? TValue : never;

type TWrapToArray<TValue> = TValue extends any ? TValue[] : never;

type TExampleUnion = string | number | boolean;

type TExampleStrings = TOnlyStrings<TExampleUnion>;
type TExampleArrays = TWrapToArray<TExampleUnion>;
```

### Где применим паттерн Distributive conditional types

- Когда нужно оставить в union только нужные варианты, например только строки
- Когда нужно превратить каждый вариант union в новую форму
- Когда вы строите типизированный event bus, router или registry и хотите отсекать некоторые события/пути/итп

### Объяснение паттерна Distributive conditional types

Если слева от `extends` стоит "голый" юнион, TypeScript запускает условие отдельно для каждого его элемента.
Важно объяснить слово "голый". Голый - это когда просто `string | number` - просто юнион. Не-голый будет, когда юнион "одет"/"обернут" в функцию - `(arg: string | number) => void`.
Читай "голый" = "дистрибутивность". Т.е. когда слышишь "дистрибутивность" - думай о голых union. (хотел пошутить, но слишком пошло будет)
Не-голый юнион пригодится нам в [Union to intersection](#union-to-intersection), о нем ниже, и картинка в голове сложится.

Голый юнион - `string | number` - не проверяется как один большой тип, а раскладывается на две проверки: сначала для `string`, потом для `number`.
После этого результаты снова собираются обрано в юнион, а всё, что имеет значение never - отсекается. Именно так можно фильтровать юнионы.

### Лайфхак
Фильтрация union через тернарник иногда зашумляет код - появляются лишние ветки условий, которые путают.
Вместо этого можно использовать элегантное отсечение, если вы точно знаете какие типы входят в union. ТОЛЬКО если вы знаете какие типы входят в union.
Например, keyof T возвращает поля объекта, если его сигнатура известа. А в общем случае объектов keyof T вернет number | string | symbol В сложных типах, особенно рекурсивных, символ очень гадит, и его хочется отсекать.

```typescript
declare const UserBrand: unique symbol;

type TUser = {
    id: string;
    name: string;
    [UserBrand]: 'userBrand'
}
```
Пример очень синтетический и решается другими средствами, но для примера оставлю. Предположим, надо отсечь от типа поля, ключ которых - символ.

```typescript
type TReplaceSymbolToNever<Obj> = {
  [K in keyof Obj]: K extends string | number ? Obj[K] : never; // вот чертов тернарник, который портит всю жизнь просто потому, что keyof подсовывает нам символ
}

type TNonNeverKeys<Obj> = {
    [K in keyof Obj]: Obj[K] extends never ? never : K // лайфхак, мапим не на значение, а на ключ
}[keyof Obj]; // и достаем значения

type TUserValidKeys = TNonNeverKeys<TReplaceSymbolToNever<TUser>>; // ключи объекта User, которые не Symbol

type TUserWithoutSymbol = {
  [K in TUserValidKeys]: TUser[K]; // пройдемся только по валидным ключам и возьмем их значения (повезет, если тут не возникнет у TS вопроса "а точно ли K является ключом TUser?")
}
```
Раньше так и делали, во времена древних версий тайпскрипта.
Чтобы избавиться от чертова тернарника можно просто сразу обрезать символ от keyof 
```typescript
type TOnlyNonSymbolKeys<Keys> = Keys extends string | number ? Keys : never;

type TUserWithoutSymbol = {
  [K in TOnlyNonSymbolKeys<keyof TUser>]: TUser[K];
}
```
Всё ещё много писанины. Прибегаем к ninja coding!
```typescript
type TUserWithoutSymbol = {
  [K in keyof TUser & (string | number)]: TUser[K];
}
```
Если мы применим intersection одного юниона к другому, то исключится всё лишнее.
```typescript
type TSomeType = (string | object) & (object | string | number) // string | object
```

## Union-to-intersection

Это один из самых интересных хаков в TypeScript.
Другое название этого паттерна - "избавиться от дистрибутивности". Когда понимаете, что надо работать с целым юнионом и весь его собрать в одно - вам сюда!
`Union-to-intersection` сам по себе превращает union в пересечение.
На первый взгляд это выглядит странно, но на практике очень полезно, когда у вас есть набор вариантов и из них нужно собрать один объединенный тип.

### Пример использования Union-to-intersection

```typescript
type TUnionToIntersection<TUnion> =
  (
    TUnion extends any
      ? (value: TUnion) => void
      : never
  ) extends (value: infer TResult) => void
    ? TResult
    : never;

type TExampleUnion =
  | { id: string }
  | { name: string }
  | { active: boolean };

type TExampleIntersection = TUnionToIntersection<TExampleUnion>;
```

### Где применим паттерн Union-to-intersection

- Когда нужно собрать тип из набора отдельных частей
- Когда вы объединяете плагины, middleware или feature-модули в один контракт
- Когда что-то сначала задается как union вариантов, а потом нужен итоговый собранный объект
- Когда вы строите сложные builder-цепочки и аккумулируете состояние через пересечения

### Объяснение паттерна Union-to-intersection

Этот прием работает через "одевание" голого юниона и особенности TS при работе с аргументами функции.
Сперва у нас дистрибутивный(голый) юнион `A | B | C`. Сперва мы его одеваем в функцию, и получаем `(value: A) => void | (value: B) => void | (value: C) => void`. 
Дальше TypeScript пытается понять, какой один тип аргумента подойдет для такой функции в целом, и получает пересечение `A & B & C`.
Это связано с тем, как проверяются типы аргументов функций - когда в позиции аргумента стоит объединение(union) функций - TS должен вывести такой тип, который безопасно передать в любую из этих функций.
В сложном мире это называется "контрвариантность", но про эти штуки мы тут говорить не будем.
Выглядит капец непрозрачно, но идея у него простая: через функцию мы заставляем TypeScript "схлопнуть" варианты в одну общую форму.
Подробнее можно почитать [тут](https://dev.to/anuraghazra/explain-like-im-five-typescript-uniontointersection-type-5ako)

## Intersection-to-union

Казалось бы, union в intersection превращать умеем, надо бы уметь и наоборот. Но тут "фигвам". Обратно этот фарш не проворачивается в мясо.

У TS есть встроенная дистрибутивность - о ней были два предыдущих паттерна. Т.е. если у вас УЖЕ есть откуда-то взявшиеся union-ы - вы можете вертеть ими как хотите.
Но если вы их навертели в intersection - TS считает это единым типом, и он не обязан и не хранит "как так получилось". Все, котлетка слеплена.
Поэтому, если хотите всетаки сохранить исходный union - сохраняйте его отдельно(чаще с помощью паттерна builder или вообще не трогайте исходный тип).

Парочка примеров:
```typescript
// главный тип
type TAction =
  | { type: 'a'; value: string }
  | { type: 'b'; value: number };

type TDispatcher<U> = (action: U) => void;
type THandlers<U extends { type: PropertyKey }> = {
  [K in U['type']]: (action: Extract<U, { type: K }>) => void;
};

type H = THandlers<TAction>;
/*
{
  a: (action: { type: 'a'; value: string }) => void;
  b: (action: { type: 'b'; value: number }) => void;
}
*/
```
Здесь у нас TAction - главный исходный тип, от которого мы строим все остальные типы.

Либо вы можете воспользоваться `Pick`/`Omit`/`Exclude`, чтобы извлечь/отсечь нужные куски типа, но это не сохранит начальные условия его появления.

## Mapped types

Mapped types позволяет пройтись по всем ключам объекта и массово преобразовать его свойства. 
Здесь у нас появляются способы извлекать ключи объекта и итераторы - мы уже знали `if` на типах, теперь узнаем что-то типа `map`.
Это способ сказать TypeScript: "возьми ту же структуру ключей, но поменяй значения по определенному правилу".
Такой паттерн удобен, когда нужно строить DTO, readonly-версии, nullable-версии и любые массовые трансформации значений модели.

### Пример использования Mapped types

```typescript
type TNullable<TObject> = {
  [K in keyof TObject]: TObject[K] | null;
};

type TApiUser = {
  id: string;
  name: string;
  isAdmin: boolean;
};

type TNullableApiUser = TNullable<TApiUser>;
```

### Где применим паттерн Mapped types

- Когда нужно получить nullable-, readonly-, partial- или required-версию объекта
- Когда вы строите разные представления одной и той же доменной модели
- Когда нужно автоматически преобразовать схему API в схему формы, таблицы или фильтра
- Когда вы делаете кодогенерацию типов поверх известных ключей

### Объяснение паттерна Mapped types

Ключевые моменты:
- `keyof TObject` дает union всех ключей объекта: `keyof { foo: number; bar: string} === 'foo' | 'bar'`.
- `[K in keyof TObject]` - это итератор, он пройдется по каждому ключу, и для каждого ключа TS создает новое значение - то, что после двоеточия - и положит в результирующий тип пару ключ-новое_значение.
- `TObject[K]` - доступ к значению по ключу во время итерации
Благодаря этому не нужно руками переписывать каждое поле(мы же тут про генерацию). Мы просто задаем правило, а TypeScript применяет его ко всей структуре.

## Key remapping

Key remapping расширяет mapped types и позволяет не только менять значения, но и переименовывать или удалять ключи. 
Это полезно, когда из внутренней схемы нужно получить внешний API с другими именами методов или когда часть полей надо отфильтровать.
Добавлю, что прикручивание сюда template literal types позволяет творить невероятную мощь - всякие ToCamelCase, ToSnakeCase итп.

### Пример использования Key remapping

```typescript
// (фильтрация) отсекаем поля, начинающиеся с _
type TPublicOnly<TObject> = {
  [TKey in keyof TObject as TKey extends `_${string}` ? never : TKey]: TObject[TKey];
};

// (простое переименование) изменяем названия ключей, добавляя префикс api_ - работает только если ключи - строки
type TPrefixedApi<TKnownObject> = {
  [TKey in keyof TKnownObject as `api_${TKey}`]: TKnownObject[TKey];
};
// Хорошая практика - всетаки проверять ключи на то, что они строки, и только после этого их преобразовывать
type TPrefixedApiOnlyStrings<TAnyObject> = {
    [TKey in keyof TAnyObject as TKey extends string ? `api_${TKey}` : never]: TAnyObject[TKey];
};

type TInternalConfig = {
  _token: string;
  baseUrl: string;
  timeout: number;
};

type TPublicConfig = TPublicOnly<TInternalConfig>;
type TPrefixedConfig = TPrefixedApi<TPublicConfig>;
```

### Где применим паттерн Key remapping

- Когда нужно убрать служебные поля
- Когда из ключей объекта нужно сгенерировать новый объект, с новыми именами
- Когда из ключей-значений надо создать объект с ключами-методами
- Когда требуется добавить префиксы или суффиксы к ключам для DSL или SDK
- Когда внутреннюю модель нужно преобразовать в более удобный публичный контракт

### Объяснение паттерна Key remapping
Обычный mapped type оставляет имя ключа таким же, каким оно было. Key remapping добавляет часть `as`, в которой можно вычислить новое имя.
Если в `as` вернуть `never`, TypeScript просто не создаст такой ключ в итоговом объекте.
Если вернуть строковый шаблон(о них далее) - ключ будет переименован.
Поэтому этот паттерн удобен сразу для двух задач: фильтрации и генерации нового API поверх старой структуры.

Ключевые моменты этого паттерна:
- просто `as` - переименование ключа
- `as ... extends ? ... : never ` - фильтрация и/или переименование ключа (потому что never отсекается)

Обязательно надо учитывать, что это только типы - если вы переименовываете ключи - ваш рантайм должен тоже это делать, и там надо обмазаться защитами.
Т.е. вы в рантайме по тем же правилам, по которым сформировали тип с новыми ключами - должны сформировать новый объект с этими же ключами.
Если что-то пойдет не так и рантайм не сгенерирует какой-нибудь ключ - получите например ошибку undefined is not a function - так себе удовольствие это дебажить.
Чтобы этого избежать и проще дебажить - генерацию в рантайте можно сделать на Proxy, чтобы ваш get или apply в хэндлере понимали, что вот такой ключ получен в процессе генерации, и почему-то значения нет.
Тогда можно выплюнуть более понятную ошибку.
Запомните: трансформация типов не делает runtime-работу за вас - runtime-реализация должна повторять те же правила, и внятные ошибки в консоль/сентри помогут быстрее ловить ошибки, если реализации разошлись.

### Лайфхак
Не обязательно использовать never в конструкции `as ... extends` - можно использовать разные переименования. Например:
```typescript
type TApiConstructor<TObject> = {
  [TKey in keyof TObject as TKey extends `_${string}` ? `UNSTABLE${TKey}` : TKey]: TObject[TKey];
};

type TMethodWithResult = {
    m1: void;
    _m2: string;
}

type TApi = TApiConstructor<TMethodWithResult>; // { m1: void, UNSTABLE_m2: string }
```

## Template literal types

Они же строковые литералы.
Template literal types позволяют разбирать и строить строки.
Эта возможность полезна там, где строки на самом деле несут значение и структуру: роуты, события, путь к полю(obj.a.b.c).
Используя строковые литералы TypeScript может проверять не просто "это строка", а "это строка нужного формата".

ВАЖНО! Как и в JS для шаблонных строк - в TypeScript для строковых литералов надо использовать обратные кавычки.

### Пример использования Template literal types

```typescript
type TRouteParams<TRoute extends string> =
  TRoute extends `${string}:${infer TParam}/${infer TRest}`
    ? { [TKey in TParam | keyof TRouteParams<`/${TRest}`>]: string }
    : TRoute extends `${string}:${infer TParam}`
      ? { [TKey in TParam]: string }
      : {};

type TEventName<TDomain extends string, TAction extends string> =
  `${TDomain}.${TAction}`;

type TExampleRoute = "/orders/:orderId/items/:itemId";
type TExampleParams = TRouteParams<TExampleRoute>;

type TUserCreatedEvent = TEventName<"user", "created">;
```

### Где применим паттерн Template literal types

- Когда нужно извлекать параметры из роуты
- Когда вы строите события, DSL-команды или ключи с понятным шаблоном
- Когда хотите валидировать строковые пути к полям объекта
- Когда нужно генерировать имена методов или событий из частей строки

### Объяснение паттерна Template literal types

TS умеет смотреть на строковый литерал как на шаблон, а не как на сплошной текст.
Если вы пишете строку вида `${infer TLeft}.${infer TRight}`, TypeScript пытается распарсить исходную строку по этому шаблону и положить найденные части в `TLeft` и `TRight`.
В результате строки превращаются в маленький DSL, который можно разбирать, анализировать и собирать обратно.

## Recursion

Как бы объяснить рекурсию без рекурсии...
Рекурсивные типы позволяют типу вызывать самого себя на вложенных частях структуры.
Это нужно, когда объект может быть глубоким, а преобразование должно применяться не только к верхнему уровню, но и ко всем вложенным полям. Такой паттерн лежит в основе например `DeepPartial`, `DeepReadonly`.
Или когда надо глубокий тип сплющить.

С этого момента начинается композиция всех вышеидущих паттернов

### Пример использования рекурсии в типах

Довольно скучная рекурсия и просто учебный пример (в реально жизни Date это тоже object, и может случиться всякое)
```typescript
type TDeepPartial<TValue> =
  TValue extends object
    ? {
        [TKey in keyof TValue]?: TDeepPartial<TValue[TKey]>;
      }
    : TValue;

type TOrder = {
  id: string;
  customer: {
    name: string;
    contact: {
      email: string;
      phone: string;
    };
  };
  items: Array<{
    sku: string;
    quantity: number;
  }>;
};

type TPartialOrder = TDeepPartial<TOrder>;
```
или веселая рекурсия
```typescript
// Сплющивание без аккумулятора

type Primitive = string | number | boolean | bigint | symbol | null | undefined;

type TPaths<T> =
    T extends Primitive
        ? never // примитивы вообще не обрабатываем
        : {
            [K in keyof T & (string | number)]: // mapped types с отсеченным символом
            T[K] extends Primitive // проверка на то, что дальше не объект
                ? `${K}` // если дальше не объект - возвращаем ключ
                : T[K] extends readonly (infer U)[] // проверка на массив
                    ? `${K}` | `${K}.${number}` | `${K}.${number}.${TPaths<U>}` // формируем ключи для массива
                    : `${K}` | `${K}.${TPaths<T[K]>}` // формируем ключи для объекта
        }[keyof T & (string | number)]; // достаем значения по ключам с отсеченным символом
```
Ключевая идея рекурсии здесь - сформировать объект следующего вида
```typescript
type TIntermediate = {
  id: 'id';
  customer: 'customer' | ('customer.name' | 'customer.contact') /* TPaths<TOrder['customer]> */ | ('customer.contact.email' | 'customer.contact.phone') /* TPaths<TOrder['customer']['contact']> */;
  items: 'items' | ('items.0' | 'items.1' | 'items.2' | '...etc...') /* TPaths<TOrder['items']> */ | ('items.0.sku' | 'items.0.quantity') /* TPaths<TOrder['items'][number]> */;
}
```
и взять все его значения.
Попробуйте в уме или на бумажке отладить такой тип и прочувствовать, как он построен.
Вообще, пример с путями в объекте я вынес в отдельный паттерн.

### Где применим паттерн Recursion

- Когда нужно строить Deep-типы
- Когда требуется обходить вложенные конфиги, схемы или DTO
- Когда вы генерируете строковые пути до вложенных полей
- Когда бизнес-модель глубокая, а правило должно работать на всех уровнях
- Когда глубокий тип надо разложить в плоский

### Объяснение паттерна Recursion

TypeScript позволяет типу ссылаться на самого себя. Обычно это выглядит так: если текущее значение внутри логики типа еще подходит под обработку - применяем ту же самую логику к текущему значению.
Главное здесь наличие базового случая, иначе TypeScript не поймет, где нужно остановиться.

Будьте аккуратны! Глубокие рекурсии могут заметно тормозить.

## Simplify

`Simplify` не меняет смысл типа, но делает его более удобным для чтения в IDE.

### Пример использования Simplify

```typescript
type TSimplify<TValue> = {
  [TKey in keyof TValue]: TValue[TKey];
} & {};

type TBaseConfig = {
  baseUrl: string;
};

type TAuthConfig = {
  token: string;
};

type TRetryConfig = {
  retries: number;
};

type TRawConfig = TBaseConfig & TAuthConfig & TRetryConfig;
type TSimplifiedConfig = TSimplify<TRawConfig>;
```

### Где применим паттерн Simplify

- Когда IDE показывает слишком сложный результат после пересечений и mapped types
- Когда вы хотите улучшить DX ваших типов
- Когда промежуточный тип вычисляется корректно, но плохо читается человеком
- Когда builder или API-клиент постепенно накапливает много частей в один итоговый объект

### Объяснение паттерна Simplify

После сложных преобразований TypeScript часто хранит тип как композицию нескольких частей, а не как обычный "плоский" объект. 
`TSimplify` берет все ключи итогового типа и заново создает новый объект с теми же значениями. Смысл типа не меняется, но представление становится проще.
Это не магия и не новый алгоритм, а скорее способ попросить TypeScript показать уже вычисленный результат в более удобной форме.
Без `& {}` в некоторых версиях IDE результат всё равно показывает исходную форму.

---

## Variadic tuples

Этот паттерн позволяет работать с кортежами и аргументами функций так, как будто это конструктор из частей.
Так можно отрезать хвост или голову кортежу, добавлять аргументы, сохранять сигнатуры wrapper-функций.
Этот паттерн особенно полезен там, где обычный `any[]` разрушает всю точность типов.

### Пример использования Variadic tuples

```typescript
type TTail<TItems extends unknown[]> =
  TItems extends [unknown, ...infer TRest]
    ? TRest
    : [];

type TPrepend<TItem, TItems extends unknown[]> = [TItem, ...TItems];

type TExampleArgs = [url: string, retries: number, enabled: boolean];

type TExampleTail = TTail<TExampleArgs>;
type TExamplePrepended = TPrepend<"GET" | "POST", TExampleArgs>;

type TAnyFunction = (...args: any[]) => any;

function withLogging<TFunction extends TAnyFunction>(fn: TFunction) {
  return (...args: Parameters<TFunction>): ReturnType<TFunction> => {
    console.log("call args", args);
    return fn(...args);
  };
}

const tWrappedRequest = withLogging((url: string, retries: number) => {
  return { url, retries };
});
```

### Где применим паттерн Variadic tuples

- Когда нужно писать middleware, decorators и wrapper-функции без потери аргументов
- Когда вы делаете `pipe`, `compose`, `curry` или цепочку вызовов
- Когда нужно добавлять или убирать аргументы у функций на этапе работы с типами

### Объяснение паттерна Variadic tuples

Кортеж в TypeScript может хранить не просто "массив чего-то", а точную последовательность элементов.
И как и в JS - у TS есть понятие rest(собрать) и spread(разобрать) оператор.
Variadic tuples позволяют в шаблоне сказать: "вот один элемент, а дальше идет произвольный хвост". Когда TypeScript сопоставляет tuple с таким шаблоном, он умеет отдельно вынуть хвост через спред-оператор и `infer`.
Поэтому, например, аргументы функции можно разбирать и собирать так же, как обычную структуру. Это и дает возможность писать обертки, которые не ломают сигнатуру.

#### Лайфхак
Со строками паттерн работает точно так же, и можно разбирать строку хоть по одному символу. TS при записи `${infer TLeft}${infer TRight}` в TLeft помещает одну букву.  
```typescript
type TLetters<Str extends string> = Str extends `${infer TLeft}${infer TRight}` ? TLeft | TLetters<TRight> : never;

// когда тип разберет всю строку и останется пустая строка - будет never, и он отсечется от union
type TA = TLetters<'hello'>; // h | e | l | o
```
Особенность в том, что используя union мы потеряем повторяющиеся буквы. Решается с помощью кортежей:
```typescript
type TLetters<Str extends string, Acc extends Array<string> = []> = Str extends `${infer TLeft}${infer TRight}` ? [...Acc, TLeft, ...TLetters<TRight, Acc>] : Acc;

type TA = TLetters<'hello'>; // [h, e, l, l, o]
```

## Branded types

Бренды добавляют примитиву специальную "метку", чтобы TypeScript перестал считать похожие значения взаимозаменяемыми.
Это полезно, когда у вас несколько разных идентификаторов, которые все являются `string`, но путать их нельзя.

### Пример использования Branded types

```typescript
// Можно же было обойтись алиасами, но это небезопасно
type TSomeId = string;

type TBrand<TValue, TName extends string> = TValue & {
  readonly __brand: TName;
};

type TUserId = TBrand<string, "TUserId">;
type TOrderId = TBrand<string, "TOrderId">;

type TUser = {
  id: TUserId;
  name: string;
};

declare function getUserById(userId: TUserId): TUser;

// к сожалению, без кастов тут не обойтись
const tUserId = "user-1" as TUserId;
const tOrderId = "order-1" as TOrderId;

const tUser = getUserById(tUserId);
// getUserById(tOrderId); // Ошибка типов
```
та же история может быть с символами
```typescript
declare const UserBrand: unique symbol;

type TUserId = string & { [UserBrand]: 'userBrand' };
```

### Где применим паттерн Branded types

- Когда есть несколько разных ID одного и того же базового типа
- Когда нужно различать "сырую строку" и "валидированный токен" или "нормализованный email"
- Когда вы хотите защититься от случайной подстановки не того значения в API
- Когда доменная модель требует более строгих различий

### Объяснение паттерна Branded types

По умолчанию TypeScript смотрит на структуру типа. Если два типа выглядят одинаково, он считает их совместимыми.
Branded type добавляет к значению фиктивное поле, которого нет в реальном объекте в рантайме, но которое существует для системы типов.
Из-за этой "метки" `string & { __brand: "TUserId" }` и `string & { __brand: "TOrderId" }` уже считаются разными типами.
Так мы создаем дополнительный уровень строгости без изменения runtime-данных.

PS вас заколебет использование `as`, поэтому можно сделать функци-конструкторы брендированных значений
```typescript
function makeUserId(rawId: string): TUserId {
    return rawId as TUserId;
}
```
Теперь вас заколебет вызов makeUserId :D

## Builder / state machine

Этот паттерн закрепляет состояние API прямо в типах. Идея в том, что после каждого шага builder возвращает новую версию самого себя, но уже с другим состоянием.
За счет этого можно сделать так, что неправильный порядок действий даже не выражается в коде: нужный метод либо недоступен, либо является строкой-с-ошибкой, пока не выполнены обязательные шаги.

### Пример использования Builder / state machine

```typescript
type TClient = {
  send(): void;
};

type TClientBuilder<
  THasBaseUrl extends boolean = false,
  THasAuth extends boolean = false
> = {
  withBaseUrl(url: string): TClientBuilder<true, THasAuth>;
  withAuth(token: string): TClientBuilder<THasBaseUrl, true>;
  build: THasBaseUrl extends true
    ? THasAuth extends true
      ? () => TClient
      : 'Auth token not provided'
    : 'Base url not provided'
;
};

declare function createClient(): TClientBuilder;

const tBuilder = createClient();
const tConfiguredBuilder = tBuilder
  .withBaseUrl("https://api.example.com")
  .withAuth("token");

tBuilder.build(); // ошибка, строка не вызываемая
tConfiguredBuilder.build(); // все ок
```
Если использовать не строки, а never в поле build - можно обернуть весь тип в отсекатель значений never, и тогда .build даже в автокомплите не будет светиться.

### Где применим паттерн Builder / state machine

- Когда ваш класс должен проходить обязательные шаги конфигурации
- Когда нужно запретить вызов метода до того, как собраны все обязательные данные
- Когда вы строите query builder, API client, transport, pipeline или сложный API
- Когда важно сделать неправильный сценарий использования невозможным еще до рантайма (но в рантайме все равно защищайтесь)

### Объяснение паттерна Builder / state machine

У такого builder есть скрытое состояние, но хранится оно не в объекте, а в дженериках типа.
Каждый метод меняет эти параметры и возвращает новую версию builder.
Дальше условный тип смотрит на текущее состояние и решает, какие методы доступны. 
Если обязательный шаг еще не пройден, TypeScript подставляет `never`, и метод фактически исчезает из API. 
Поэтому этот паттерн так хорошо подходит для защиты от неправильной конфигурации: вы не ждете ошибку в рантайме, а запрещаете саму возможность написать некорректную цепочку вызовов.

## Path generation

Path generation строит union строковых путей до полей объекта. Такой паттерн полезен там, где поле выбирается не напрямую через `obj.user.name`, а строкой вроде `"user.name"`(например в lodash)
Вместо "любая строка" мы получаем строгий список допустимых путей, который IDE подсказывает автоматически.
Важно сказать, что этот паттерн очень "скользкий" - у него множество пограничных случаев(даты, ключи-символы, массивы). Ниже - не готовый рецепт, а пример.
В реальной жизни такой паттерн быстро разрастается, начинает тормозить и может генерировать лишние пути.

### Пример использования Path generation

```typescript
type TPaths<TValue> =
  TValue extends object
    ? {
        [TKey in keyof TValue]:
          TKey extends string
            ? TValue[TKey] extends object
              ? TKey | `${TKey}.${TPaths<TValue[TKey]>}`
              : TKey
            : never;
      }[keyof TValue]
    : never;

type TSettings = {
  profile: {
    name: string;
    contacts: {
      email: string;
      phone: string;
    };
  };
  theme: {
    mode: "light" | "dark";
  };
};

type TSettingsPaths = TPaths<TSettings>;
```

### Где применим паттерн Path generation

- Когда нужно типизировать строки путей для `get`, `set`, `watch`, `sortBy`
- Когда вы делаете form engine, state manager или таблицу с доступом к вложенным полям
- Когда фильтры, сортировка или селекторы принимают путь к полю строкой
- Когда хотите заменить "магические строки" на проверяемый набор допустимых значений

### Объяснение паттерна Path generation

Этот паттерн сочетает recursion, mapped types и template literal types.
TypeScript идет по каждому ключу объекта. Если значение по ключу еще объект, он создает два варианта: путь только до текущего ключа и путь до вложенных полей через точку.
Если значение уже не объект, остается только сам ключ. Потом все варианты собираются в один union.
В итоге получается полный список допустимых строковых путей по всей структуре.

---

## Numbers

В TypeScript математика на типах обычно делается не над самим `number`, а над его представлением. 
Самый популярный прием: представить число как длину кортежа. Если кортеж содержит 3 элемента, то на уровне типов это и есть "число 3".
Отсюда появляется базовая ментальная модель: вместо арифметики над числами мы делаем преобразования над массивами фиксированной длины.

### Пример использования Numbers

```typescript
type TZero = [];
type TOne = [unknown];
type TTwo = [unknown, unknown];

type TLength<TItems extends unknown[]> = TItems["length"];

type TBuildTuple<
  TNumber extends number,
  TAcc extends unknown[] = []
> = TAcc["length"] extends TNumber
  ? TAcc
  : TBuildTuple<TNumber, [...TAcc, unknown]>;

type TExample0 = TBuildTuple<0>;
type TExample3 = TBuildTuple<3>;
type TExample5 = TBuildTuple<5>;

type TLength0 = TLength<TZero>;
type TLength1 = TLength<TOne>;
type TLength2 = TLength<TTwo>;
type TLength3 = TLength<TExample3>;
type TLength5 = TLength<TExample5>;
```

### Где применим паттерн Numbers

- Когда вы хотите делать арифметику на типах: сложение, вычитание, умножение, инкремент, декремент
- Когда нужно ограничивать глубину рекурсии в type-level утилитах
- Когда вы пишете продвинутые utility types, где важно считать шаги или уровни вложенности
- Когда нужно понимать, на чем вообще строится "математика" в TypeScript

### Объяснение паттерна Numbers

TypeScript умеет точно хранить длину tuple как литерал: не просто `number`, а конкретно `0`, `1`, `2`, `3` и так далее.
Этим и пользуются type-level паттерны. Мы кодируем число не как "математическую сущность", а как "количество элементов в массиве".
Вспомните школу, как вы учились считать в первом классе? С помощью счетных палочек. Кучка из двух палочек, кучка из трех палочек, сложили в одну кучку, посчитали - пять палочек! (нам кстати  на линейной алгебре на первом курсе так же расказывали)
Дальше любую операцию можно свести к операциям над кортежами: склеить два массива, убрать хвост, добавить элемент, сравнить длины.
Поэтому почти вся математика на типах крутится вокруг `["length"]`.

## Math operations

Когда число представлено как кучка палочек, арифметические операции начинают выглядеть очень естественно.
Сложение превращается в складывание двух кучек в одну, вычитание в удалении нескольких палочек из кучи, а умножение превращается в повторяющееся сложение.
Важно помнить, что это не "настоящая" арифметика над `number`, а вычисление через структуру типов.

### Пример использования Math operations

```typescript
type TBuildTuple<
  TNumber extends number,
  TAcc extends unknown[] = []
> = TAcc["length"] extends TNumber
  ? TAcc
  : TBuildTuple<TNumber, [...TAcc, unknown]>;

type TAdd<TLeft extends number, TRight extends number> =
  [...TBuildTuple<TLeft>, ...TBuildTuple<TRight>]["length"];

type TSubtract<TLeft extends number, TRight extends number> =
  TBuildTuple<TLeft> extends [...TBuildTuple<TRight>, ...infer TRest]
    ? TRest["length"]
    : never;

type TInc<TValue extends number> =
  [...TBuildTuple<TValue>, unknown]["length"];

type TDec<TValue extends number> =
  TBuildTuple<TValue> extends [...infer TRest, unknown]
    ? TRest["length"]
    : never;

type TMultiply<
  TLeft extends number,
  TRight extends number,
  TAcc extends unknown[] = []
> = TRight extends 0
  ? TAcc["length"]
  : TMultiply<TLeft, TSubtract<TRight, 1>, [...TAcc, ...TBuildTuple<TLeft>]>;

type TAddExample = TAdd<2, 3>; // 5
type TSubtractExample1 = TSubtract<5, 2>; // 3
type TSubtractExample2 = TSubtract<2, 5>; // never! Отрицательные числа - отдельная история
type TIncExample = TInc<4>;
type TDecExample = TDec<4>;
type TMultiplyExample1 = TMultiply<3, 4>;
type TMultiplyExample2 = TMultiply<5, 0>;
```

### Объяснение паттерна Math operations

Сложение работает так: строим кортеж длины `A`, строим кортеж длины `B`, склеиваем их и берем новую длину.
Вычитание работает через pattern matching: если кортеж длины `A` можно разложить как "сначала кортеж длины `B`, а потом хвост", значит длина хвоста и есть `A - B`.
Инкремент и декремент это просто добавление или удаление одного элемента.
Умножение обычно строят как повторяющееся сложение: на каждом шаге добавляют в аккумулятор еще `A` элементов, пока счетчик `B` не дойдет до нуля.
Все это работает, потому что TypeScript умеет сохранять точные длины кортежей и рекурсивно вычислять условные типы.

## Compare numbers

Сравнение чисел на типах тоже делается через кортежи. Идея простая: если кортеж длины `A` можно представить как "кортеж длины `B` плюс еще какой-то хвост", значит `A >= B`.
Из таких проверок строятся `>`, `<`, `<=`, `>=`, а если нужен более подробный результат, можно сравнивать два кортежа рекурсивно и получать `"less" | "equal" | "greater"`.

### Пример использования Compare numbers

```typescript
type TBuildTuple<
  TNumber extends number,
  TAcc extends unknown[] = []
> = TAcc["length"] extends TNumber
  ? TAcc
  : TBuildTuple<TNumber, [...TAcc, unknown]>;

type TGte<TLeft extends number, TRight extends number> =
  TBuildTuple<TLeft> extends [...TBuildTuple<TRight>, ...unknown[]]
    ? true
    : false;

type TGt<TLeft extends number, TRight extends number> =
  TLeft extends TRight ? false : TGte<TLeft, TRight>;

type TLte<TLeft extends number, TRight extends number> = TGte<TRight, TLeft>;

type TLt<TLeft extends number, TRight extends number> =
  TLeft extends TRight ? false : TLte<TLeft, TRight>;

type TCompareTuples<
  TLeft extends unknown[],
  TRight extends unknown[]
> =
  TLeft extends [unknown, ...infer TLeftRest]
    ? TRight extends [unknown, ...infer TRightRest]
      ? TCompareTuples<TLeftRest, TRightRest>
      : "greater"
    : TRight extends [unknown, ...infer _TRightRest]
      ? "less"
      : "equal";

type TCompare<TLeft extends number, TRight extends number> =
  TCompareTuples<TBuildTuple<TLeft>, TBuildTuple<TRight>>;

type TGteExample1 = TGte<5, 3>;
type TGteExample2 = TGte<3, 5>;
type TGtExample = TGt<4, 4>;
type TLteExample = TLte<2, 7>;
type TLtExample = TLt<7, 2>;

type TCompareExample1 = TCompare<2, 5>;
type TCompareExample2 = TCompare<7, 1>;
type TCompareExample3 = TCompare<4, 4>;
```

### Объяснение паттерна Compare numbers

В проверке `A >= B` TypeScript пытается понять, можно ли кортеж длины `A` представить как "кортеж длины `B` и еще какой-то хвост".
Если можно, значит `A` не меньше `B`. Для строгого `>` добавляют отдельную проверку на равенство, потому что `>=` для одинаковых чисел тоже даст `true`.
Рекурсивное сравнение работает еще прямолинейнее: мы одновременно снимаем по одному элементу с обоих кортежей.
Если один закончился раньше, он меньше. Если оба закончились одновременно, они равны. Поэтому вся логика сравнения снова сводится к тому, как TypeScript умеет сопоставлять кортежи по форме.
