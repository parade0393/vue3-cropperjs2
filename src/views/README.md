## any、unknown、never 的对比

| 类型      | 含义                       | 可赋值方向                               | 用途和特点                                               |
| --------- | -------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| `any`     | 任意类型，完全放弃类型检查 | 可赋值给任何类型，也可接受任何值         | 用于快速迁移老代码、第三方库场景，不建议大量使用         |
| `unknown` | 不确定类型，更安全的 `any` | 可接受任何值，使用前必须先进行类型检查   | 用于类型安全场合，比如`JSON.parse()`，避免类型污染       |
| `never`   | 永远不会出现值的类型       | 不能赋值为其他类型，只能接受没有值的场合 | 用于错误处理、无限循环、类型收窄，表示代码不会正常return |

简单示例

```typescript
let a: any = 123
let u: unknown = 'hello'
let n: never

a = true // ✅ any接受任何值
a.someMethod() // ✅ any调用不会检查

// u = 123;            ✅ unknown可以接受值
// u.someMethod();      ❌ 报错，unknown不能调用属性

function error(): never {
  throw new Error('This function never returns')
}
```

## never vs never[] 的重要区别

这是最容易混淆的概念！

### never 作为具体类型

```typescript
// never 表示"不存在"，不能有值
let value: never // 不能给它赋任何值

// never[] 表示"不存在的数组"
let arr: never[] = [] // 只能是空数组，不能添加元素
// arr.push(1); // ❌ 错误！不能添加任何元素
```

### never[] 在泛型约束中的特殊含义

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

### 为什么 never[] 可以匹配任何参数？

类型理论解释：

1. never 是所有类型的子类型（底类型）
2. 在函数参数位置（逆变位置），子类型可以安全地替换超类型
3. 因此 (...args: never[]) 可以"兼容"任何参数列表
4. 这是类型系统的巧妙设计，让我们能表达"任意函数签名"

## never[]的实际应用场景

### 函数防抖（Debounce）

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

## 关于协变和逆变

### 简单说

1. 协变
   - 协变：方向相同，子类型之间是顺着方向走。
   - 比如：如果 Dog 是 Animal 的子类型，那么`Array<Dog>` 就是 `Array<Animal>` 的子类型。
   - 用途场合：**多数是“读值”** 的地方。
2. 逆变
   - 比如：如果 Dog 是 Animal 的子类型，那么Handler<Animal> 就是 Handler<Dog> 的子类型。
   - 用途场合：**多数是“写值”**或参数类型的位置。

### 用实际例子说

我们说「狗是动物」，代码里：

```ts
class Animal {}
class Dog extends Animal {}
```

- 协变场景：数组读取

```ts
let dogs: Dog[] = [new Dog()]
let animals: Animal[] = dogs

animals.forEach((a) => {
  // 我只需要 Animal 类型，用 Dog 对我也没问题。
})
```

- 逆变场景：参数传递

```ts
function handleAnimal(a: Animal) {
  console.log(a)
}
function handleDog(d: Dog) {
  console.log(d)
}

// 你期望一个 Animal handler，我给你一个 Dog handler，行不行？
// 👇 按照直觉是不行的  =等号左边可以理解为实参，右边可以理解为行参。执行animalHandler实际执行的是handleDog,因为handleDog只接受dog，但由于animalHandler却可以接受dog之外的animal，所以对于实际执行的函数handleDog是不可接受的
let animalHandler: (a: Animal) => void = handleDog // ❌
```

为啥？因为：

我说：「我接受 Animal 类型」

但实际提供的 handler：「只能接受 Dog 类型」，那如果以后传来一只 Cat，handleDog就崩溃了。
但反过来：

```ts
let dogHandler: (d: Dog) => void = handleAnimal // ✅
```

因为：

我说：「我接受 Dog」

但实际提供的 handler：「能接受 Animal」，那接受 Dog 完全没问题。

这种参数方向反过来的类型关系，就是逆变。

换一个工作中的例子形象的说明
假如`animalHandler`是老板,`handleAnimal`是员工，如果老板接项目的时候，向客户夸下了海口(左边类型大，既可以是Dog，又可以是Cat)，超出了员工的能力范围(右边类型小，实际只是个Dog)，那项目肯定是要蹦的。但是反过来，如果老板接项目的时候，重复考虑到员工的情况，尽量安排员工能力之内的项目，那项目肯定顺丰顺水
