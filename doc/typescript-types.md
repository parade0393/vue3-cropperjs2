# TypeScript 中的特殊类型：any、unknown、never 及泛型变型

## 1. any、unknown 和 never 的区别

### 1.1 any 类型

`any` 类型是 TypeScript 中最灵活的类型，它允许你对该类型的值进行任何操作，同时也会绕过类型检查。

```typescript
// any 类型示例
let anyValue: any = 42;
anyValue = "hello";
anyValue = true;
anyValue = { name: "张三" };

// 可以对 any 类型进行任何操作，不会有类型检查错误
anyValue.foo();
anyValue.bar.baz;
anyValue[0];
console.log(anyValue + 1);
```

**特点**：
- 可以赋值给任何类型（除了 never）
- 可以被任何类型赋值
- 对其进行任何操作都不会有类型检查错误
- 使用 `any` 会失去 TypeScript 的类型安全保障

### 1.2 unknown 类型

`unknown` 类型是 TypeScript 3.0 引入的类型，它比 `any` 更安全。它表示一个未知的类型，你可以将任何值赋给它，但在使用前必须进行类型检查或类型断言。

```typescript
// unknown 类型示例
let unknownValue: unknown = 42;
unknownValue = "hello";
unknownValue = true;
unknownValue = { name: "李四" };

// 错误：不能直接对 unknown 类型进行操作
// unknownValue.foo();
// unknownValue.bar.baz;
// unknownValue[0];
// console.log(unknownValue + 1);

// 正确：使用类型检查后可以操作
if (typeof unknownValue === "string") {
  console.log(unknownValue.toUpperCase());
}

// 正确：使用类型断言
const value = unknownValue as string;
console.log(value.toUpperCase());
```

**特点**：
- 可以被任何类型赋值
- 不能直接赋值给除了 `any` 和 `unknown` 以外的类型
- 不能直接对其进行操作，必须先进行类型检查或类型断言
- 比 `any` 更安全，因为它强制你在使用前进行类型检查

### 1.3 never 类型

`never` 类型表示永远不会发生的值的类型。它通常用于表示函数永远不会返回（如抛出异常或无限循环）或者是不可能存在的类型。

```typescript
// 永远不会返回的函数（抛出异常）
function throwError(message: string): never {
  throw new Error(message);
}

// 永远不会返回的函数（无限循环）
function infiniteLoop(): never {
  while (true) {}
}

// 不可能存在的类型（条件类型中的排除）
type NonString<T> = T extends string ? never : T;
type Result = NonString<string | number | boolean>; // number | boolean
```

**特点**：
- `never` 是所有类型的子类型
- 没有类型是 `never` 的子类型（除了 `never` 自身）
- 不能将任何值赋给 `never` 类型（除了 `never` 类型的值）
- 通常用于表示永远不会发生的情况

## 2. never 和 never[] 的区别

### 2.1 never 类型

如前所述，`never` 类型表示永远不会发生的值的类型。它是一个"底部类型"（bottom type），意味着它可以赋值给任何其他类型，但没有任何值可以赋值给它（除了 `never` 类型的值）。

```typescript
// never 类型不能有值
let neverValue: never;
// 错误：不能将任何值赋给 never 类型
// neverValue = 42;
// neverValue = "hello";

// 正确：可以赋值给任何类型
let anyVal: any = neverValue;
let numVal: number = neverValue;
let strVal: string = neverValue;
```

### 2.2 never[] 类型

`never[]` 是一个数组类型，其元素类型为 `never`。由于 `never` 类型不能有值，所以 `never[]` 实际上表示一个永远为空的数组。

```typescript
// never[] 类型只能是空数组
let neverArray: never[] = [];

// 错误：不能向 never[] 添加任何元素
// neverArray.push(42);
// neverArray.push("hello");

// 正确：可以进行数组操作
console.log(neverArray.length); // 0
neverArray = neverArray.concat([]); // 仍然是空数组
```

**never 和 never[] 的关键区别**：

1. `never` 是一个不能有值的类型，而 `never[]` 是一个只能为空的数组类型
2. `never` 不能被赋值，而 `never[]` 可以被赋值为空数组 `[]`
3. `never[]` 可以进行数组操作，但不能添加任何元素

**实际应用场景**：

`never[]` 在实际开发中很少直接使用，但在条件类型和类型操作中很有用：

```typescript
// 过滤数组中的某些类型
type FilterString<T> = T extends string ? never : T;
type StringsRemoved = FilterString<string | number | boolean>[];
// 等价于 (number | boolean)[]，即不包含字符串的数组
```

## 3. 泛型的协变和逆变

### 3.1 什么是变型（Variance）

变型描述了类型之间的子类型关系如何影响包含这些类型的复杂类型之间的子类型关系。

假设我们有类型 A 和 B，且 A 是 B 的子类型（记作 A <: B）。那么：

- **协变（Covariant）**：如果 F<A> 是 F<B> 的子类型，即 F<A> <: F<B>，则 F 是协变的
- **逆变（Contravariant）**：如果 F<B> 是 F<A> 的子类型，即 F<B> <: F<A>，则 F 是逆变的
- **不变（Invariant）**：如果 F<A> 和 F<B> 之间没有子类型关系，则 F 是不变的

### 3.2 通俗理解变型

让我们用一个简单的例子来理解：

假设我们有：
- `动物` 类型
- `狗` 类型（是 `动物` 的子类型）

#### 协变示例

数组类型在 TypeScript 中是协变的：

```typescript
// 定义类型关系
type 动物 = { 吃(): void };
type 狗 = { 吃(): void; 汪汪(): void };

// 狗 是 动物 的子类型
let 一只狗: 狗 = { 吃() {}, 汪汪() {} };
let 一只动物: 动物 = 一只狗; // 正确：子类型可以赋值给父类型

// 数组是协变的
let 狗数组: 狗[] = [一只狗];
let 动物数组: 动物[] = 狗数组; // 正确：狗[] 是 动物[] 的子类型
```

协变很直观：如果 `狗` 是 `动物` 的子类型，那么 `狗数组` 也是 `动物数组` 的子类型。

#### 逆变示例

函数参数类型在 TypeScript 中是逆变的：

```typescript
// 定义函数类型
type 喂动物 = (动物: 动物) => void;
type 喂狗 = (狗: 狗) => void;

// 实现函数
const 喂任何动物: 喂动物 = (动物) => { 动物.吃(); };
const 专门喂狗: 喂狗 = (狗) => { 狗.吃(); 狗.汪汪(); };

// 逆变关系
let 喂动物的函数: 喂动物 = 喂任何动物;
let 喂狗的函数: 喂狗 = 喂任何动物; // 正确：喂动物 可以赋值给 喂狗
// let 喂动物的函数2: 喂动物 = 专门喂狗; // 错误：喂狗 不能赋值给 喂动物
```

逆变可能不那么直观：能喂任何动物的函数也能喂狗（因为狗是动物），但专门喂狗的函数不一定能喂所有动物（因为它可能使用了狗特有的方法）。

### 3.3 为什么需要变型

变型保证了类型系统的安全性和一致性：

- **协变**确保我们可以在期望父类型的地方使用子类型，这符合"里氏替换原则"
- **逆变**确保函数参数的类型安全，防止在函数内部调用子类型特有的方法而导致错误

### 3.4 TypeScript 中的变型规则

1. **数组、对象和返回值类型是协变的**：
   ```typescript
   // 对象属性是协变的
   type 动物容器 = { 宠物: 动物 };
   type 狗容器 = { 宠物: 狗 };
   let 狗盒子: 狗容器 = { 宠物: 一只狗 };
   let 动物盒子: 动物容器 = 狗盒子; // 正确：协变
   ```

2. **函数参数是逆变的**（在 `strictFunctionTypes` 开启时）：
   ```typescript
   // 函数返回值是协变的
   type 返回动物 = () => 动物;
   type 返回狗 = () => 狗;
   let 获取狗: 返回狗 = () => ({ 吃() {}, 汪汪() {} });
   let 获取动物: 返回动物 = 获取狗; // 正确：返回值是协变的
   ```

3. **泛型类型通常是不变的**，除非它们在特定位置使用类型参数：
   ```typescript
   // 泛型类是不变的
   class 盒子<T> {
     内容: T;
     constructor(内容: T) {
       this.内容 = 内容;
     }
   }
   
   let 狗盒: 盒子<狗> = new 盒子(一只狗);
   // let 动物盒: 盒子<动物> = 狗盒; // 错误：盒子<狗> 不是 盒子<动物> 的子类型
   ```

## 总结

1. **any**：完全不安全但灵活的类型，绕过类型检查
2. **unknown**：安全的顶部类型，需要类型检查才能使用
3. **never**：底部类型，表示永远不会发生的值
4. **never[]**：永远为空的数组类型
5. **协变**：如果 A 是 B 的子类型，则 F<A> 是 F<B> 的子类型
6. **逆变**：如果 A 是 B 的子类型，则 F<B> 是 F<A> 的子类型

理解这些概念对于编写类型安全的 TypeScript 代码至关重要，尤其是在处理复杂的泛型类型时。