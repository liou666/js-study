
## Iterator 和 for...of循环

#### 迭代器的相关概念名词
+ `Iterator` 接口的目的，就是为所有数据结构，提供了一种统一的访问机制，即`for...of`循环。当使用`for...of`循环遍历某种数据结构时，该循环会自动去寻找 `Iterator` 接口。

+ 当 `for..of` 循环启动时，它会调用`Symbol.iterator`这个方法（如果没找到，就会报错）。这个方法必须返回一个**迭代器（iterator）** --- **一个有 next 方法的对象。**
+ 从此开始，`for..of` 仅适用于这个被返回的对象。当 `for..of` 循环希望取得下一个数值，它就调用这个对象的 `next()` 方法。
+ `next()` 方法返回的结果的格式必须是 `{done: Boolean, value: any}`，当 `done=true` 时，表示迭代结束，否则 `value` 是下一个值。

```js
//一个迭代的小demo

class RangeIterator {
    constructor(start, stop) {
        this.start = start;
        this.stop = stop;
    }
    [Symbol.iterator]() {
        return this
    }
    next() {
        if (this.start < this.stop) {
            return { done: false, value: this.start++ }
        } else {
            return { done: true }
        }
    }
}

function range(start, stop) {
  return new RangeIterator(start, stop);
}

for (var value of range(0, 3)) {
  console.log(value); // 0, 1, 2
}
```

#### 原生具备 Iterator 接口的数据结构如下。

+ Array
+ Map
+ Set
+ String
+ TypedArray
+ 函数的 arguments 对象
+ NodeList 对象

```js
//原生数组自带Iterator 接口
const arr = [1, 2, 3];
const iterator = arr[Symbol.iterator]();
console.log(iterator.next());//{ value: 1, done: false }
console.log(iterator.next());// { value: 2, done: false }
console.log(iterator.next());// { value: 3, done: false }
console.log(iterator.next());// { value: undefined, done: true }

//也可以采用循环让迭代自动完成
let res = iterator.next();
while (!res.done) {
    console.log(res.value);//依次输出1 2 3
    res = iterator.next()
}
```

#### 调用 Iterator 接口的场合
+ 解构赋值
+ 扩展运算符
+ yield* 
+ for...of
+ Array.from()
+ Map(), Set(), WeakMap(), WeakSet()（比如new Map([['a',1],['b',2]])）
+ Promise.all()
+ Promise.race()

```js
//解构赋值会自动调用Iterator 接口（demo）
let str = new String("hi");
[...str]//["h","i"]；


//更改迭代器接口
str[Symbol.iterator] = function () {
    return {
        next() {
            
            if (this._first) {
                this._first = false;
                return { value: "wrold", done: false }
            } else {
                return { done: true }
            }
        },
        _first: true
    }
}
console.log([...str])//['wrold']
```


#### 遍历器对象的 return()，throw()
>`return()`方法的使用场合是，如果`for...of`循环提前退出（通常是因为出错，或者有`break`语句），就会调用`return()`方法。

```js
//将上例中RangeIterator类中增加retern方法
class RangeIterator{
    ...;
    return() {
      console.log('提前退出');
      return { done: true };
  }
}

for (var value of range(0, 3)) {
   if (value === 2) {
       break
   }
  console.log(value); // 0, 1, "提前退出"
}
```
>`throw()`方法主要是配合 `Generator` 函数使用，一般的遍历器对象用不到这个方法。