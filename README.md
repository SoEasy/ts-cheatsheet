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
- [Accumelator](#accumulator)
- [Simplify](#simplify)

Как типы описывают протокол использования API

- [variadic tuples](#variadic-tuples)
- [branded types](#branded-types)
- [typed builder](#builder)
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
Это базовый паттерн для написания сложных типов в TypeScript: мы программируем условие "если тип похож на такой, то
вернуть одно, иначе вернуть другое".
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
- Когда нужно разрешить или запретить часть API в зависимости от конфигурации - можно возвращать более широкий или узкий
  интерфейс
- Когда получаемый интерфейс зависит от режима работы, например `GET`-запрос и `POST`-запрос имеют разный конфиг
- Когда нужно типизировать что-то с несколькими ветками поведения

### Объяснение паттерна Conditional types

Это, по сути, тернарный оператор: если левый тип совместим с правым, берется ветка после `?`, если нет, то ветка после
`:`.
Отличие от тернарника и условий в JS в том, что TypeScript умеет сравнивать именно типы - один тип с другим через
`extends`.
Extends - это не совсем равенство в нашем понимании. Тут вам стоит посмотреть на иерархию базовых типов в TS.
Круто, что это происходит не в рантайме, а во время вычисления типа - рантайм еще не родился, а типы уже ветвятся.

### Лайфхак

Мякотка, которая помогает и вам, и нейросетям. Наверняка вы встречали ошибку "Type '...' is not assignable to type '
never'".
Попробуй пойми, откуда там never, особенно если много вложенных тернарников...
В случае тупиковых ситуаций описывайте не never, а нормальное сообщение об ошибке.

```typescript
type TExampleIf<TCondition, TData> = TCondition extends true ? { data: TData } : "Invalid condition";

type TResult = TExampleIf<false, string>;
const result: TResult = {data: 'hello, data!'}; // Error: Invalid condition - теперь все понятно
```

Но! Это меняет модель типа. Вместо невозможного состояния(never) теперь валидная строка, и в каких-то случаях она может
утечь дальше по реализации типов.
Поэтому иногда такой лайфхак удобно применять при разработке сложных типов, но после того, как типы отлажены - возможно
стоит вернуться к never.

## infer

`infer` позволяет вытащить __часть__ типа, сохранить ее в переменную и вернуть.
Проще всего думать о нем как о маске: мы описываем форму, которую хотим распознать, и если тип в эту форму попадает, TS
достает нужный кусок наружу.

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
- Когда тип должен автоматически выводиться из структуры, а не задаваться вручную - infer тут сработает для получения
  начальной "картины"

### Объяснение паттерна infer

Важная штука: `infer` работает только внутри conditional types, которые описаны выше.
TypeScript пытается сопоставить ваш тип с шаблоном слева от `?`. Если шаблон подходит, все места, где стоит `infer`,
заполняются значениями типов из исходного типа.
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
- похожестей не осталось, остался infer. Что там в переданном типе на месте infer? А там void. Ок, вот его в infer и
  положим.
- срабатывает тернарник, похожести соблюдены, в TResult уже лежит значение, возвращаем его

## Object.values & key access

Иногда бывает так, что у вас есть четко описанный тип глубокого объекта, или вы построили тип с помощью маппинга, как-то
преобразовали его значения, и теперь вам нужны только значения, а не ключи.
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
Когда мы пишем `TSomeObject[keyof TSomeObject]`, TypeScript берет объект и говорит: "дай мне тип значения по каждому из
этих ключей".
Поскольку ключей несколько, результатом становится union всех значений объекта. Поэтому этот паттерн и похож на
Object.values: мы как будто выбрасываем ключи и оставляем только содержимое. А дальше этот union уже удобно фильтровать,
преобразовывать и использовать в других паттернах.

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

function getUserById(userId: number): TUser {
}

// КОГДА ТИП ПРОХОДИТ НАСКВОЗЬ
// теперь сама функция не развалится, а развалятся конкретные её вызовы в коде - минус одна ручная правка
function getUserById(userId: TUser['id']): TUser {
}
```

## Tuple to union

Кортеж - это как массив, но конечный и со строгой длиной.
Иногда набор значений удобнее сначала описать как кортеж(tuple): можно использовать его в рантайме и получать точные
литеральные типы.
Но в TS часто нужен не сам кортеж, а union всех его элементов. Для этого в TypeScript есть очень короткий, но полезный
прием: TTuple[number].
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

Во-первых, видим новое слово - typeof. Эта штука позволяет из рантаймового значения получить тип. Удобно, чтобы не
дублировать. Т.е. рантаймовое значение первично.
Во-вторых, кортежи. Кортежи - это частный случай массива, у которого известны точные элементы по индексам.
Когда мы пишем TRoles[number], мы говорим TS: "дай мне тип значения, которое лежит в этом tuple по любому числовому
индексу".
Поскольку в tuple по разным индексам лежат разные литералы, TypeScript собирает их в union: "admin" | "user" | "guest".
Поэтому этот паттерн удобен как мост между списком значений и union-типом, с которым уже можно работать дальше.

---

## Distributive conditional types

Дистрибутивность в TS применяют условие к каждому элементу union по отдельности. Казалось бы, тот же COnditional type,
но с черной магией для юнионов.
Это очень мощный механизм, потому что позволяет фильтровать union, преобразовывать его или разбирать на группы. Если
хотите фильтровать объединения типов - вам точно сюда!

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
Важно объяснить слово "голый". Голый - это когда просто `string | number` - просто юнион. Не-голый будет, когда юнион "
одет"/"обернут" в функцию - `(arg: string | number) => void`.
Читай "голый" = "дистрибутивность". Т.е. когда слышишь "дистрибутивность" - думай о голых union. (хотел пошутить, но
слишком пошло будет)
Не-голый юнион пригодится нам в [Union to intersection](#union-to-intersection), о нем ниже, и картинка в голове
сложится.

Голый юнион - `string | number` - не проверяется как один большой тип, а раскладывается на две проверки: сначала для
`string`, потом для `number`.
После этого результаты снова собираются обрано в юнион, а всё, что имеет значение never - отсекается. Именно так можно
фильтровать юнионы.

### Лайфхак

Фильтрация union через тернарник иногда зашумляет код - появляются лишние ветки условий, которые путают.
Вместо этого можно использовать элегантное отсечение, если вы точно знаете какие типы входят в union. ТОЛЬКО если вы
знаете какие типы входят в union.
Например, keyof T возвращает поля объекта, если его сигнатура известа. А в общем случае объектов keyof T вернет number |
string | symbol В сложных типах, особенно рекурсивных, символ очень гадит, и его хочется отсекать.

```typescript
declare const UserBrand: unique symbol;

type TUser = {
  id: string;
  name: string;
  [UserBrand]: 'userBrand'
}
```

Пример очень синтетический и решается другими средствами, но для примера оставлю. Предположим, надо отсечь от типа поля,
ключ которых - символ.

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
Другое название этого паттерна - "избавиться от дистрибутивности". Когда понимаете, что надо работать с целым юнионом и
весь его собрать в одно - вам сюда!
`Union-to-intersection` сам по себе превращает union в пересечение.
На первый взгляд это выглядит странно, но на практике очень полезно, когда у вас есть набор вариантов и из них нужно
собрать один объединенный тип.

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
Сперва у нас дистрибутивный(голый) юнион `A | B | C`. Сперва мы его одеваем в функцию, и получаем
`(value: A) => void | (value: B) => void | (value: C) => void`.
Дальше TypeScript пытается понять, какой один тип аргумента подойдет для такой функции в целом, и получает пересечение
`A & B & C`.
Это связано с тем, как проверяются типы аргументов функций - когда в позиции аргумента стоит объединение(union)
функций - TS должен вывести такой тип, который безопасно передать в любую из этих функций.
В сложном мире это называется "контрвариантность", но про эти штуки мы тут говорить не будем.
Выглядит капец непрозрачно, но идея у него простая: через функцию мы заставляем TypeScript "схлопнуть" варианты в одну
общую форму.
Подробнее можно почитать [тут](https://dev.to/anuraghazra/explain-like-im-five-typescript-uniontointersection-type-5ako)

## Intersection-to-union

Казалось бы, union в intersection превращать умеем, надо бы уметь и наоборот. Но тут "фигвам". Обратно этот фарш не
проворачивается в мясо.

У TS есть встроенная дистрибутивность - о ней были два предыдущих паттерна. Т.е. если у вас УЖЕ есть откуда-то взявшиеся
union-ы - вы можете вертеть ими как хотите.
Но если вы их навертели в intersection - TS считает это единым типом, и он не обязан и не хранит "как так получилось".
Все, котлетка слеплена.
Поэтому, если хотите всетаки сохранить исходный union - сохраняйте его отдельно(чаще с помощью паттерна builder или
вообще не трогайте исходный тип).

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

Либо вы можете воспользоваться `Pick`/`Omit`/`Exclude`, чтобы извлечь/отсечь нужные куски типа, но это не сохранит
начальные условия его появления.

## Mapped types

Mapped types позволяет пройтись по всем ключам объекта и массово преобразовать его свойства.
Здесь у нас появляются способы извлекать ключи объекта и итераторы - мы уже знали `if` на типах, теперь узнаем что-то
типа `map`.
Это способ сказать TypeScript: "возьми ту же структуру ключей, но поменяй значения по определенному правилу".
Такой паттерн удобен, когда нужно строить DTO, readonly-версии, nullable-версии и любые массовые трансформации значений
модели.

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
- `[K in keyof TObject]` - это итератор, он пройдется по каждому ключу, и для каждого ключа TS создает новое значение -
  то, что после двоеточия - и положит в результирующий тип пару ключ-новое_значение.
- `TObject[K]` - доступ к значению по ключу во время итерации
  Благодаря этому не нужно руками переписывать каждое поле(мы же тут про генерацию). Мы просто задаем правило, а
  TypeScript применяет его ко всей структуре.

## Key remapping

Key remapping расширяет mapped types и позволяет не только менять значения, но и переименовывать или удалять ключи.
Это полезно, когда из внутренней схемы нужно получить внешний API с другими именами методов или когда часть полей надо
отфильтровать.
Добавлю, что прикручивание сюда template literal types позволяет творить невероятную мощь - всякие ToCamelCase,
ToSnakeCase итп.

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

Обычный mapped type оставляет имя ключа таким же, каким оно было. Key remapping добавляет часть `as`, в которой можно
вычислить новое имя.
Если в `as` вернуть `never`, TypeScript просто не создаст такой ключ в итоговом объекте.
Если вернуть строковый шаблон(о них далее) - ключ будет переименован.
Поэтому этот паттерн удобен сразу для двух задач: фильтрации и генерации нового API поверх старой структуры.

Ключевые моменты этого паттерна:

- просто `as` - переименование ключа
- `as ... extends ? ... : never ` - фильтрация и/или переименование ключа (потому что never отсекается)

Обязательно надо учитывать, что это только типы - если вы переименовываете ключи - ваш рантайм должен тоже это делать, и
там надо обмазаться защитами.
Т.е. вы в рантайме по тем же правилам, по которым сформировали тип с новыми ключами - должны сформировать новый объект с
этими же ключами.
Если что-то пойдет не так и рантайм не сгенерирует какой-нибудь ключ - получите например ошибку undefined is not a
function - так себе удовольствие это дебажить.
Чтобы этого избежать и проще дебажить - генерацию в рантайте можно сделать на Proxy, чтобы ваш get или apply в хэндлере
понимали, что вот такой ключ получен в процессе генерации, и почему-то значения нет.
Тогда можно выплюнуть более понятную ошибку.
Запомните: трансформация типов не делает runtime-работу за вас - runtime-реализация должна повторять те же правила, и
внятные ошибки в консоль/сентри помогут быстрее ловить ошибки, если реализации разошлись.

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
Эта возможность полезна там, где строки на самом деле несут значение и структуру: роуты, события, путь к полю(
obj.a.b.c).
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
Если вы пишете строку вида `${infer TLeft}.${infer TRight}`, TypeScript пытается распарсить исходную строку по этому
шаблону и положить найденные части в `TLeft` и `TRight`.
В результате строки превращаются в маленький DSL, который можно разбирать, анализировать и собирать обратно.

## Recursion

Как бы объяснить рекурсию без рекурсии...
Рекурсивные типы позволяют типу вызывать самого себя на вложенных частях структуры.
Это нужно, когда объект может быть глубоким, а преобразование должно применяться не только к верхнему уровню, но и ко
всем вложенным полям. Такой паттерн лежит в основе например `DeepPartial`, `DeepReadonly`.
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

TypeScript позволяет типу ссылаться на самого себя. Обычно это выглядит так: если текущее значение внутри логики типа
еще подходит под обработку - применяем ту же самую логику к текущему значению.
Главное здесь наличие базового случая, иначе TypeScript не поймет, где нужно остановиться.

Будьте аккуратны! Глубокие рекурсии могут заметно тормозить.

## Accumulator

Аккумулятор вместе с рекурсией позволяет вам в дженерике накапливать какую-то информацию, задаваемую динамически, в
коде.

### Пример использования аккумулятора

Предположим, надо разбить строку на слова по разделителю "пробел".
С помощью рекурсии мы бы сделали так:

```typescript
type TWordsRecursion<TString> = TString extends `${infer TWord} ${infer TRest}`
  ? TWord | TWordsRecursion<TRest>
  : TString extends '' // базовый случай
    ? never
    : TString;
```

С помощью аккумулятора несколько иначе

```typescript
// TAcc - аккумулятор - объект. В его ключи будем складывать слова
type TWordsAcc<TString, TAcc extends object = {}> = TString extends ''
  ? keyof TAcc // базовый случай - доразобрали всю строку
  : TString extends `${infer TWord} ${infer TRest}` // разбираем пока есть пробелы
    ? TWordsAcc<TRest, TAcc & { [K in TWord]: any }> // рекурсивно вызываем сами себя и пополняем аккумулятор
    : TString extends `${infer TWord}` // разбираем когда пробелов уже нет
      ? TWordsAcc<'', TAcc & { [K in TString & string]: any }> // перейдем в базовый случай
      : never;
```

### Где применим аккумулятор

- Когда вы парсите строковый DSL и хотите накапливать найденные токены, параметры или сегменты в отдельном типе, чтобы
  потом продолжить с ним работу
- Когда вы строите builder-подобные типы, где на каждом шаге добавляется новая информация

### Объяснение аккумулятора

Аккумулятор в программировании на типах работает почти так же, как в обычном reduce в JS. Мы передаем в рекурсивный тип
дополнительный параметр, в котором храним уже накопленный результат, и на каждом шаге не только разбираем новый кусок
входных данных, но и обновляем этот параметр.

В этом примере аккумулятор сделан объектом, потому что объект удобно накапливать через пересечение &. На каждом шаге мы
добавляем новое слово как ключ: `TAcc & { [K in TWord]: any }`
Так TypeScript собирает объект, у которого ключами становятся все найденные слова. В конце нам нужны не значения этого
объекта, а только имена ключей, поэтому в базовом случае мы возвращаем keyof TAcc. Это и дает union всех слов.

Почему здесь удобно использовать именно объект и keyof:

- объект позволяет естественно "добавлять" новые элементы через пересечение типов;
- если слово встретилось несколько раз, ключ просто останется тем же, то есть мы автоматически получаем уникальные
  значения;
- пустая строка не попадет в результат, потому что в базовом случае при TString extends '' мы просто возвращаем уже
  накопленные ключи и ничего нового не добавляем;
- если входная строка была пустой с самого начала, аккумулятор останется {}, а keyof {} даст never, что как раз логично:
  слов нет.

Иногда паттерн с аккумулятором удобнее прямой рекурсии. Прямая рекурсия обычно сразу "возвращает"результат
наружу, а аккумулятор позволяет сначала собрать все промежуточные данные, а уже потом в одном месте превратить их в
финальный тип(как в этом примере - преобразование = keyof).

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

После сложных преобразований TypeScript часто хранит тип как композицию нескольких частей, а не как обычный "плоский"
объект.
`TSimplify` берет все ключи итогового типа и заново создает новый объект с теми же значениями. Смысл типа не меняется,
но представление становится проще.
Это не магия и не новый алгоритм, а скорее способ попросить TypeScript показать уже вычисленный результат в более
удобной форме.
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
  return {url, retries};
});
```

### Где применим паттерн Variadic tuples

- Когда нужно писать middleware, decorators и wrapper-функции без потери аргументов
- Когда вы делаете `pipe`, `compose`, `curry` или цепочку вызовов
- Когда нужно добавлять или убирать аргументы у функций на этапе работы с типами

### Объяснение паттерна Variadic tuples

Кортеж в TypeScript может хранить не просто "массив чего-то", а точную последовательность элементов.
И как и в JS - у TS есть понятие rest(собрать) и spread(разобрать) оператор.
Variadic tuples позволяют в шаблоне сказать: "вот один элемент, а дальше идет произвольный хвост". Когда TypeScript
сопоставляет tuple с таким шаблоном, он умеет отдельно вынуть хвост через спред-оператор и `infer`.
Поэтому, например, аргументы функции можно разбирать и собирать так же, как обычную структуру. Это и дает возможность
писать обертки, которые не ломают сигнатуру.

#### Лайфхак

Со строками паттерн работает точно так же, и можно разбирать строку хоть по одному символу. TS при записи
`${infer TLeft}${infer TRight}` в TLeft помещает одну букву.

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
Branded type добавляет к значению фиктивное поле, которого нет в реальном объекте в рантайме, но которое существует для
системы типов.
Из-за этой "метки" `string & { __brand: "TUserId" }` и `string & { __brand: "TOrderId" }` уже считаются разными типами.
Так мы создаем дополнительный уровень строгости без изменения runtime-данных.

PS вас заколебет использование `as`, поэтому можно сделать функци-конструкторы брендированных значений

```typescript
function makeUserId(rawId: string): TUserId {
  return rawId as TUserId;
}
```

Теперь вас заколебет вызов makeUserId :D

## Builder

Этот паттерн закрепляет состояние API прямо в типах. Идея в том, что после каждого шага builder возвращает новую версию
самого себя, но уже с другим состоянием.
За счет этого можно сделать так, что неправильный порядок действий даже не выражается в коде: нужный метод либо
недоступен, либо является строкой-с-ошибкой, пока не выполнены обязательные шаги.

### Пример использования Builder

```typescript
type TClient = {
  send(): void;
};

type TApiBuilder<HasUrl extends boolean = false, HasAuth extends Boolean = false> =
  HasUrl extends true
    ? HasAuth extends true
      ? TClient
      : { setAuth: (authToken: string) => TApiBuilder<HasUrl, true> }
    : HasAuth extends true
      ? { setUrl: (url: string) => TApiBuilder<true, HasAuth> }
      : {
        setAuth: (authToken: string) => TApiBuilder<HasUrl, true>,
        setUrl: (url: string) => TApiBuilder<true, HasAuth>
      };

declare function createClient(): TApiBuilder;

const tBuilder = createClient();
const tConfiguredBuilder = tBuilder
  .setUrl("https://api.example.com")
  .setAuth("token");

tBuilder.send(); // ошибка, метода send нет
tConfiguredBuilder.send(); // все ок, получили TClient и вызвали send
```

### Где применим паттерн Builder

- Когда ваш класс должен проходить обязательные шаги конфигурации
- Когда нужно запретить вызов метода до того, как собраны все обязательные данные
- Когда вы строите query builder, API client, transport, pipeline или сложный API
- Когда важно сделать неправильный сценарий использования невозможным еще до рантайма (но в рантайме все равно
  защищайтесь)

### Объяснение паттерна Builder

У такого builder есть скрытое состояние, но хранится оно не в объекте, а в дженериках типа.
Каждый метод меняет эти параметры и возвращает новую версию builder. Такой подход - тоже своего рода аккумулятор.
Дальше условный тип смотрит на текущее состояние и решает, какие методы доступны.
Если обязательный шаг еще не пройден, TypeScript подставляет другой тип, и метод фактически исчезает из API.
Поэтому этот паттерн так хорошо подходит для защиты от неправильной конфигурации: вы не ждете ошибку в рантайме, а
запрещаете саму возможность написать некорректную цепочку вызовов.

### Более сложный builder

Кроме того, что можно требовать строгий набор настроек(без которого ваш код вообще не должен работать) - можно ещё
элегантно на стыке типов и рантайма добавлять пользовательские методы, которыми потом пользоваться в программе.
Например, можно удобно описать вообще все возможные вызовы API и типизировать их. Предположим пример:

```typescript
type TUser = { name: string };
type TOrderCreate = { orderId: number; amount: number; };
type TOrder = { id: number };
type TOrderUpdate = { amount: number };

const apiBuilder = createClient();
const api = apiBuilder
  .setUrl("https://api.example.com")
  .setAuth("token")
  // круто было бы затипизировать и параметры запроса, и ответ и имя сразу одной строкой конфига
  .addGetMethod<Array<TUser>>()('getAllUsers', '/user')
  .addGetMethod<TUser>()('getUserById', '/user/:id')
  .addPostMethod<TOrderCreate, TOrder>()('createOrder', '/order')
  .addPostMethod<TOrderUpdate, TOrder>()('updateOrder', '/order/:orderId')
  .build();

// Круто было бы, чтобы вызывать методы в api через точку, и чтобы все типы были на месте

// GET без параметров, тип ответа выведен
await api.getAllUsers(); // Array<TUser>

// GET с параметром, параметр типизирован, тип ответа выведен  
await api.getUserById({params: {id: 1}}); // TUser

// POST, тело типизировано, тип ответа выведен
await api.createOrder({body: {orderId: 1}}); // TOrder

// POST, тело типизировано, параметры типизированы, тип ответа выведен
await api.updateOrder({params: {orderId: 1}, body: {amount: 5}}) // TOrder
```

Вот как это сделать - здесь и рантайм, и типы:

[Ссылка на playground, где можно посмотреть более наглядно](https://www.typescriptlang.org/play/?#code/PTAEmIQR+EEARBD4QRJEEAwghWEEDwgpBCIIDhBKGkQUgLCCDcIIPIgoAkgCICiAUAC4CeADgKagAqAygJYC2zADY8AZowA8HAHygAvKADeoANoBpUDwB2oANatGAexGcAugC5Oak6AC+oAGSKbAblq0QoQBgggdhBATCBFkQC4QFHRIRGhEVGQMaHwsb1BAQRAY2ECsBhZ2DmoAD3oAJwBDAGN6AAVCor4AZ3EuUFY81k0AE2rQaoKtAHMZWVpQUHrG+ma20AADABIFTvyemzMZrRFWfNAKqptgZc1V9YAlVk6bCYHB0AB+RRV1LQ3Kwr5QAB9dfSNOXIKS8sea8RHTpScwdLqabq2c6DCzDJqtdrTWbg7qLXb7B5bM4XC7XJRqDTaTZPUFzHpQnEwpyudxgQD4IHBfMhoCQ0AAaUDIZKAGRBQEQsKBfLBYIBGEHgyFAgSIgDkQQDiIALfKBRYApECCGGQGTYFGq1AETAA8gAjABWrFKkiNptKfXehmMHEtZvoDXh400rAAbmsrqAAOQATWoXF9oAsvoAcvrfTSPJqsgBhQqCQSFQ2CVgAWUKzDYLQ4mUkWeYLtGCNABhNTptCnOBPuejtnCLoOhoA8AAoAIwASlAgEIQCLeQDCIKAhyLkAQsKhEiQObKMLBQJBmRPEIBeEDXuEAnCCgeKgNeIQKwSABNdYMhHteAERAsIBmEFAgFEQWAYXmyocvgC0b5fguZm63AB0EDjoAbCAECEM5LougRDokwSoByGCBKO+6IIknI8qA+SsAAjgArsc9AckeC6xDuiqJEQQ6yigGz6lwHAYS+HKIIKSRUTREoAOLUIxa7RKAH6gIAsiALqAqD4DAsBMtAU6EIgRDcoAEiDIABrYcEWVglmM7RKHwrD0AAFgYLQWCs3ocAAEhwHBlBmBnGS02llr6PEcCGby+mU9HuRSlLXFZNl2Q5JnOeMrm8SGHYAEy9oAqCCUQKbmCZgi7gYpKlLoyzJyZeN73ge+5DiBm4Sge6BSnK6SUriTbZlpIw6Tc2HVMwBiaNUrBmXsFlAm1HWsByYxtVo9DdRi2StCNmjOo1ZZkhCthtmA7YAMy9kJUkyay6Dzouip5Xe3G8UVJW4BKlXykx3KtjVHhXrA0CoFg0BDmgoBXkOa6BIAYiBYNKSRZYgvhYIpm7PUu4SRNEETPe2oMCuOCRng+grSYkyC9gKMDA6D3Lg0qGBMguqC3f52q6swBqVua2R5EUpTEgCk0tNN9BSDIc3hYGwY+u2vayDIZT5AYfA8J1kh9e1nUyBY7bFO1Ig8N0FhKMw-zVBYdM-IzGuSNQU0GKNMg2ALQsi2LEscFLA1SGTOJhuQmgekmPBOclCt7MrUUret+7ILeiDYL4B5kMTwQpQdsDXkdhCkKAyX8RgWHHP1nW7sACMNIbo326GnDWbZ9lGaFXPtF5Pk+6A7axf7gfB6Hv5IaxQlcvy-sJ7xiGLqgsAgQJ7YACy9t5DGXHngwBZpqjWGXzW4QRnTjb1C+Ec4KetdLXWEhNNudevw1GzNy-rCzbNhe0C2QnYHYAKwbVl0nMjtc5iVHMe3hKo+MVyP5rsVpVUIVRlPKCeFx7qPWeq9d6n1vp-QBuhQ0JlGC7iBiDMGuAIZhFhjDSIWB4btyRvuLAqMhS+AxljNBeMCbPmJrAUmNUarXHlorZWqtQBIJaIwLWRx8KESWo4ds5AdR6kYI6Wm3wGZ-CqLUM+R92ac1dOXHmIY8R2HYerGRWtJG-CZrIg2rN5Em27GbDYFtxasElqnLedtGGUkds7V2Tlv6gE9krboVd2z3zrkHLAIdEBh2iBHIS798oSmIGQFxScU74Ras6LAmd24tTTuwBJWdD65zsVSX0TsXbCCcoXMooB9Ilycu2fgbVqjVB4GmVg3ZPHDx8Q3AJTdgkd2SgKfidEGIpTXIuEcB50KBSLiFJyc8Iq+U8t-X0LFQDeAPPKZOBBjxYESLgXwiAAYVIMFUmp6Y1I1QcXkt2rjWGQkKM6JMghozLWrj2JpfjG7hxboTcC71khwBgoDWASksAiiPFuWgNg3BMC1BpUZRYABCeEeCCBaGsQs2YL7lhpuzOQihzgeEADgg0RYAEHwIUFoLQAIksfskRcB5ggnj8Zs5OWBryQMQPAtiIpYAZVgDuaAyBhyKWwK9aU8AsA7mwJATFYBADoIAQLkvJYCymQGuXk9L1RcsSP+cS9D5XJ3QsEaAslGTriwO9ISolFx4vpbuBIiRICwESKKaSoBBxSv5D+RU3AlSTjXIQfu-FZLIXQgQXwQ5UBkGwdDGIeDUKEA2V6+VOUI7oTxWeUIUMogpAYaAQlLQuIGWLo5Kxm9bb8wsOIT2nRQDhieOwOeV8OSludAY8+1aURSHbJoStFgK36SGjnY+oAG3yNMeC0pUKYVwoRRpJFjh8R3G0J21goI9KjLDG5G5ySt48OsQNA+Paxp9p3UtGwUhXCDEzWUHZ9Bc0mSsXwzoHJrabplkW0AJbpbOjncimtpyOr1v3U2+YEIW1tv0h2yt3bDGjQsP20ag7L0tBHbC+F+REXFinbcQk5bK0LuKUuv00z17YRvbu+9hH8MPu3vegt+9s7gd7VBmaB6j1uEGNi3FBAOGjqctaj5sAsDJz6b4bcmAcCAwE94dZIRMECkung84hoONPu4BU4QYhJCJmTKmdMRZcz5jYMhjmrgXAgsyJwAAgswHg0KEMIssoUaoABVfIghkVIIMOmQo2h5AiCTJ1DkNnqgmbwkZZFkKDCudYO59FXnBAyzkOcPzDmnNzwKARVs1w-MBaC0l-IKWmFNghdmSzY6kMKEPWTdhnV6AZcMnLQogXDIcAMHoTQFgr6DrMxZjj1nbMJY5Ml1gJtWwWHS3V5FfWyZ4g6AZBLcs8KOZayiNr5nCuIfEH13ztmqsDcOTcCrVWat1Ya01+b-7uiLY61ZpD8XHO9ey-1jkFXpvV1m4IY7PQzvLYRWt0Aw2jImxpKCrIdnOrrHkEoID28r62FcADzg+p8iIfjNhC57BQflnh2scgplQCaDwnwQ03pDMw4dOjkHNw3YWBx3jgn0PjPE8Q3Z5gLRkfoqUE8AweFe2U-x+sQztBigpiqaAeMwhmjOhrIMZg8wXajFAM917i03g4+TMe0AkueDS-YLVoyh3mjy8hIrvCyvzhq419h4d2YLBHAVvD8Qn7F2lOXZFV4uHK7bpo7uyHh6We88GA9xz7Y5dghO72cXFwjLiwAs99Fz2VeDGwvQWb2hw-VBV8C84u26vti1-Vxruug9vYxTiZPAFs8648xmg7ufNCx5Tgn-ISfDLi1T0xjNRLs0XtGa29t+eIRgbZnrkPrZi8lMcppcH1hUcj5Mo79yff5FQ9bPHxPoBk-N-OKe89sGu-AZ790OfEHd+D6L436oAEp9wfquPlnZvHJhmmfv+jhmcRL-ryvk-a-BhyYQ-zQvFwX8N6b3OGBTTxEA51KB4HalcSR1GGFx4FFwU3aw+yQyix81ABQP61-zrVcRF3o3kHdAAHchccD6B+YaRBhQDNBwDIDsIhAShWAKgjI9F2wMle1P1NEnhNZQArcDAbdP13NGBqxTZf848DJl8WD6AAIaCUxihWB2xgAzB2xlBCgPwAAvEzD8AALQAAYPwABOAAfRMAAGpuxgA99q59COQGxTFQ8cRRBq52CahlAGwJ88DDdBAj9GEjIRZCCCC+18gRZ8h2wJgMxxZqlFp1YgtHCLAZgGxThuwa9BhgUap-9s4FZ4U7MDhyB4xRZ+p4CuAUR2xHDqhnD9ATATEa9TYP80CwD6AIDtBsI+ADAvQgdWAWgGDDImDWxxC9c2RWxijLczQeCWhbcUQOR+DBDzhPDBgsCWpDdnRUcSUAJiiF9WwRAeDq4sC+ALlihDJyxjBxCz8djDITNkw5CFClDVD1DtC9DDCTCzCTFhCLh4V0wZc5jBB6BlBtj6BdjlBOwTATBKiW8RC68Gjjh5jqjUi-DhYDAchGB5ZiCM12h+CORbDQBugDJ2x6BKgMSiJVcRZmAORsIZD1c1hpiLh7CsTMhPhJcDBiwABCWQeQX0K+X0ck5-UQ1-I4EQdMUoACXErEnEgyDkWkwklOEkr0fIBIsmNPGqSk0U9DbE-IXE9ki4LAvJAidFbk3kiQgUpU3EkUgkoks0VgUkqUxIv-TkpPak4wDUlHJkv0CgqgzQVRUAO0gCQ0LQFoQU5UgyXsCwO0i02UykLY0ZHIr2SEeQfUgyM-fLZgZQUUwE4Eik4wdsek8-cM9xVUwYbwgwXw1gQg6gAIng4I2DUAAAIhmFFJsHLI0HaE0AMGdDcWVlmzaImGlJqmDI5NBKRMYEoOriZ2xMuAsBRNVw1mHK4KGN4LGIzU0AEJsLzkpPpOjIkOe2zLD0Mh8OxwLP8MCPbF9EhVs3YEyIABk6zsdGzJt6A2SLSkjkycQsD7dHIH9nQ7B5AMyzkyDGFXi0CtAkwEt0UVzI9HNQAjDqM2Zbyv0y1DJwtENODuDpyTt7tm0WcwFfRwzSx6APwdNWBfQwxswhAeBigLl6jgBjRqh2oZk84fdGFKSgLS8q91zBgYLCU1hT8qseCeAVCSLID5AJhIVwtsJ1gZh6LK8mtThIKuzKRKTz85AHSJk2SniaosDIjOiNZ0VBzChXg3gStILyC-zBAAKoyhTVyQKwKpC6COimDxCRSLk1KZFpSwEsC+F8hGA9F0VGjmjWBWj2i7LrKd1bLGCNYOysksDOpKhdj0U-DTyuBBLdimDHKsk0CNiWFv0VAGwOQ7TrBPhxEJDRd5hjh2wXK3LgqmKUzq47TQBGTsd3CyqcRwr8hdiS8cwxh2wMqhhCi7SKiwE7ykqpLlLX1QBiqCiTt0UGqmr6ADARqehSCwFKThqFslLGElY21DKzK+LLgZgFqTsJKwF+qQzBq10Bp0VCh8DCgeBnRVgfjDJ2wVr-zrslqapz8Z8qKkrmLYK2K+i+rEqsklyjrOoAJGs6q38ty-Ciy9yJhClQAZh-rWAAJOgLk8JqhFhoaFBYb4bsSE9qgOARh4i9KoQwFUj0byL2pZq7F9rBgZLRk5LmTpkyq7q1qnNjLfTTKnNzLWBaCZCrKNZmCAqxyZFtKnAfq7FZiyMTqzqLq0CDJdjbqDKetHrKRnqXcGJXq3qWK4Kvq3rOFuFQBNKqq3DkwfQAApLgfUcMDGk7UQeEzS-02XVoVgFatozW8m4W2i1M+k9GoGhW4-UGnc8GksyGoKVG9GhGrGlGmGsjDGxG7G3G9s-Gimy0ns4miizQMmxhBO3M-Mws4soIiYOzDqPCHMHg0YApIOpWmYc-PGmUyozWqooFNwLA7Mc7IrdFYoaA1gWA+A6Uxu8zE6pbTrfIc4eGqbf3cswyegegZgTWEAJugCRoJ4IQOGhWPgcs7sIejPIydscsyaprVesVSUV6MSEUXAIgRcY+uZXxSAeIF6aSIORVSGHBGIfAbwRAKSRIRADkAUWhASAUKcMgRIQcEcRNDlJIFIAFSUaOfKRAIezNdvWDcQEzAIwoCQDgVo-IDmfmfc3E04wy4HaoGZP0YAJGtYNkmBtvHNUZSQNBltbsLBqbYHSFRgTHAh30Ih4HeQt2UhwYEvIlM9ToeBunNYRHcLUYO9OHRDGh-ctukR1gcRkhjkVh4Ykhte7hjffhyhwR-IBnQcwaWHEnSR30QunRuR-IFh4AJR-IeQix5hlR0AD0+TRyjwZKIhUNVNRkRCYTb+jAViATJkKVNcWgU686y5czfkgyHBtB6oUg25RBooFB6htwJxk6dCVxmGehDkVJmIITK+7wG+jZNcDxyAQmYmHx9ZCceVQYQJiWkJngMJ+gNBxhzHdsNWDWdhcnP0TsEMOwKo25VB4HRJsAb+RCZAU+7J6+-VPpQp4p6IUpvx+VKp4JjNUJ6R5HEx5pjhZBdhaxrHTsA9BI3pkxgZ7pDgYZ0ZkVcZ2+yZh+sNWIMZ3JiZ3AKZrx2Z8pgJoJyW2eox1ZkndZgYm4bZsMTp2wDkbW9hNnDnXdW+WwIQjwTRtwIAA)

```typescript
// Упрощаем типы для IDE
type TSimplify<T> = { [K in keyof T]: T[K] } & {};

// Извлекаем параметры из строки
type TExtractParams<S extends string> =
  S extends `${string}:${infer Param}/${infer Rest}`
    ? { [K in Param | keyof TExtractParams<Rest>]: string }
    : S extends `${string}:${infer Param}`
      ? { [K in Param]: string }
      : {};

// Проверяем, есть ли вообще ключи в объекте
type IsEmptyObject<TObject> = keyof TObject extends never ? 'YES' : 'NO';

// 
type TCallableMappedType<TMap extends object> = {
  [K in keyof TMap]:
  // (1) Сразу убедимся, что переданный из накопления конфиг хоть чуть-чуть верный. Убеждаемся по кускам, тк у нас есть request, который в случае POST есть, а в случае GET нет - это мы проверим дальше.
  TMap[K] extends { method: infer THTTPMethod extends 'GET' | 'POST' }
    ? THTTPMethod extends 'GET' // (2) Если GET - то дальше проверим конфиг на нужные нам ключи
      ? TMap[K] extends { response: infer TResponse, endpoint: infer TEndpoint extends string } // (3) - проверяем что в конфиге GET нужные ключи есть
        // формируем функцию с правильными параметрами(или без них вовсе) и правильным ответом
        ? IsEmptyObject<TExtractParams<TEndpoint>> extends 'YES' ? () => Promise<TResponse> : (config: {
          params: TExtractParams<TEndpoint>
        }) => Promise<TResponse>
        : 'Invalid GET config' // (3) негативная ветка - в конфиге для GET нет response и/или endpoint
      : THTTPMethod extends 'POST' // (2) негативная ветка - если не GET, то может (4) POST?
        ? TMap[K] extends { request: infer TRequest; response: infer TResponse; endpoint: infer TEndpoint extends string } // (5) - проверяем, что в конфиге POST есть нужные нам ключи
          // формируем функцию с body и правильными параметрами(или без них вовсе) и правильным ответом
          ? (config: { body: TRequest } & (IsEmptyObject<TExtractParams<TEndpoint>> extends 'YES' ? {} : {
            params: TExtractParams<TEndpoint>
          })) => Promise<TResponse>
          : 'Invalid POST config' // (5) негативная ветка - в конфиге для POST нет requrest и/или response и/или endpoint
        : 'Invalid HTTP method (impossible)' // (4) негативная ветка - не GET и не POST - но у нас THTTPMethod extends 'GET' | 'POST', а значит дописываю impossible.
    : 'Invalid config at all'; // (1) негативная ветка - отдаем строку с ошибкой
}

type TMethodMapBuilder<TMap extends object> = {
  // Методы add... просто накапливают информацию в большой результирующий тип
  // Здесь очень интересный момент с каррированием - это один из способов разделить в TS один дженерик с двумя параметрами на два дженерика с одним параметром
  addGetMethod<TResponse>(): <const Name extends string, const Endpoint extends string>(name: Name, endpoint: Endpoint) => TMethodMapBuilder<TMap & {
    [K in Name]: {
      method: 'GET';
      response: TResponse;
      endpoint: Endpoint
    }
  }>;
  addPostMethod<TRequest, TResponse>(): <const Name extends string, const Endpoint extends string>(name: Name, endpoint: Endpoint) => TMethodMapBuilder<TMap & {
    [K in Name]: {
      method: 'POST';
      request: TRequest;
      response: TResponse;
      endpoint: Endpoint
    }
  }>;

  // Метод build построит новый тип с вызываемыми ключами
  build(): TSimplify<TCallableMappedType<TMap>>;
};

type TApiBuilder<HasUrl extends boolean = false, HasAuth extends Boolean = false> =
  HasUrl extends true
    ? HasAuth extends true
      ? TMethodMapBuilder<{}>
      : { setAuth: (authToken: string) => TApiBuilder<HasUrl, true> }
    : HasAuth extends true
      ? { setUrl: (url: string) => TApiBuilder<true, HasAuth> }
      : {
        setAuth: (authToken: string) => TApiBuilder<HasUrl, true>,
        setUrl: (url: string) => TApiBuilder<true, HasAuth>
      };

type TUser = { name: string };
type TOrderCreate = { orderId: number };
type TOrder = { id: number };
type TOrderUpdate = { amount: number };

class Client {
  private url: string | null;
  private authToken: string | null;
  private methodMap: Record<string, { method: 'GET' | 'POST'; endpoint: string }> = {};

  setUrl(url: string) {
    this.url = url;
    return this;
  }

  setAuth(authToken: string) {
    this.authToken = authToken;
    return this;
  }

  addGetMethod = () => (name: string, endpoint: string) => {
    this.methodMap[name] = {method: 'GET', endpoint};
    return this;
  }

  addPostMethod = () => (name: string, endpoint: string) => {
    this.methodMap[name] = {method: 'POST', endpoint};
    return this;
  }

  build() {
    return this;
  }
}

function createClient(): TApiBuilder<false, false> {
  const client = new Client();

  function replacePathParams(endpoint: string, params: Record<string, any> = {}) {
    return endpoint.replace(/:([a-zA-Z0-9_]+)/g, (_, key) => {
      if (params[key] == null) {
        throw new Error(`Missing path param: ${key}`);
      }
      return encodeURIComponent(String(params[key]));
    });
  }

  function removeUsedPathParams(
    endpoint: string,
    params: Record<string, any> = {}
  ) {
    const result = {...params};

    for (const match of endpoint.matchAll(/:([a-zA-Z0-9_]+)/g)) {
      delete result[match[1]];
    }

    return result;
  }

  return new Proxy(client as any, {
    get(target, prop, receiver) {
      if (typeof prop !== 'string') {
        return Reflect.get(target, prop, receiver);
      }

      if (prop in target) {
        const value = Reflect.get(target, prop, receiver);
        return typeof value === 'function' ? value.bind(target) : value;
      }

      const methodConfig = target.methodMap[prop];

      if (!methodConfig) {
        throw new Error(`Method "${prop}" is not configured`);
      }

      return async (data?: any, params?: Record<string, any>) => {
        if (!target.url) {
          throw new Error('Base URL is not set');
        }

        const {method, endpoint} = methodConfig;

        let finalUrl = target.url + endpoint;
        const headers: Record<string, string> = {
          'Content-Type': 'application/json',
        };

        if (target.authToken) {
          headers.Authorization = `Bearer ${target.authToken}`;
        }

        if (method === 'GET') {
          const pathParams = data || {};
          finalUrl = target.url + replacePathParams(endpoint, pathParams);

          const queryParams = removeUsedPathParams(endpoint, pathParams);
          const search = new URLSearchParams();

          for (const [key, value] of Object.entries(queryParams)) {
            if (value != null) {
              search.append(key, String(value));
            }
          }

          const queryString = search.toString();
          if (queryString) {
            finalUrl += `?${queryString}`;
          }

          const response = await fetch(finalUrl, {
            method: 'GET',
            headers,
          });

          if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
          }

          return response.json();
        }

        if (method === 'POST') {
          finalUrl = target.url + replacePathParams(endpoint, params || {});

          const response = await fetch(finalUrl, {
            method: 'POST',
            headers,
            body: data != null ? JSON.stringify(data) : undefined,
          });

          if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
          }

          return response.json();
        }

        throw new Error(`Unsupported HTTP method: ${method}`);
      };
    },
  });
}

const apiBuilder = createClient();
const api = apiBuilder
  .setUrl("https://api.example.com")
  .setAuth("token")
  // круто было бы затипизировать и параметры запроса, и ответ и имя сразу одной строкой конфига
  .addGetMethod<Array<TUser>>()('getAllUsers', '/user')
  .addGetMethod<TUser>()('getUserById', '/user/:id')
  .addPostMethod<TOrderCreate, TOrder>()('createOrder', '/order')
  .addPostMethod<TOrderUpdate, TOrder>()('updateOrder', '/order/:orderId')
  .build();

// GET без параметров, тип ответа выведен
await api.getAllUsers(); // Array<TUser>

// GET с параметром, параметр типизирован, тип ответа выведен  
await api.getUserById({params: {id: '1'}}); // TUser

// POST, тело типизировано, тип ответа выведен
await api.createOrder({body: {orderId: 1}}); // TOrder

// POST, тело типизировано, параметры типизированы, тип ответа выведен
await api.updateOrder({params: {orderId: '1'}, body: {amount: 5}}) // TOrder
```

### Объяснение более сложного билдера

На данный момент ваших знаний достаточно, чтобы понять почти всё, что происходит в коде выше - условные типы, infer,
аккумулятор...
Остаются непонятный пока момент - каррирование дженериков.
У дженериков есть особенность - если в дженерике несколько аргументов без дефолтных значений - передать надо либо ни
одного, либо все.

```typescript
function greet<const Name extends string, const Surname extends string>(name: Name, surname: Surname) {
  console.log('hello', name, surname);
}

greet('John', 'Smith'); // Name = John, Surname = Smith
greet<'John', 'Smith'>('John', 'Smith'); // Name = John, Surname = Smith
greet<'John'>('John', 'Smith'); // ошибка - ожидается два аргумента в дженерике
```

Если не передали ни одного - типы выведутся автоматически. В случае со строками надо добавлять const в объявлении
дженерика, чтобы тип не привелся просто к строке.
Если передали оба - ну и хорошо.
А вот передать только один - нельзя.

Один из способов победить эту проблему - каррирование - разделить одну функцию с дженериком от двух аргументов на две
функции с дженериками от одного аргумента.

```typescript
function greet<const Name extends string>(name: Name): <const Surname extends string>(surname: Surname) => void {
  return <Surname extends string>(surname: Surname) => {
    console.log('hello', name, surname);
  }
}
```

Таким образом в билдере в строках типа `.addGetMethod<TUser>()('getUserById', '/user/:id')` мы первый аргумент - тип
ответа - делаем обязательным, чтобы разработчик его указал, а второй аргумент - название метода - уносим в возвращаемую
функцию, и он будет выведен.
```typescript
// функция addGetMethod имеет один дженерик-аргумент
// возвращает функцию, которая так же имеет один дженерик-аргумент
// и в типе возвращаемого значения мы имеем доступ до обоих аргументов дженериков
addGetMethod<TResponse>(): <const Name extends string, const Endpoint extends string>(name: Name, endpoint: Endpoint) => TMethodMapBuilder<TMap & {
    [K in Name]: {
      method: 'GET';
      response: TResponse;
      endpoint: Endpoint
    }
}>;
```

## Path generation

Path generation строит union строковых путей до полей объекта. Такой паттерн полезен там, где поле выбирается не
напрямую через `obj.user.name`, а строкой вроде `"user.name"`(например в lodash)
Вместо "любая строка" мы получаем строгий список допустимых путей, который IDE подсказывает автоматически.
Важно сказать, что этот паттерн очень "скользкий" - у него множество пограничных случаев(даты, ключи-символы, массивы).
Ниже - не готовый рецепт, а пример.
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
TypeScript идет по каждому ключу объекта. Если значение по ключу еще объект, он создает два варианта: путь только до
текущего ключа и путь до вложенных полей через точку.
Если значение уже не объект, остается только сам ключ. Потом все варианты собираются в один union.
В итоге получается полный список допустимых строковых путей по всей структуре.

---

## Numbers

В TypeScript математика на типах обычно делается не над самим `number`, а над его представлением.
Самый популярный прием: представить число как длину кортежа. Если кортеж содержит 3 элемента, то на уровне типов это и
есть "число 3".
Отсюда появляется базовая ментальная модель: вместо арифметики над числами мы делаем преобразования над массивами
фиксированной длины.

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
Этим и пользуются type-level паттерны. Мы кодируем число не как "математическую сущность", а как "количество элементов в
массиве".
Вспомните школу, как вы учились считать в первом классе? С помощью счетных палочек. Кучка из двух палочек, кучка из трех
палочек, сложили в одну кучку, посчитали - пять палочек! (нам кстати на линейной алгебре на первом курсе так же
расказывали)
Дальше любую операцию можно свести к операциям над кортежами: склеить два массива, убрать хвост, добавить элемент,
сравнить длины.
Поэтому почти вся математика на типах крутится вокруг `["length"]`.

## Math operations

Когда число представлено как кучка палочек, арифметические операции начинают выглядеть очень естественно.
Сложение превращается в складывание двух кучек в одну, вычитание в удалении нескольких палочек из кучи, а умножение
превращается в повторяющееся сложение.
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
Вычитание работает через pattern matching: если кортеж длины `A` можно разложить как "сначала кортеж длины `B`, а потом
хвост", значит длина хвоста и есть `A - B`.
Инкремент и декремент это просто добавление или удаление одного элемента.
Умножение обычно строят как повторяющееся сложение: на каждом шаге добавляют в аккумулятор еще `A` элементов, пока
счетчик `B` не дойдет до нуля.
Все это работает, потому что TypeScript умеет сохранять точные длины кортежей и рекурсивно вычислять условные типы.

## Compare numbers

Сравнение чисел на типах тоже делается через кортежи. Идея простая: если кортеж длины `A` можно представить как "кортеж
длины `B` плюс еще какой-то хвост", значит `A >= B`.
Из таких проверок строятся `>`, `<`, `<=`, `>=`, а если нужен более подробный результат, можно сравнивать два кортежа
рекурсивно и получать `"less" | "equal" | "greater"`.

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

В проверке `A >= B` TypeScript пытается понять, можно ли кортеж длины `A` представить как "кортеж длины `B` и еще
какой-то хвост".
Если можно, значит `A` не меньше `B`. Для строгого `>` добавляют отдельную проверку на равенство, потому что `>=` для
одинаковых чисел тоже даст `true`.
Рекурсивное сравнение работает еще прямолинейнее: мы одновременно снимаем по одному элементу с обоих кортежей.
Если один закончился раньше, он меньше. Если оба закончились одновременно, они равны. Поэтому вся логика сравнения снова
сводится к тому, как TypeScript умеет сопоставлять кортежи по форме.
