## this 指向问题

#### 作为函数调用

这样一个最简单的函数，不属于任何一个对象，就是一个函数，这样的情况在 `JavaScript` 的在浏览器中的非严格模式默认是属于全局对象 `window` 的，在严格模式，就是 `undefined`

```js
let age = 18 //这里如果换成var 则会打印18,因为在全局下var定义的变量会加到window上
function a() {
  let age = 19
  console.log(this.age)
}
a() //undefined
```

#### 作为函数方法调用

这种情况可以理解为和**作为函数调用**的情况类似（它就是作为一个函数调用的，没有挂载在任何对象上，所以对于没有挂载在任何对象上的函数，在非严格模式下 `this` 就是指向 `window` 的）

```js
let age = 18

function fn() {
  let age = 19
  innerFunction()
  function innerFunction() {
    console.log(this.age)
  }
}

fn() //undefined
```

#### 匿名函数

**匿名函数的 `this` 永远指向 `window`**

```js
   let age = 18;

    let a = {
        age :19,

        func1: function () {
            console.log(this.age)
        },

        func2: function () {
            setTimeout(function () {
                this.func1()
            },100 ;
        }

    };

    a.func2()//匿名函数中的this指向window，所以会报错，window上不存在func1属性

```
