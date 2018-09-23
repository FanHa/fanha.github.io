### 对象的`prototype`, 实例的 `__proto__`
```js
a1 = new A()
a2 = new A()
```
+ a1,a2都是都是A对象得实例,各自有一块属于自己的地盘(内存);
+ 但A对象可能有一些各个实例都能共享的方法 `A.method1()`,`A.method2()`,
+ 如果每一个A实例都单独保存一份`method1`,`method2`,既浪费空间也不方便维护,
+ 于是所有公共方法统一放在一块地方,然后每个实例新增一个隐的属性`__proto__`指向这块公共区域,这块公共区域被称为`prototype`,
+ `a1.__proto__`,`a2.__proto__`都指向这块公共的`prototype`区域;
+ 这样`a1.method1()`,`a2.method1()`都能实际调用到公共区域里的`A.method1()`方法;
  + 这里a1,a2,实际上是先查看了自身实例里有没有`method1`这个方法,如果有则执行自身实例的`method1`,如果无则将调用`method1`的责任交给`__proto__`指向的`prototype`;
  + 其实`prototype`本身也是一个对象实例,也可以有`__proto__`,所以这形成了一个递归责任链,依次递归对象实例的`__proto__`,直到找到需要调用的`method1`或返回`undefined`;

---
### 对象的定义
在前面`a1 = new A()` 之前首先得有地方定义A 得结构,PHP,C++等是用Class来制定这个结构,javascript用`function` 来制定;
```php
class A {
  function __construct(){

  }

  function method1(){

  }

  function method2(){
    
  }

}
```
```js
function A() {
  
  ...
}

A.prototype.method1 = function () {

}

A.prototype.method2 = function () {

}
```
其实在js钟 `function A() {}` 可以看作定义了一块prototype区域A,然后把 `function A()` 里的内容赋予`A.prototype.constructor`,即很多语言所说的构造函数;
```js
A.prototype.constructor = // function A 的内容
```

---
### 继承
+ `DerivedA`继承`A`,
+ 我们想要的是在`prototypeA`的基础上构造一个`prototypeDerivedA`,
+ `prototypeDerivedA`扩展一些`A`的功能的同时,又能使用`prototypeA`中已有的属性,方法等;

+ js中 prototype本身也是个对象实例,所有也可以拥有对象实例的`__proto__`
+ 于是`prototypeDerivedA.__proto__` 是可以指向 `protoTypeA`的;
```js
function A = {}
function DerivedA = {}
DerivedA.prototype.__proto__ = A.prototype
```
经过上面一般操作,就可以说DerivedA继承A了;
```js
let da = new DerivedA()
// da.x ...
// da.xxx() ...
```
+ 新建一个`DerivedA`实例`da`,
+ 当`da`调用属性或方法`x`时,首先会在从da实例自身的属性里找,如果没有就会把调用`x`的责任下发给`da.__proto__`,即`DerivedA`的原型`prototypeDerivedA`;
+ `DerivedA`的原型`prototypeDerivedA`也是个对象,先找自己有没有`x`方法,如果没有,则继续把调用`x`的责任下发给自己的`__proto__`,即`A`的原型`protoTypeA`;

- 上面就形成了基本的继承机制
---

### 基础Object 与 有名有姓的Object
+ 上面说的对象实例都是通过`new A()`,`new B()`生成的;  
+ 实际所有的对象实例都共享一套基础的方法,属性(如`defineProperty`,`keys`等等),因此有一块存放这些共享属性的区域,然后所有其他对象的原型都继承自这块区域;这块区域就是`Object`的原型`prototype`;
+ 这样`A.prototype.__proto__`, `B.prototype.__proto__` 都指向 `Object.prototype`; `DerivedA.prototype`也能通过自己的`__proto__`访问到A,再最终访问到`Object.prototype`里的内容;

js的使用过程中,经常也会直接使用`obj = new Object()` 或 `obj = {}`之类的方式直接生成一个基础Object,而这个`obj.__proto__`就是`Object.prototype`;  

---
### function
定义一个对象的原型的时候,使用了`A.prototype.method1 = function () {}` 的方式,其实,function本身也是继承自`Object`的一种对象实例;

---
### 为什么
+ 有点绕,但其实就是一层层把可以共用的部分放在一个区域,然后通过指针(或引用)之类的机制使用公共区域的内容,既减少了空间的占用,也能够更便捷的管理和维护公共内容;
+ 很多语言都或多或少提供这种修改数据结构底层结构的方式以便于更高级的功能实现,但可能另外提供了更加平易近人的接口(或语法糖)来实现继承等基本设计思想;
+ js可能在这方面比较hacker和粗犷,直接暴露出更多底层实现细节,让使用者自己调度拼装这些细节;