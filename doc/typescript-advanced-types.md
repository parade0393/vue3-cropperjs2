## 1. any、unknown、never 的对比

| 类型      | 含义                       | 可赋值方向                               | 用途和特点                                               |
| --------- | -------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| `any`     | 任意类型，完全放弃类型检查 | 可赋值给任何类型，也可接受任何值         | 用于快速迁移老代码、第三方库场景，不建议大量使用         |
| `unknown` | 不确定类型，更安全的 `any` | 可接受任何值，使用前必须先进行类型检查   | 用于类型安全场合，比如`JSON.parse()`，避免类型污染       |
| `never`   | 永远不会出现值的类型       | 不能赋值为其他类型，只能接受没有值的场合 | 用于错误处理、无限循环、类型收窄，表示代码不会正常return |

### 1.1 any 类型：完全放弃类型检查的双刃剑

`any` 类型是 TypeScript 中最灵活的类型，它允许你对该类型的值进行任何操作，同时也会绕过类型检查。

```typescript
// any 类型示例
let anyValue: any = 42
anyValue = 'hello'
anyValue = true
anyValue = { name: '张三' }

// 可以对 any 类型进行任何操作，不会有类型检查错误
anyValue.foo()
anyValue.bar.baz
anyValue[0]
console.log(anyValue + 1)
```

**特点**：

- 可以赋值给任何类型（除了 never）
- 可以被任何类型赋值
- 对其进行任何操作都不会有类型检查错误
- 使用 `any` 会失去 TypeScript 的类型安全保障

### 1.2 unknown 类型：更安全的 any

`unknown` 类型是 TypeScript 3.0 引入的类型，它比 `any` 更安全。它表示一个未知的类型，你可以将任何值赋给它，但在使用前必须进行类型检查或类型断言。

```typescript
// unknown 类型示例
let unknownValue: unknown = 42
unknownValue = 'hello'
unknownValue = true
unknownValue = { name: '李四' }

// 错误：不能直接对 unknown 类型进行操作
// unknownValue.foo();
// unknownValue.bar.baz;
// unknownValue[0];
// console.log(unknownValue + 1);

// 正确：使用类型检查后可以操作
if (typeof unknownValue === 'string') {
  console.log(unknownValue.toUpperCase())
}

// 正确：使用类型断言
const value = unknownValue as string
console.log(value.toUpperCase())
```

**特点**：

- 可以被任何类型赋值
- 不能直接赋值给除了 `any` 和 `unknown` 以外的类型
- 不能直接对其进行操作，必须先进行类型检查或类型断言
- 比 `any` 更安全，因为它强制你在使用前进行类型检查

### 1.3 never 类型：表示永不存在的值

`never` 类型表示永远不会发生的值的类型。它通常用于表示函数永远不会返回（如抛出异常或无限循环）或者是不可能存在的类型。

```typescript
// 永远不会返回的函数（抛出异常）
function throwError(message: string): never {
  throw new Error(message)
}

// 永远不会返回的函数（无限循环）
function infiniteLoop(): never {
  while (true) {}
}

// 不可能存在的类型（条件类型中的排除）
type NonString<T> = T extends string ? never : T
type Result = NonString<string | number | boolean> // number | boolean
```

**特点**：

- `never` 是所有类型的子类型（底类型）
- 没有类型是 `never` 的子类型（除了 `never` 自身）
- 不能将任何值赋给 `never` 类型（除了 `never` 类型的值）
- 通常用于表示永远不会发生的情况

## 2. never 和 never[] 的区别：最容易混淆的概念

### 2.1 never 类型：不能有值的类型

`never` 类型表示永远不会发生的值的类型。它是一个"底部类型"（bottom type），意味着它可以赋值给任何其他类型，但没有任何值可以赋值给它（除了 `never` 类型的值）。

```typescript
// never 类型不能有值
let neverValue: never
// 错误：不能将任何值赋给 never 类型
// neverValue = 42;
// neverValue = "hello";

// 正确：可以赋值给任何类型
let anyVal: any = neverValue
let numVal: number = neverValue
let strVal: string = neverValue
```

### 2.2 never[] 类型：只能为空的数组

`never[]` 是一个数组类型，其元素类型为 `never`。由于 `never` 类型不能有值，所以 `never[]` 实际上表示一个永远为空的数组。

```typescript
// never[] 类型只能是空数组
let neverArray: never[] = []

// 错误：不能向 never[] 添加任何元素
// neverArray.push(42);
// neverArray.push("hello");

// 正确：可以进行数组操作
console.log(neverArray.length) // 0
neverArray = neverArray.concat([]) // 仍然是空数组
```

**never 和 never[] 的关键区别**：

1. `never` 是一个不能有值的类型，而 `never[]` 是一个只能为空的数组类型
2. `never` 不能被赋值，而 `never[]` 可以被赋值为空数组 `[]`
3. `never[]` 可以进行数组操作，但不能添加任何元素

### 2.3 never[] 在泛型约束中的特殊含义

```typescript
// 在泛型约束中，never[] 变成了"万能匹配符"
function wrapper<T extends (...args: never[]) => unknown>(fn: T) {
  return (...args: Parameters<T>) => {
    console.log('调用函数')
    return fn(...args)
  }
}

// 可以传入任何函数！
const add = (a: number, b: number) => a + b
const greet = (name: string) => `Hello ${name}`
const noArgs = () => 'no arguments'

const wrappedAdd = wrapper(add) // ✅ 类型：(a: number, b: number) => number
const wrappedGreet = wrapper(greet) // ✅ 类型：(name: string) => string
const wrappedNoArgs = wrapper(noArgs) // ✅ 类型：() => string
```

### 2.4 为什么 never[] 可以匹配任何参数？

类型理论解释：

1. never 是所有类型的子类型（底类型）
2. 在函数参数位置（逆变位置），子类型可以安全地替换超类型
3. 因此 (...args: never[]) 可以"兼容"任何参数列表
4. 这是类型系统的巧妙设计，让我们能表达"任意函数签名"

### 2.5 never[]的实际应用场景：函数防抖（Debounce）

```typescript
const debounce = function <T extends (...args: never[]) => unknown>(
  func: T,
  wait: number = 1000,
  immediate: boolean = true,
): (...args: Parameters<T>) => void {
  let timer: number | null = null

  return function (this: ThisParameterType<T>, ...args: Parameters<T>): void {
    if (timer) clearTimeout(timer)

    if (immediate) {
      const callNow = !timer
      timer = setTimeout(() => {
        timer = null
      }, wait)
      if (callNow) func.apply(this, args)
    } else {
      timer = setTimeout(() => {
        func.apply(this, args)
      }, wait)
    }
  }
}
```

## 3. 泛型的协变和逆变：类型关系的方向性

### 3.1 什么是变型（Variance）

变型描述了类型之间的子类型关系如何影响包含这些类型的复杂类型之间的子类型关系。

假设我们有类型 A 和 B，且 A 是 B 的子类型（记作 A <: B）。那么：

- **协变（Covariant）**：如果 `F<A>` 是` F<B>` 的子类型，即 `F<A> <: F<B>`，则 F 是协变的
- **逆变（Contravariant）**：如果 `F<B>` 是 `F<A>` 的子类型，即 `F<B> <: F<A>`，则 F 是逆变的
- **不变（Invariant）**：如果 `F<A>` 和 `F<B>` 之间没有子类型关系，则 F 是不变的

### 3.2 通俗理解变型

#### 简单说

1. **协变**

   - 方向相同，子类型之间是顺着方向走
   - 比如：如果 Dog 是 Animal 的子类型，那么`Array<Dog>` 就是 `Array<Animal>` 的子类型
   - 用途场合：**多数是"读值"** 的地方

2. **逆变**
   - 方向相反，子类型之间是逆着方向走
   - 比如：如果 Dog 是 Animal 的子类型，那么Handler<Animal> 就是 Handler<Dog> 的子类型
   - 用途场合：**多数是"写值"**或参数类型的位置

#### 用实际例子说明

我们说「狗是动物」，代码里：

```typescript
class Animal {}
class Dog extends Animal {}
```

##### 协变场景：数组读取

```typescript
let dogs: Dog[] = [new Dog()]
let animals: Animal[] = dogs

animals.forEach((a) => {
  // 我只需要 Animal 类型，用 Dog 对我也没问题
})
```

##### 逆变场景：参数传递

```typescript
function handleAnimal(a: Animal) {
  console.log(a)
}
function handleDog(d: Dog) {
  console.log(d)
}

// 你期望一个 Animal handler，我给你一个 Dog handler，行不行？
// 👇 按照直觉是不行的
let animalHandler: (a: Animal) => void = handleDog // ❌
```

为什么不行？因为：

我说：「我接受 Animal 类型」

但实际提供的 handler：「只能接受 Dog 类型」，那如果以后传来一只 Cat，handleDog就崩溃了。

但反过来是可以的：

```typescript
let dogHandler: (d: Dog) => void = handleAnimal // ✅
```

因为：

我说：「我接受 Dog」

但实际提供的 handler：「能接受 Animal」，那接受 Dog 完全没问题。

这种参数方向反过来的类型关系，就是逆变。

#### 工作中的形象比喻

假如`animalHandler`是老板，`handleAnimal`是员工：

- 如果老板接项目的时候，向客户夸下了海口（左边类型大，既可以是Dog，又可以是Cat），超出了员工的能力范围（右边类型小，实际只是个Dog），那项目肯定是要崩的。
- 但是反过来，如果老板接项目的时候，充分考虑到员工的情况，尽量安排员工能力之内的项目，那项目肯定顺风顺水。

### 3.3 为什么需要变型

变型保证了类型系统的安全性和一致性：

- **协变**确保我们可以在期望父类型的地方使用子类型，这符合"里氏替换原则"
- **逆变**确保函数参数的类型安全，防止在函数内部调用子类型特有的方法而导致错误

### 3.4 TypeScript 中的变型规则

1. **数组、对象和返回值类型是协变的**：

   ```typescript
   // 对象属性是协变的
   type 动物容器 = { 宠物: 动物 }
   type 狗容器 = { 宠物: 狗 }
   let 狗盒子: 狗容器 = { 宠物: new Dog() }
   let 动物盒子: 动物容器 = 狗盒子 // 正确：协变

   // 函数返回值是协变的
   type 返回动物 = () => 动物
   type 返回狗 = () => 狗
   let 获取狗: 返回狗 = () => new Dog()
   let 获取动物: 返回动物 = 获取狗 // 正确：返回值是协变的
   ```

2. **函数参数是逆变的**（在 `strictFunctionTypes` 开启时）：

   ```typescript
   type 喂动物 = (动物: 动物) => void
   type 喂狗 = (狗: 狗) => void

   const 喂任何动物: 喂动物 = (动物) => {
     /* 处理动物 */
   }
   const 专门喂狗: 喂狗 = (狗) => {
     /* 处理狗 */
   }

   let 喂狗的函数: 喂狗 = 喂任何动物 // 正确：逆变
   // let 喂动物的函数: 喂动物 = 专门喂狗; // 错误：不符合逆变规则
   ```

3. **泛型类型通常是不变的**，除非它们在特定位置使用类型参数：

   ```typescript
   // 泛型类是不变的
   class 盒子<T> {
     内容: T
     constructor(内容: T) {
       this.内容 = 内容
     }
   }

   let 狗盒: 盒子<狗> = new 盒子(new Dog())
   // let 动物盒: 盒子<动物> = 狗盒; // 错误：盒子<狗> 不是 盒子<动物> 的子类型
   ```
