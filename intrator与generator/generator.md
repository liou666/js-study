## Generator函数

#### 基本使用

调用 `Generator` 函数后，**该函数并不执行，返回的也不是函数运行结果，而是一个指向内部状态的指针对象。（也就是一个迭代器对象）**

迭代器对象的`next`方法的运行逻辑如下。

1. 遇到`yield`表达式，就暂停执行后面的操作，并将紧跟在yield后面的那个表达式的值，作为返回的对象的value属性值。

2. 下一次调用`next`方法时，再继续往下执行，直到遇到下一个`yield`表达式。

3. 如果没有再遇到新的`yield`表达式，就一直运行到函数结束，直到`return`语句为止，并将`return`语句后面的表达式的值，作为返回的对象的`value`属性值。

4. 如果该函数没有`return`语句，则返回的对象的`value`属性值为`undefined`。
```js
function* helloWorldGenerator() {
    console.log(1);
    yield 'hello';//第一次调用next方法，执行yield上面👆的代码。并将yiled之后的表达式“hello”作为返回对象的value
    console.log(2);
    yield 'world';
    return 'ending';
}

const hw = helloWorldGenerator();

console.log(hw.next());//1 { value: 'hello', done: false }
console.log(hw.next());//2 { value: 'world', done: false }
console.log(hw.next());//2 { value: 'ending', done: true }

```
#### 与 Iterator 接口的关系
>`Generator`函数其实就是一个迭代器生成函数，因此可以把 `Generator` 赋值给对象的`Symbol.iterator`属性，从而使得该对象具有 `Iterator` 接口。
```js
//demo
let myIterable={};
myIterable[Symbol.iterator] = function* () {
    yield 1;
    yield 2;
    yield 3;
}
console.log(...myIterable);//1,2,3
```

#### next函数的参数
>`next`函数传的参数会作为**上一个**`yiled`表达式的返回值。不传的话为`underfind`

```js
function* helloWorldGenerator() {
    let res1 = yield 'hello';
    //执行第二次next函数，代码会执行到这这里，
    //第二次next函数的参数为b，所以第一个yield表达式的返回值res1就为b
    console.log(res1);//b
    let res2 = yield 'world';
    //同理这里输出c
    console.log(res2);
    return 'ending';
}
const hw = helloWorldGenerator();

console.log(hw.next("a"));
console.log(hw.next("b"));
console.log(hw.next("c"));
```

#### for...of
>for...of循环可以自动遍历 Generator 函数运行时生成的Iterator对象，且此时不再需要调用next方法。
```js
function* gen() {
    yield 1;
    yield 2;
    yield 3;
    return 4;
}
//for...of循环遇到{done:true}时会终止循环
 for (const value of gen()) {
     console.log(value);//1 2 3
 }
 
```
利用`generator`和`for...of`实现斐波那契数列
```js
function* feibonacci() {
    let [pre, current] = [0, 1];
    while (true) {
        yield current;
        [pre, current] = [current, pre + current]
    }
}
for (const n of feibonacci()) {
    if (n > 100) break;
    console.log(n);
}
```

#### yield* 表达式
>`yield*`表达式，用来在一个 `Generator` 函数里面执行另一个 `Generator` 函数。

```js
function* bar() {
    yield 1;
    yield* foo();  //👈相当于yield 4;yield 5;yield 6;相当于执行了 for(let n of foo()){}
    yield 2;
}

function* foo() {
    yield 4;
    yield 5;
    yield 6;
}

console.log(...bar());//1 4 5 6 2
```

如果上例中foo函数有return值，则return的值会作为bar函数中 yield*表达式的返回值。

```js
function* bar() {
    yield 1;
    const res = yield* foo();//res的值为foo的return值-"hahaha"
    console.log(res);
    yield 2;
}

function* foo() {
    yield 4;
    yield 5;
    yield 6;
    return "hahaha"
}
let g = bar();
console.log(g.next());//{ value: 1, done: false }
console.log(g.next());//{ value: 4, done: false }
console.log(g.next());//{ value: 5, done: false }
console.log(g.next());//{ value: 6, done: false }
console.log(g.next());//hahaha { value: 2, done: false }
```

利用yield* 实现数组扁平化
```js
function* flat(arr) {
    if (Array.isArray(arr)) {
        for (const n of arr) {
            yield* flat(n)
        }
    } else {
        yield arr
    }
};
let test = ['a', ['b', 'c'], ['d', 'e']];

console.log([...flat(test)]);//[ 'a', 'b', 'c', 'd', 'e' ]
```

#### Generator函数自执行
`Thunk` 函数是自动执行 `Generator` 函数的一种方法。
```js
//传统形式容易形成回调地狱
fs.readFile('./hello.txt', function (err1, data1) {
    if (err1) throw err1;
    console.log(data1.toString());
    fs.readFile('./wrold.txt', function (err2, data2) {
        if (err2) throw err2;
        console.log(data2.toString());
    });
});
```
将上例用thunk和Generator函数改造。

```js

//之后读取文件可以通过readFile("hello.txt")(callback)形式
function trunkify(fn) {
    return function (...args) {
        return function (callback) {
            return fn.call(this, ...args, callback)
        }
    }
}
const readFile = trunkify(fs.readFile);
```

```js
function* gen() {
    const res1 = yield readFile("./hello.txt");
    console.log(res1.toString());
    const res2 = yield readFile("./wrold.txt");
    console.log(res2.toString());
}

//自执行函数
function run(gen) {
    let g = gen();
    function go(err, data) {
        let res = g.next(data);
        if (res.done) return
        res.value(go)
    }
    go()
}

run()//依次输出hello wrold
```

Thunk 函数并不是 Generator 函数自动执行的唯一方案。Promise 对象也可以做到这一点。
```js
//目标：依次执行getParams，getData函数
function getParams() {
    return new Promise(resolve => {
        setTimeout(() => {
            resolve("liou")
        }, 2000)
    })
}

function getData(name) {
    return new Promise(resolve => {
        setTimeout(() => {
            resolve("My name is" + name)
        }, 2000)
    })
}

function* gen() {
    let res1 = yield getParams();
    console.log(res1);//两秒后输出："liou"
    let res2 = yield getData(res1);
    console.log(res2);//再过两秒后输出："My name is liou"
}

//手动调用generator函数
let res = g.next();
res.value.then(res1 => {
    res = g.next(res1);
    res.value.then(res2 => {
        g.next(res2)
    })
})

//自执行函数
 function run(gen) {
     let g = gen();
     function go(a) {
         let res = g.next(a);
         if (res.done) return;
         res.value.then(go)
     }
     go()
 }
 run(gen)
```


