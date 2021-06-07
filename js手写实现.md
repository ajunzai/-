### debounce防抖
```javascript
const debounce = (fn, delay) => { 
    let timer = null; 
    return (...args) => { 
        clearTimeout(timer); 
        timer = setTimeout(() => { 
            fn.apply(this, args); 
        }, delay); 
     }; 
};
```
* * *

### throttle节流
```javascript
const throttle = (fn, delay = 500) => { 
    let flag = true; 
    return (...args) => { 
        if (!flag) return; 
        flag = false; 
        setTimeout(() => { 
            fn.apply(this, args); 
            flag = true; 
        }, delay); 
    }; 
 };
```
* * *

### object.create 、call 、Apply 、bind 
```javascript
function myCreate() {
    // 创建一个新对象
    const obj = {}
    // 第一个参数是构造函数
    const Constructor = [].shift.call(arguments)
    // 修改原型
    obj.__proto__ = Constructor.prototype
    //3.执行构造函数，并将this指向创建的空对象obj
    const ret = Constructor.apply(obj, arguments)
    //4.如果构造函数返回的值是对象则返回，不是对象则返回创建的对象obj
    return typeof ret === 'object' ? ret : obj
}
```
```javascript
function mycall(context) {
    context = context || window
    // 拿到参数
    let args = [...arguments].slice(1)
    // 将调用函数设置为对象的属性
    context.fn = this
    // 执行函数
    let result = context.fn(args)
    // 删除属性
    delete context.fn
    return result
}
```
```javascript
function myApply(context) {
    context = context || window
    context.fn = this
    let result
    if (arguments[1]) {
        result = context.fn([...arguments[1]]) // 调用赋值的函数
    } else {
        result = context.fn()
    }
    delete context.fn
    return result
}
```
```javascript
function myBind(context) {
    // 获取第一个context的其他参数
    let bindArgs = [...arguments].slice(1)
    let self = this
    return function Fn() {
        let fnArgs = [].slice.call(arguments)
        return self.apply(this instanceof Fn ? this : context, bindArgs.concat(fnArgs))
    }
}
```
* * *

### promise手写进阶版 **race** **all**
```javascript
const STATUS_PENDING = 'pending'
const STATUS_FULFILLED = 'fulfilled'
const STATUS_REJECTED = 'rejected'

class myPromise {
    constructor(exector) {
        this.status = STATUS_PENDING
        this.value = ''
        this.resean = ''

        // 成功存放的数组
        this.onResolvedCallbacks = []
        // 失败存放的数组
        this.onRejectedCallbacks = []

        let resolve = value => {
            if (this.status === STATUS_PENDING) {
                this.status = STATUS_FULFILLED
                this.value = value
                this.onResolvedCallbacks.forEach(fn => fn())
            }
        }

        let reject = resean => {
            if (this.status === STATUS_PENDING) {
                this.status = STATUS_REJECTED
                this.resean = resean
                this.onRejectedCallbacks.forEach(fn => fn())
            }
        }
        try{
            exector(resolve, reject)
        }catch(err){
            reject(err)
        }
    }

    then(onFulfilled = () => {}, onRejected = () => {}) {
        // !!重复回调
        // onFulfilled如果不是函数，就忽略onFulfilled，直接返回value
        // onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value;
        // // onRejected如果不是函数，就忽略onRejected，扔出错误
        // onRejected = typeof onRejected === 'function' ? onRejected : err => {
        //     throw err
        // };

        if (this.status === STATUS_FULFILLED) {
            onFulfilled(this.value)
            // 异步解决：
            // onRejected返回一个普通的值，失败时如果直接等于 value => value，
            // 则会跑到下一个then中的onFulfilled中，
            // setTimeout(() => {
            //     try {
            //         let x = onFulfilled(this.value);
            //         resolvePromise(promise2, x, resolve, reject);
            //     } catch (e) {
            //         reject(e);
            //     }
            // }, 0);

        }
        if (this.status === STATUS_REJECTED) {
            onFulfilled(this.resean)
        }
        if (this.status === STATUS_PENDING) {
            this.onResolvedCallbacks.push(() => onFulfilled(this.value))
            this.onRejectedCallbacks.push(() => onRejected(this.resean))
        }
    }

    resolve(value) {
        return new myPromise((resolve,reject) => resolve(value))
    }

    reject(reason) {
        return new myPromise((resolve, reject) => reject(reason))
    }
}

/**
 * 用来做then持续回调
 * @param promise2 新new的promise
 * @param x 第一个then的返回值
 * @param resolve 构造函数的结果回调函数
 * @param reject
 * @returns {*}
 */
function resolvePromise(promise2, x, resolve, reject) {
    if (x === promise2) {
        return reject(new TypeError('循环引用'))
    }
    // 防止重复调用
    let called
    if (x !== null && (typeof x === 'object'|| typeof x ==='function')) {
        try {
            // A+规定，声明then = x的then方法
            let then = x.then
            if (typeof then === 'function') {
                // 就让then执行 第一个参数是this   后面是成功的回调 和 失败的回调
                then.call(x, y => {
                    // 成功和失败只能调用一个
                    if (called) return
                    called = true
                    // resolve的结果依旧是promise 那就继续解析
                    resolvePromise(promise2, y, resolve,reject)
                }, err => {
                    // 成功和失败只能调用一个
                    if (called) return
                    called = true
                    reject(err)
                })
            } else {
                resolve(x) // 直接成功即可
            }
        } catch (e) {
            if (called) return
            called = true
            reject(e)
        }
    } else {
        resolve(x)
    }
}

myPromise.race = function(promises) {
    return new myPromise((resolve,reject)=>{
        let len = promises.length
        if (len === 0) return
        for (let i = 0;i<len; i++) {
            Promise.resolve(promises[i]).then(data => {
                resolve(data)
                return
            }, err => {
                reject(err)
                return
            })
        }
    })
}

myPromise.all = function (promises) {
    return new myPromise((resolve, reject) => {
        let index = 0
        let result = []
        if (promises.length === 0) {
            resolve(result)
        } else {
            function processValue(i, data) {
                result[i] = data
                if (++index === promises.length) {
                    resolve(result)
                }
            }
            for (let i =0; i< promises.length; i++) {
                Promise.resolve(promises[i]).then((data) => {
                    processValue(i, data)
                }, err => {
                    reject(err)
                    return
                })
            }
        }
    })
}

new myPromise((resolve,reject) => {
    setTimeout(() => {
        resolve(1)
    }, 1000)
}).then(res => {
    console.log(res)
},err=> {
    console.log(err)
})
```
* * *

### lru缓存，keep-alive
``` javascript
class LRUCache {
  constructor(capacity) {
    this.capacity = capacity
    this.cache = new Map()
  }

  put(key, value) {
    if (this.cache.has(key)) {
      // 存在即更新（删除后加入）
      this.cache.delete(key)
    } else if (this.cache.size >= this.capacity) {
      // 不存在即加入
      // 缓存超过最大值，则移除最近没有使用的
      // keys()拿到所有key，next为最前一个生成对象{value: '', done: false},取value值
      this.cache.delete(this.cache.keys().next().value)
    }
    this.cache.set(key, value)
  }

  get(key) {
    if (this.cache.has(key)) {
      // 存在即更新
      let temp = this.cache.get(key)
      this.cache.delete(key)
      this.cache.set(key, temp)
      return temp
    }
    return -1
  }
}
```
* * *

### proxy深拷贝
```javascript
const MY_IMMER = Symbol('my-immer')

// 检查 value 是否是普通对象。 也就是说该对象由 Object 构造函数创建，或者 [[Prototype]] 为 null 。
const isPlainObject = value => {
  if (
    !value ||
    typeof value !== 'object' ||
    Object.prototype.toString.call(value) !== '[object Object]'
  ) {
    return false
  }

  const proto = Object.getPrototypeOf(value)
  if (proto === null) {
    return true
  }
  const Ctor = proto.hasOwnProperty('constructor') && proto.constructor
  return (
    typeof Ctor === 'function' &&
    Ctor instanceof Ctor &&
    Function.prototype.toString.call(Ctor) ===
      Function.prototype.toString.call(Object)
  )
}

// 判断是否为proxy对象
const isProxy = value => !!value && !!value[MY_IMMER]

const produce = (baseState, fn) => {

  // 存放生成的proxy对象
  const proxies = new Map()
  // 存放set过的值 其对象，里面储存最新变化数据
  const copies = new Map()

  const getProxy = data => {
    if (isProxy(data)) {
      return data
    }
    if (isPlainObject(data) || Array.isArray(data)) {
      if (proxies.has(data)) {
        return proxies.get(data)
      }
      const proxy = new Proxy(data, handler)
      proxies.set(data, proxy)
      return proxy
    }
    return data
  }

  const handler = {
    get(target, key) {
      if (key === MY_IMMER) return target
      const data = copies.get(target) || target
      return getProxy(data[key])
    },
    set(target, key, val) {
      const copy = getCopy(target)
      const newValue = getProxy(val)
      // 这里的判断用于拿到 proxy 的target
      // 否则直接 copy[key] = newValue 的话外部拿到的对象是个 proxy
      copy[key] = isProxy(newValue) ? newValue[MY_IMMER] : newValue
      return true
    }
  }

  const getCopy = data => {
    if (copies.has(data)) {
      return copies.get(data)
    }
    const copy = Array.isArray(data) ? data.slice() : {...data}
    copies.set(data, copy)
    return copy
  }

  const isChange = data => {
    if (proxies.has(data) || copies.has(data)) return true
  }

  const finalize = data => {
    if (isPlainObject(data) || Array.isArray(data)) {
      if (!isChange(data)) {
        return data
      }
      const copy = getCopy(data)
      Object.keys(copy).forEach(key => {
        copy[key] = finalize(copy[key])
      })
      return copy
    }
    return data
  }

  const proxy = getProxy(baseState)
  fn(proxy)
  return finalize(baseState)
}

const state = {
  info: {
    name: 'yck',
    career: {
      first: {
        name: '111'
      }
    }
  },
  data: [1]
}

const data = produce(state, draftState => {
  draftState.info.age = 26
  draftState.info.career.first.name = '222'
})
console.log(data, state)
console.log(data.data === state.data)
```
