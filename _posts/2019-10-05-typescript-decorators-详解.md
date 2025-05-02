---
layout: post
title: "一篇搞懂 TypeScript 装饰器"
date: 2019-10-05 20:11:00 +0800
categories: Code
tags: [Typescript, Decorator]
comments: true
---

如果你写过 Angular、NestJS、TypeORM，肯定见过这些东西：

```ts
@Component({...})
class AppComponent {}

@Injectable()
class UserService {}

@Entity()
class User {}
```

这些前面带 `@` 的小标签，就是 **Decorator（装饰器）**。它们看起来有点魔法：
加一行注解，类和方法突然多了能力。

这篇文章，主要围绕三点：

1. 讲清楚 **装饰器到底是什么**、在 TypeScript 里怎么启用
2. 按类别把 **五种装饰器用法** 说透：类 / 方法 / 访问器 / 属性 / 参数
3. 结合几个完整例子，看看它们在真实项目里能干什么

## 装饰器到底是什么？

用一句最朴素的话概括：

**装饰器就是一个函数**，在类定义阶段被调用，用来“观察或改造”类、方法、属性或参数。

TypeScript 官方文档的定义是：装饰器是一种特殊的声明，可以附加在类声明、方法、访问器、属性或参数上，
语法是 `@expression`，其中 `expression` 必须返回一个函数，这个函数会在运行时被调用。

简单理解就是：

- 写代码时：你在某个声明前面加上 `@XXX(...)`
- 编译后跑起来：TypeScript 会在类被创建时，**自动调用**对应的函数，把“被修饰的东西”当参数传进去
- 在函数里，你可以：

  - 改写类或原型
  - 修改方法实现
  - 调整属性描述符（`enumerable`、`configurable` 等）
  - 记录/附加一些元数据（metadata），供运行时使用

### Decorator 可以修饰哪些东西？

TypeScript 支持这几类：

- **类装饰器**（Class Decorator）
- **方法装饰器**（Method Decorator）
- **访问器装饰器**（Accessor Decorator：`get` / `set`）
- **属性装饰器**（Property Decorator）
- **参数装饰器**（Parameter Decorator）


## 启用装饰器

默认情况下，TypeScript 编译器是不会让你随便用装饰器的，需要在 `tsconfig.json` 中显式打开。

最常见的配置是（legacy 装饰器模式）：

```jsonc
{
  "compilerOptions": {
    "target": "ES6",
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

- `experimentalDecorators: true`

  启用 **旧版（实验性）装饰器语法**

  Angular / NestJS / TypeORM 这类传统生态基本都是靠它

- `emitDecoratorMetadata: true`（可选）

  让编译器额外生成一些类型元数据（需要搭配 `reflect-metadata` 使用）

  典型用途：依赖注入框架需要知道构造函数参数的类型，自动帮你 new 对象

如果用了装饰器而没开 `experimentalDecorators`，你可能见过这个经典报错：

> “Experimental support for decorators is a feature that is subject to change…”

打开上面两个选项就好了。

## Decorator 的基本形态

先从一个最小例子开始：

```ts
function sealed(target: Function) {
  Object.seal(target);
  Object.seal(target.prototype);
}

@sealed
class Greeter {
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  greet() {
    return "Hello, " + this.greeting;
  }
}
```

`@sealed` 做了什么？

- `@sealed` 其实就是在类定义完之后，调用 `sealed(Greeter)`
- 在 `sealed` 函数里面，我们用 `Object.seal` 让：

  - `Greeter` 这个构造函数本身
  - `Greeter.prototype` 原型对象
    都变成 **不可扩展**：不能再往上面随便加新属性

这是一个很典型的【类装饰器 + 元编程】小例子——只需要一行 `@sealed`，就改变了这个类的“可塑性”。

### 装饰器工厂（Decorator Factory）

很多时候我们希望装饰器还能“带参数”，比如：

```ts
@color('red')
class Button {...}
```

这时就会用到 **装饰器工厂**：

```ts
function color(value: string) {
  // 工厂：接收配置，返回真正的装饰器函数
  return function (target: Function) {
    // 在这里既能访问到 target，也能看到 value
    Reflect.defineMetadata("color", value, target);
  };
}

@color("red")
class Button {}
```

你可以把它理解为两层函数：

- 最外层：**配置阶段** —— 你写 `@color('red')` 时就执行了
- 返回的函数：**装饰阶段** —— 编译后的 JS 会在类定义后调用它，把 `Button` 传进去

## 类装饰器

类装饰器的函数签名（legacy 模式）很简单：

```ts
type ClassDecorator = (constructor: Function) => void | Function;
```

如果你 **返回一个新的构造函数**，就等于“改造了这个类”。

### 给类加属性

```ts
function classDecorator<T extends { new (...args: any[]): {} }>(
  constructor: T
) {
  return class extends constructor {
    newProperty = "new property";
    hello = "override";
  };
}

@classDecorator
class Greeter {
  property = "property";
  hello: string;

  constructor(m: string) {
    this.hello = m;
  }
}

console.log(new Greeter("world"));
```

这里发生了什么？

1. 原始 `Greeter` 只有 `property` 和 `hello`
2. `@classDecorator` 返回了一个 **继承自原 Greeter 的匿名子类**
3. 这个子类额外加了 `newProperty`，并把 `hello` 的默认值改成 `"override"`

所以最终是 `new Greeter("world")` 拿到的实例，其实是这个“增强版子类”的实例。

### 冻结类，禁止继承

接下来是一个 `Frozen` 的例子，先看一眼：

```ts
function Frozen(constructor: Function) {
  Object.freeze(constructor);
  Object.freeze(constructor.prototype);
}

@Frozen
class IceCream {}

console.log(Object.isFrozen(IceCream)); // true

class FroYo extends IceCream {} // 会因为被冻结而报错
```

这个装饰器的用途很直白：**让类和原型都变成只读**。
在某些严格模式或辅助工具下，这可以帮助你避免别人后续给类乱打补丁。

类装饰器的常见场景：

- 注册依赖注入容器（DI）：`@Injectable()`、`@Controller()`
- 声明实体模型：`@Entity()`、`@Schema()`
- 给类附加一些元信息：权限级别、模块标识等

Angular 文档里就提到，`@Injectable()` 这个装饰器用于声明一个类可被注入，并配合元数据生成依赖注入需要的信息。

## 方法装饰器

方法装饰器的签名是：

```ts
type MethodDecorator = (
  target: any,
  propertyKey: string | symbol,
  descriptor: PropertyDescriptor
) => void | PropertyDescriptor;
```

重点是第三个参数：`descriptor`。你可以改写 `descriptor.value`，相当于**替换这个方法本身**。

### 控制可枚举性（@enumerable）

```ts
function enumerable(value: boolean) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    descriptor.enumerable = value;
  };
}

class Greeter {
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  @enumerable(false)
  greet() {
    return "Hello, " + this.greeting;
  }
}
```

这里我们改写了 `descriptor.enumerable`，就决定了这个方法在 `for...in` 里要不要被遍历到。

### 二次确认弹窗（@Confirmable）

接下来的 `Confirmable` 装饰器是个非常有意思的例子：

```ts
function Confirmable(message: string) {
  return function (
    target: Object,
    key: string | symbol,
    descriptor: PropertyDescriptor
  ) {
    const original = descriptor.value;

    descriptor.value = function (...args: any[]) {
      const allow = confirm(message);

      if (allow) {
        return original.apply(this, args);
      } else {
        return null;
      }
    };

    return descriptor;
  };
}

class IceCreamComponent {
  toppings: string[] = [];

  @Confirmable("Are you sure?")
  @Confirmable("Are you super, super sure? There is no going back!")
  addTopping(topping: string) {
    this.toppings.push(topping);
  }
}
```

这个装饰器做了几件事：

1. 保存原方法 `original`
2. 用一个新函数替换它，先弹 `confirm` 再决定要不要继续执行
3. 支持多次叠加装饰：多个 `@Confirmable` 会按 **自下而上** 的顺序包裹

这种写法非常适合下面这些场景：

- 权限/角色检查（`@RequireRole('admin')`）
- 日志和埋点（`@Log()`）
- 重试机制（`@Retry(3)`）
- 性能统计（`@Time('getUser')`）

## 访问器装饰器

访问器装饰器长得跟方法装饰器差不多，也是拿着 `descriptor` 来改。

```ts
function configurable(value: boolean) {
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    descriptor.configurable = value;
  };
}

class Point {
  private _x: number;
  private _y: number;

  constructor(x: number, y: number) {
    this._x = x;
    this._y = y;
  }

  @configurable(false)
  get x() {
    return this._x;
  }

  @configurable(false)
  get y() {
    return this._y;
  }
}
```

这里我们把 `configurable` 设为 `false`，相当于说：

“以后别再试图 `delete` 或 `defineProperty` 改写这些 getter 了！”

访问器装饰器常用场景不算多，但在做一些“只读视图”或安全相关的代码时会挺方便。

## 属性装饰器

属性装饰器的函数签名比前面都朴素：

```ts
type PropertyDecorator = (target: any, propertyKey: string | symbol) => void;
```

注意：**因为拿不到 `descriptor`**，所以你不能直接通过这个签名来改 getter/setter。

### 用 Reflect.metadata 存格式字符串

```ts
import "reflect-metadata";

const formatMetadataKey = Symbol("format");

function format(formatString: string) {
  return Reflect.metadata(formatMetadataKey, formatString);
}

function getFormat(target: any, propertyKey: string) {
  return Reflect.getMetadata(formatMetadataKey, target, propertyKey);
}

class Greeter {
  @format("Hello, %s")
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  greet() {
    const formatString = getFormat(this, "greeting");
    return formatString.replace("%s", this.greeting);
  }
}
```

这里我们做了一个简单的小型“注解系统”：

- `@format("Hello, %s")` 把格式字符串存成元数据
- 运行时通过 `Reflect.getMetadata` 取回来用

这种元数据 + 装饰器的组合，是很多 ORM / DI / 校验类库的基础。

### 自动包 Emoji

```ts
function Emoji() {
  return function (target: any, key: string | symbol) {
    let val = target[key];

    const getter = () => val;

    const setter = (next: string) => {
      console.log("updating flavor...");
      val = `🍦 ${next} 🍦`;
    };

    Object.defineProperty(target, key, {
      get: getter,
      set: setter,
      enumerable: true,
      configurable: true,
    });
  };
}

class IceCreamComponent {
  @Emoji()
  flavor = "vanilla";
}

const ic = new IceCreamComponent();
ic.flavor = "chocolate";
console.log(ic.flavor); // 🍦 chocolate 🍦
```

这里我们自己动手 `defineProperty`，相当于给属性套上了一个定制化的 getter/setter：

- 设置时自动加 emoji
- 读取时拿到包装好的结果

类字段在编译后的初始化顺序 + 装饰器执行顺序细节，会影响具体行为，
实战时建议配合 TypeScript Playground 多试几次。

## 参数装饰器

参数装饰器的签名是：

```ts
type ParameterDecorator = (
  target: Object,
  propertyKey: string | symbol,
  parameterIndex: number
) => void;
```

你会拿到：

- `target`：类的原型对象（对实例方法来说）
- `propertyKey`：方法名
- `parameterIndex`：参数在形参列表里的索引

本身它不能改方法行为，常见用法是：**记录参数信息**，配合同一方法上的“方法装饰器”一起玩。

看你给的 `@required + @validate` 组合：

```ts
import "reflect-metadata";

const requiredMetadataKey = Symbol("required");

function required(
  target: Object,
  propertyKey: string | symbol,
  parameterIndex: number
) {
  const existingRequiredParameters: number[] =
    Reflect.getOwnMetadata(requiredMetadataKey, target, propertyKey) || [];

  existingRequiredParameters.push(parameterIndex);

  Reflect.defineMetadata(
    requiredMetadataKey,
    existingRequiredParameters,
    target,
    propertyKey
  );
}

function validate(
  target: any,
  propertyName: string,
  descriptor: TypedPropertyDescriptor<Function>
) {
  const method = descriptor.value!;

  descriptor.value = function (...args: any[]) {
    const requiredParameters: number[] =
      Reflect.getOwnMetadata(requiredMetadataKey, target, propertyName) || [];

    for (const index of requiredParameters) {
      if (index >= args.length || args[index] === undefined) {
        throw new Error("Missing required argument.");
      }
    }

    return method.apply(this, args);
  };
}

class Greeter {
  greeting: string;

  constructor(message: string) {
    this.greeting = message;
  }

  @validate
  greet(@required name: string) {
    return `Hello ${name}, ${this.greeting}`;
  }
}
```

这里的“套路”非常经典：

1. `@required`（参数装饰器）把“第几个参数必填”记录到元数据里
2. `@validate`（方法装饰器）在真正执行方法前，读取元数据，检查对应参数是否传了
3. 没传就抛错，传了就放行

这就是很多验证框架 / 路由参数解析库常用的写法模型。

## 装饰器的执行顺序

再往深入一点，装饰器是有执行顺序的（以 legacy 装饰器为例）：

1. **如果一个位置有多个装饰器**，会先从上到下 **求值表达式**，然后 **从下到上** 调用装饰器函数

   `@A @B method()` → 先算出 `A` 和 `B`，再按 `B` → `A` 的顺序调用

2. 整个类里，执行顺序 roughly 是：

   - 参数装饰器（每个方法/构造函数参数）
   - 方法装饰器、访问器装饰器
   - 属性装饰器
   - 最后才是类装饰器

通常你只要记住一件事：**距离被修饰对象最近的装饰器最后执行**，就能解释大多数链式效果。

## Decorator 在项目里的常见用法

了解语法之后，更关键的是：**在项目里它能帮你干掉哪些“重复劳动”？**

实际工程中，装饰器主要解决几类问题：

1. **依赖注入 / IOC**

   - `@Injectable()`、`@Service()`：把类注册进容器
   - `@Inject()` / `@Autowired()`：标记构造参数或属性需要注入

2. **数据建模 / ORM 映射**

   - `@Entity()`、`@Column()`：描述表结构
   - `@PrimaryKey()`、`@Index()` 等：附加数据库约束

3. **Web 框架路由**

   - `@Controller('/users')`、`@Get('/:id')`、`@Post('/')`

4. **校验与转换**

   - `@Required()`、`@Length(1, 20)`、`@IsEmail()` 等

5. **横切逻辑（AOP）**

   - 日志、缓存、性能统计、权限校验、事务控制……

现在看到的这些例子（`@sealed`、`@Frozen`、`@Emoji`、`@Confirmable`、`@required + @validate`），
基本覆盖了装饰器能玩的主流套路。剩下的就是在自己项目里把它们组装成框架的问题了。

## 写装饰器时的几个建议

最后补几条实践经验向的小提示，省你踩坑：

1. **别把业务逻辑都塞进装饰器里**

   装饰器负责“包装”和“注册”，具体业务逻辑还是留在类/方法本身

2. **注意 this / 原方法绑定**

   方法装饰器里改写 `descriptor.value` 时，记得用 `original.apply(this, args)`

3. **搞清楚运行时 vs 编译时**

   装饰器是在类定义时执行的，不是每次调用都会跑一遍初始化

4. **慎重依赖 emitDecoratorMetadata**

   这个选项目前依然被标为 experimental，生态也在往“标准装饰器 + 自己生成 metadata”的方向走

5. **调试时多用 console.log**

   想弄清楚 `target`/`propertyKey`/`descriptor` 到底长什么样，最好就是手写几个小例子在 Playground 跑一跑

## 参考材料

- [Documentation - Decorators](https://www.typescriptlang.org/docs/handbook/decorators.html)
- [Documentation - TypeScript 5.0](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-0.html)
- [TSConfig Option: emitDecoratorMetadata](https://www.typescriptlang.org/tsconfig/emitDecoratorMetadata.html)
- [Modular design with dependency injection](https://angular.dev/essentials/dependency-injection)
- [A practical guide to TypeScript decorators](https://blog.logrocket.com/practical-guide-typescript-decorators/)
