### LazyMan问题

>  实现一个LazyMan，可以按照以下方式调用:
> LazyMan(“Hank”)输出:
> Hi! This is Hank!

>  LazyMan(“Hank”).sleep(10).eat(“dinner”)输出
> Hi! This is Hank!
> //等待10秒..
> Wake up after 10
> Eat dinner~

> LazyMan(“Hank”).eat(“dinner”).eat(“supper”)输出
> Hi This is Hank!
> Eat dinner~
> Eat supper~

>  LazyMan(“Hank”).sleepFirst(5).eat(“supper”)输出
> //等待5秒
> Wake up after 5
> Hi This is Hank!
> Eat supper
> 以此类推。

分析题目，打招呼和eat的方式都是最简单的，直接输出语句，难度主要在于sleep和sleepFirst的处理，因为涉及到等待，我们需要用到异步，在javascript中的异步有setTimeout和promise，setTimeout一定是会用到的。因为需要触发下一个流程，我们也自然会想到用promise的then去触发。然后又因为这里能无限的添加任务内容，所以可能会一直.下去，所以我们只能用一个列表来存储这些准备执行的任务，而在列表、顺序执行任务、promise这几个关键词让我们想起来promise的串行执行写法。这个写法可以参考[reduce 这篇文章](./reduce.md)里面的线性执行promise的做法。

#####所以这里先诞生了一种思路，串行promise形成的方式。

这里先给出一个直接的样例使用

```javascript
// 用来生成返回promise的方法的
generatePromise = (fn, times) => {
    return () => new Promise((resolve) => {
        setTimeout(() => {
            fn();
            resolve();
        }, times * 1000)
    })
}
let list = [];
list.push(generatePromise(() => {
            console.log(`Hello`);
        }, 2))
list.push(generatePromise(() => {
            console.log(`world`);
        }, 1))
list.reduce((lastFn, currFn) => lastFn.then(() => currFn()), Promise.resolve())
// 两秒后输出
// Hello
// 再过一秒
// world
```

这里可以清晰地看出来 **generatePromise**方法是生成一个能在串行promise中执行方法的方法（返回一个方法的方法），第二个参数是延迟的秒数，这样就可以完美解决题目中的sleep这个问题了。

接下来则是sleepFirst的处理，则需要在sleepFirst调用的时候，把这个延迟时间的方法通过unshift设置到队列的第一个位置。这样也就实现了一个事件序列。

但是需要在什么时候触发这个事件序列开始执行？因为不知道最后一个是哪个方法，所以在这些方法最后去触发执行，也是不合理的。这时候我们考虑一下这些事件的生成，都在一个事件循环的一个周期里面，那我们的触发放在下一个周期去执行，不就完美解决了么？setTimeout(start, 0);呼之欲出。

将会在代码中

```javascript
// LazyMan直接调用，但是我们需要一个类来承载这些操作，ES6之前用原型链方式去编写即可
class _LazyMan {
  	// 任务列表
    list = [];
    constructor(str){
      	// 添加第一个任务，打招呼任务，时间设置为0
        this.list.push(generatePromise(() => {
            console.log(`Hi, this is ${str}`);
        }, 0))
      	// 添加完之后，在下一个事件周期中去触发任务的串行执行。
        setTimeout(() => {
            this.start();
        }, 0)
    }

    eat(str){
        this.list.push(generatePromise(() => {
            console.log(`Eat ${str}~`);
        }, 0))
        return this;
    }

    sleep(times){
        this.list.push(generatePromise(() => {
            console.log(`Wake up after ${times}`);
        }, times))
        return this;
    }

		// 通过 unshift 将任务添加到队头
    sleepFirst(times){
         this.list.unshift(generatePromise(() => {
            console.log(`Wake up after ${times}`);
        }, times))
        return this;
    }

		// 触发串行任务执行
    start(){
        this.list.reduce((lastFn, currFn) => lastFn.then(() => currFn()), Promise.resolve())
    }
}

// 生成异步任务的任务
generatePromise = (fn, times) => {
    return () => new Promise((resolve) => {
        setTimeout(() => {
            fn();
            resolve();
        }, times * 1000)
    })
}

// 程序总入口
function LazyMan(str){
    return new _LazyMan(str);
}
```

该方案使用到了promise的then，而且还联合了reduce实现了promise串行。对大部分人来说心智负担会稍微有点重。并且即使在任务无需异步时，也会使用到异步，这里能实现效果，但是有点洁癖的感觉，不太能忍(其实也能处理，不过因为对于大思路没有影响，不扣这里的细节)。

##### 第二种方案，则是事件队列+手动触发执行下一个任务

这个方案则是不使用promise的then来进行操作的。因为难点的地方均在方案一中解决了，所以方案二中只不过是把触发下一个任务的执行更换一下即可。这里在每一个任务结束的时候，去调用一个_LazyMan的next()方法。talk is cheap, show you the code.

```javascript
class _LazyMan {
    tasks = [];

    constructor(str){
        const fn = () => {
            console.log(`Hi! This is ${str}`);
            this.next();
        }
        this.tasks.push(fn);
        setTimeout(this.next, 0)
    }

    sleep = (time) => {
        const fn = () => {
            setTimeout(() => {
                console.log(`Wake up after ${time}`);
                this.next();
            }, time*1000)
        }
        this.tasks.push(fn);
        return this;
    }
    
    eat = (str) => {
        const fn = () => {
            console.log(`Eat ${str}`);
            this.next();
        }
        this.tasks.push(fn);
        return this;
    }

    sleepFirst = (time) => {
        const fn = () => {
            setTimeout(() => {
                console.log(`Wake up after ${time}`);
                this.next();
            }, time*1000)
        }
        this.tasks.unshift(fn);
        return this;
    }

    next = () => {
        const task = this.tasks.shift();
        task && task();
    };

}

function LazyMan(str){
    return new _LazyMan(str);
}


```

如果在方案一中已经认真地看过了的话，方案二中的内容其实就变得相当简单了。如果需要详细解释也可以联系我写更详细些。

其实本来一直在理解方案二的，临写文章的时候想出了第一种方案，也尝试写了串行promise的写法，因为文章的写法先难后易，这篇文章已经成型，之后搬迁的时候会用另外一种方式来处理。以后会改变写作这种方式。

