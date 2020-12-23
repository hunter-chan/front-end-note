### reduce的了解和使用

内容来自 LucasHC 在GitChat的《[前端开发核心知识进阶](https://gitbook.cn/gitchat/column/5c91c813968b1d64b1e08fde)》的第五章

reduce是挂载在Array原型中的方法，用于使用特定的方法对列表内容进行循环处理。文字没法描述太多，直接看代码。

```javascript
let arr = [1,2,3,4,5,6,7,8,9,10];
let num1 = arr.reduce((last, curr) => {
    return last + curr;
}, 11)
console.log(num1); // 66
```

reduce可以传入两个参数，第一个为一个执行函数，第二个为一个初始处理的数值。开始执行的时候，会将初始值传入执行函数中。第一个参数是执行函数，其中reduce会往这个函数传入4个参数，分别是1、上一个执行函数返回的值，2、当前要处理的值，3、当前值对应的索引，4、对应的arr数组。



#### 特别的用法，线性执行promise。

```javascript
const runInPromiseSequence = (array, value) => array.reduce(
    (promiseChain, current) => promiseChain.then(current), // promiseArray执行reduce的第一个参数
    Promise.resolve(value)); // promiseArray执行reduce的第二个参数，初始值

let promiseArr = [
    (last) => new Promise((resolve) => {
        setTimeout(() => {
            console.log(last);
            resolve('first')
        }, 4000)
    }),
    (last) => new Promise((resolve) => {
        setTimeout(() => {
            console.log(last);
            resolve()
        }, 4000)
    }),
]

runInPromiseSequence(promiseArr, 'zero') // 将promiseArr传进方法中，只有在前一个promise执行完毕才会往后继续执行。
// zero  //4s之后打印
// first  //8s之后打印
```



#### 另外一个就是实现柯里化函数(这里直接复制课程内容过来了，柯里化这个内容我会在另一篇文章中添加介绍)

`reduce` 的另外一个**典型应用**可以参考函数式方法 `pipe` 的实现：`pipe(f, g, h)` 是一个 curry 化函数，它返回一个新的函数，这个新的函数将会完成 `(...args) => h(g(f(...args)))` 的调用。即 `pipe` 方法返回的函数会接收一个参数，这个参数传递给 `pipe` 方法第一个参数，以供其调用。

```javascript
let curry = (...functions) => input => functions.reduce((last, currFun) => currFun(last), input)
```



#### 手写一个reduce功能

这里的主体逻辑和课程的基本一致，只是加了一些细节的补充，然后使用for循环，等可以符合ES5执行的内容。

```javascript
function reduce(callback, initialValue){
    if(!(this instanceof Array)){ // 如果不是用array调用的，将不再往下执行，毕竟需要array才能循环起来。这里的this就是指向array
        throw new Error('Need an array');
    }
    var initial = typeof initialValue === 'undefined' ? this[0] : initialValue;
    var startIndex = typeof initialValue === 'undefined' ? 1 : 0;// 这两行用来判断当初始值为undefined的时候，从0开始循环，并且初始值为传入的初始值。若无初始值时，将array索引为0的值作为初始值传入。此处可推，初始值最好和array中的值保持为同一类型
    var length = this.length; // 缓存this.length是和JavaScript执行性能相关，避免不断往上查询属性，只有array极大时会有影响，这里暂且忽略不计。
    for(var i=0 ; i+startIndex < length ; i++){
        var current = this[i+startIndex]; // 拿到参数
        initial = callback(initial, current, i+startIndex, this); // 依照原有的规则向callback方法中传递参数，initial用来存储每次callback执行完的返回值，用于下一次循环的参数
    }
    return initial; // 返回最终的结果
}

let arr = [1,2,3,4,5,6,7,8,9,10];
let num = reduce.call(arr, (last, curr) => { // 此处没有讲reduce挂载到Array中，直接使用call来调用
    return last + curr;
}, 11);
console.log(num); // 66
```

