---
title: Typescript高级特性
date: 2021-07-23 10:37:48
excerpt: Typescript高级特性之交叉类型，联合类型，类型保护
tags:
- Typescript
- 前端
categories: 前端
---

### 引言
Typescript已经是前端工程开发的必备利器和组件库开发的不二之选。
在看源码的时候，经常会出现typescript的一些高级特性，比如交叉类型，联合类型和类型保护。适当使用这些特性能帮助我们更好的开发和阅读前端项目。

### 交叉类型

```typescript
interface ObjectConstructor {
  assign<T, U>(target: T, source: U): T & U;
}
```
上面的是ts的源码，可以看到这里将U拷贝到T类型，返回了T和U的交叉类型

再举一个简单的例子

```typescript
interface A {
  name: string,
  age: number,
  sayName: (name: string) => void
}
interface B {
  name: string,
  gender: string,
  sayGender: (gender: string) => void
}

let a: A & B

// 都是合法的
a.age
a.sayName
```
这里面的a对象继承了接口定义的所有属性

### 联合类型
联合类型是具有或关系的多个类型组合而成，只要满足其中一个类型即可。
```typescript
interface DateConstructor {
  new (value: number | string | Date): Date;
}
```
`Date`构造函数接受一个number或string或Date类型的参数，对应类型为number | string | Date
联合类型A | B要么是A要么是B，因此只有所有源类型的公共成员（“交集”）才能访问。

```typescript
interface A {
  name: string,
  age: number,
  sayName: (name: string) => void
}
interface B {
  name: string,
  gender: string,
  sayGender: (gender: string) => void
}

let a: A | B

// 可以访问
a.name

// 不可以访问
a.age
a.sayGender()
```
上面的例子中只能访问公共的属性，因为编译器不知道到底是A类型还是B类型，所以只能访问公共的属性。

### 类型保护
类型保护我的理解是，通过if等条件语句的判断告诉编译器，我知道它的类型了，可以放心调用这种类型的方法，常用的类型保护有`typeof`类型保护，`instanceof`类型保护和**自定义类型保护**

#### typeof类型保护
```typescript
function BuildURL(param: string | number): any {
  if (typeof param === 'string') {
    return param.toUpperCase()
  }
}
```

在上面的很简单的例子中，由于使用了`typeof`类型保护，所以在if的分支里可以告诉编译器放心的调用string类型的方法，编译器也会给出自动提示

```typescript
function BuildURL(param: string | number): any {
  return typeof param !== 'number' && param.startsWith("xxx")
}
```

结合类型保护编译器会非常的智能，比如上面的例子，编译器知道我们传来的param只有`string`和`number`两种类型，由于使用了类型保护，编译器知道param是string类型，所以可以调用startsWith方法。
具体的，typeof类型保护能够识别两种形式的typeof：
+ `typeof v === "typename"`
+ `typeof v !== "typename"`
  
typename只能是number、string、boolean或symbol，因为其余的typeof检测结果不那么可靠，比如

```typescript
let x: any;
if (typeof x === 'function') {
  // any类型，typeof类型保护不适用
  x;
}
if (typeof x === 'object') {
  // any类型，typeof类型保护不适用
  x;
}
```
所以有时候我们需要使用自定义方式实现保护类型

### instanceof类型保护

instanceof类型保护和typeof类型用法相似，它主要是用来判断是否是一个类的对象或者继承对象的。

```typescript
let x: Date | RegExp;
if (x instanceof RegExp) {
  // 正确 instanceof类型保护，自动对应到RegExp实例类型
  x.test('');
}
else {
  // 正确 自动对应到Date实例类型
  x.getTime();
}

interface DateOrRegExp {
    // 这里表示构造器无参，Date类型的类
    new(): Date;
    new(value?: string): RegExp;
}

let A: DateOrRegExp;
let y;
if (y instanceof A) {
    // y从any到RegExp | Date
    y;
}
```

### 自定义类型保护

typeof与instanceof类型保护能够满足一般场景，对于一些更加特殊的，可以通过自定义类型保护来对应类型,比如我们自己定义一个请求参数的接口类型

```typescript
interface RequestParams {
  url: string,
  onSuccess?: () => void,
  onFailure?: () => void
}

function isValidRequestParams(request: any): request is RequestParams {
  return request && request.url
}

let request

// 检测客户端发送过来的参数
if (isValidRequestParams(request)){
  request.url
}
```
这里面通过判断，我们需要手动告诉编译器通过isValidRequestParams的判断则request就是RequestParams类型的参数，编译器如何知道这一点呢，我们在这里通过类型谓词`parameterName is Type`告诉了编译器，isValidRequestParams返回了true则request就是RequestParams类型。
其它用法和上面所列举的一致

```typescript
let isNumber: (value: any) => value is number;

let x: string | number;
if (isNumber(x)) {
  // 缩窄到number
  x.toFixed(2);
}
else {
  // 通过类型保护，编译器知道不是number就是string
  x.toUpperCase();
}
```
结合类型保护，编译器会更加的智能，也能极大地降低bug出现的风险。

### 其他特性

+ new()类型
```typescript
function create<T>(c: { new(): T }): T {
  return new c()
}

class Test {
  constructor() {
    console.log("hello test class")
  }
}

create(Test)
```
在typescript中，要实现工厂模式是很容易的，我们只需要先声明一个构造函数的类型参数，它构造出来的是T类型new():T，然后在函数体里面返回c这个类构造出来的对象即可。

+ 空类型安全

针对空类型的潜在问题，TypeScript提供了–strictNullChecks选项，开启之后会严格检查空类型：
```typescript
let x: string;
// 错误 Type 'null' is not assignable to type 'string'.
x = null;
// 错误 Type 'undefined' is not assignable to type 'string'.
x = undefined;
```
