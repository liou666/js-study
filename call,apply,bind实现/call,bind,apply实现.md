## call,bind,apply实现

#### call的实现
>call方法第一个参数如果不传或者为null值，默认会将this绑定到window上。

```js
let foo={
    value:'foo'

}

function bar(name,age){
   console.log("this.value",this.value);
   console.log("name",name);
   console.log("age",age);
}



```

实现
```js
bar.call(foo,'liou',18)//输出：foo,liou,18

Function.prototype.myCall=function(obj,...args){
        obj=obj||window;//不传的话就将this设置为全局对象
        let Fn=Symbol();//防止属性名重复
        obj[Fn]=this;//这里的this就是当前调用的函数
        obj[Fn](...args);
        delete obj[Fn]
}

bar.myCall(foo,'liou',18)//输出：foo,liou,18

```

#### apply的实现
> apply接收数组为参数,参数不为数组会报错

```js
bar.apply(foo,['liou',18])//输出：foo,liou,18

Function.prototype.myApply=function(obj,args){
    if(!Array.isArray(args)){
        throw SyntaxError("myApply must accept array as arguments!!!")
    }
    obj=obj||window;
    let Fn=Symbol();
    obj[Fn]=this;
    obj[Fn](...args);
    delete obj[Fn]
}

bar.myApply(foo,['liou',18])//输出：foo ,liou,18

```

#### bind的实现

`bind`函数返回值是原函数的拷贝，并将`this`绑定在指定对象上。这里有两个注意点
+ `bind`函数支持**函数柯里化**的调用方式。
+ 绑定函数也可以使用 `new` 运算符构造，它会表现为目标函数已经被构建完毕了似的。提供的 this 值会被忽略，但前置参数仍会提供给模拟函数。

```js
let bindFn=bar.bind(foo,'liou')

bindFn(18)//输出：foo ,liou,18

new bindFn(18)//输出：underfind,liou,18         new bindFn时，this将不再指向foo对象，会指向new bar()的实例
```


```js
Function.prototype.mbind=function(obj,...args){
    let self=this;
    function bindFn(...bindFnArgs){
        self.apply(this instanceof self ? this: self,[...bindFnArgs,...args])
    }
    bindFn.prototype =this.prototype//这里将bindFn的原型指向原函数的原型，在new bindFn()的时候this指向bindFn函数，当执行bindFn()的时候this指向window
    return bindFn
}
```




