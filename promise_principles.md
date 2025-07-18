# Promise 原理详解

## 目录
1. [Promise 基础概念](#promise-基础概念)
2. [Promise 内部实现原理](#promise-内部实现原理)
3. [Promise 状态管理](#promise-状态管理)
4. [事件循环和微任务](#事件循环和微任务)
5. [Promise 链式调用](#promise-链式调用)
6. [错误处理机制](#错误处理机制)
7. [Promise 静态方法](#promise-静态方法)
8. [实际应用示例](#实际应用示例)

## Promise 基础概念

Promise 是 JavaScript 中处理异步操作的一种解决方案，它代表了一个异步操作的最终完成或失败，以及其结果值。

### 什么是 Promise？

Promise 是一个对象，它表示一个异步操作的最终值或失败原因。Promise 有以下特点：

- **状态不可逆**：一旦状态改变就不会再变
- **异步执行**：回调函数总是异步执行
- **链式调用**：支持 `.then()` 链式调用
- **错误传播**：错误会自动向下传播

### Promise 的三种状态

```javascript
// 三种状态
const PENDING = 'pending'     // 等待态
const FULFILLED = 'fulfilled' // 成功态
const REJECTED = 'rejected'   // 失败态
```

- **Pending（等待）**：初始状态，既没有被兑现，也没有被拒绝
- **Fulfilled（已兑现）**：操作成功完成
- **Rejected（已拒绝）**：操作失败

## Promise 内部实现原理

### 基础结构

```javascript
function MyPromise(executor) {
  // 初始状态为 pending
  this.state = 'pending'
  this.value = undefined
  this.reason = undefined
  
  // 存储回调函数
  this.onFulfilledCallbacks = []
  this.onRejectedCallbacks = []
  
  // resolve 函数
  const resolve = (value) => {
    if (this.state === 'pending') {
      this.state = 'fulfilled'
      this.value = value
      // 执行所有成功回调
      this.onFulfilledCallbacks.forEach(fn => fn())
    }
  }
  
  // reject 函数
  const reject = (reason) => {
    if (this.state === 'pending') {
      this.state = 'rejected'
      this.reason = reason
      // 执行所有失败回调
      this.onRejectedCallbacks.forEach(fn => fn())
    }
  }
  
  try {
    executor(resolve, reject)
  } catch (error) {
    reject(error)
  }
}
```

### then 方法实现

```javascript
MyPromise.prototype.then = function(onFulfilled, onRejected) {
  // 参数可选，提供默认值
  onFulfilled = typeof onFulfilled === 'function' ? onFulfilled : value => value
  onRejected = typeof onRejected === 'function' ? onRejected : reason => { throw reason }
  
  // 返回新的 Promise 实现链式调用
  const promise2 = new MyPromise((resolve, reject) => {
    if (this.state === 'fulfilled') {
      // 异步执行
      setTimeout(() => {
        try {
          const x = onFulfilled(this.value)
          resolvePromise(promise2, x, resolve, reject)
        } catch (error) {
          reject(error)
        }
      }, 0)
    } else if (this.state === 'rejected') {
      setTimeout(() => {
        try {
          const x = onRejected(this.reason)
          resolvePromise(promise2, x, resolve, reject)
        } catch (error) {
          reject(error)
        }
      }, 0)
    } else if (this.state === 'pending') {
      // 存储回调函数
      this.onFulfilledCallbacks.push(() => {
        setTimeout(() => {
          try {
            const x = onFulfilled(this.value)
            resolvePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      })
      
      this.onRejectedCallbacks.push(() => {
        setTimeout(() => {
          try {
            const x = onRejected(this.reason)
            resolvePromise(promise2, x, resolve, reject)
          } catch (error) {
            reject(error)
          }
        }, 0)
      })
    }
  })
  
  return promise2
}
```

### Promise 解析过程

```javascript
function resolvePromise(promise2, x, resolve, reject) {
  // 防止循环引用
  if (promise2 === x) {
    return reject(new TypeError('Chaining cycle detected for promise'))
  }
  
  let called = false
  
  if (x !== null && (typeof x === 'object' || typeof x === 'function')) {
    try {
      const then = x.then
      if (typeof then === 'function') {
        // x 是一个 thenable 对象
        then.call(x, y => {
          if (called) return
          called = true
          resolvePromise(promise2, y, resolve, reject)
        }, r => {
          if (called) return
          called = true
          reject(r)
        })
      } else {
        resolve(x)
      }
    } catch (error) {
      if (called) return
      called = true
      reject(error)
    }
  } else {
    resolve(x)
  }
}
```

## Promise 状态管理

### 状态转换规则

```javascript
// Promise 状态转换图
//     pending
//    /       \
//   ↓         ↓
// fulfilled  rejected
//   ↓         ↓
// [value]   [reason]
```

### 状态不可逆性

```javascript
const promise = new Promise((resolve, reject) => {
  resolve('success')
  reject('error') // 这行代码不会执行
  resolve('success again') // 这行代码也不会执行
})

promise.then(value => {
  console.log(value) // 输出: 'success'
})
```

## 事件循环和微任务

### 微任务队列

Promise 的回调函数会被放入**微任务队列**（Microtask Queue），而不是宏任务队列。

```javascript
console.log('1')

setTimeout(() => {
  console.log('2')
}, 0)

Promise.resolve().then(() => {
  console.log('3')
})

console.log('4')

// 输出顺序: 1, 4, 3, 2
```

### 执行优先级

1. **同步代码**：立即执行
2. **微任务**：Promise、queueMicrotask、MutationObserver
3. **宏任务**：setTimeout、setInterval、setImmediate

### 深入理解事件循环

```javascript
// 复杂的执行顺序示例
new Promise((resolve) => {
  console.log('外层 promise')
  resolve()
})
.then(() => {
  console.log('外层第一个 then')
  return new Promise((resolve) => {
    console.log('内层 promise')
    resolve()
  })
  .then(() => {
    console.log('内层第一个 then')
  })
  .then(() => {
    console.log('内层第二个 then')
  })
})
.then(() => {
  console.log('外层第二个 then')
})

// 输出:
// 外层 promise
// 外层第一个 then
// 内层 promise
// 内层第一个 then
// 内层第二个 then
// 外层第二个 then
```

## Promise 链式调用

### 链式调用原理

每个 `.then()` 方法都返回一个新的 Promise，这使得链式调用成为可能。

```javascript
// 链式调用示例
function fetchUserData(userId) {
  return fetch(`/api/users/${userId}`)
    .then(response => response.json())
    .then(user => {
      console.log('用户信息:', user)
      return fetch(`/api/posts?userId=${user.id}`)
    })
    .then(response => response.json())
    .then(posts => {
      console.log('用户文章:', posts)
      return posts
    })
}
```

### 值的传递

```javascript
Promise.resolve(1)
  .then(value => {
    console.log(value) // 1
    return value + 1
  })
  .then(value => {
    console.log(value) // 2
    // 如果不返回值，下一个 then 接收到 undefined
  })
  .then(value => {
    console.log(value) // undefined
  })
```

### Promise 穿透

```javascript
Promise.resolve(1)
  .then(2) // 非函数参数会被忽略
  .then(Promise.resolve(3)) // 非函数参数会被忽略
  .then(value => {
    console.log(value) // 1，值会穿透到下一个有效的 then
  })
```

## 错误处理机制

### catch 方法

```javascript
MyPromise.prototype.catch = function(onRejected) {
  return this.then(null, onRejected)
}
```

### 错误传播

```javascript
Promise.resolve()
  .then(() => {
    throw new Error('第一个错误')
  })
  .then(() => {
    console.log('这里不会执行')
  })
  .catch(error => {
    console.log('捕获错误:', error.message)
    return '恢复正常'
  })
  .then(value => {
    console.log('继续执行:', value)
  })
```

### finally 方法

```javascript
MyPromise.prototype.finally = function(callback) {
  return this.then(
    value => Promise.resolve(callback()).then(() => value),
    reason => Promise.resolve(callback()).then(() => { throw reason })
  )
}
```

## Promise 静态方法

### Promise.resolve

```javascript
Promise.resolve = function(value) {
  if (value instanceof Promise) {
    return value
  }
  return new Promise(resolve => resolve(value))
}
```

### Promise.reject

```javascript
Promise.reject = function(reason) {
  return new Promise((resolve, reject) => reject(reason))
}
```

### Promise.all

```javascript
Promise.all = function(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      throw new TypeError('参数必须是数组')
    }
    
    const results = []
    let completedCount = 0
    
    if (promises.length === 0) {
      resolve(results)
      return
    }
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise).then(value => {
        results[index] = value
        completedCount++
        if (completedCount === promises.length) {
          resolve(results)
        }
      }).catch(reject)
    })
  })
}
```

### Promise.race

```javascript
Promise.race = function(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      throw new TypeError('参数必须是数组')
    }
    
    promises.forEach(promise => {
      Promise.resolve(promise).then(resolve).catch(reject)
    })
  })
}
```

### Promise.allSettled

```javascript
Promise.allSettled = function(promises) {
  return new Promise(resolve => {
    const results = []
    let completedCount = 0
    
    if (promises.length === 0) {
      resolve(results)
      return
    }
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(value => {
          results[index] = { status: 'fulfilled', value }
        })
        .catch(reason => {
          results[index] = { status: 'rejected', reason }
        })
        .finally(() => {
          completedCount++
          if (completedCount === promises.length) {
            resolve(results)
          }
        })
    })
  })
}
```

### Promise.any

```javascript
Promise.any = function(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      throw new TypeError('参数必须是数组')
    }
    
    const errors = []
    let rejectedCount = 0
    
    if (promises.length === 0) {
      reject(new AggregateError([], 'All promises were rejected'))
      return
    }
    
    promises.forEach((promise, index) => {
      Promise.resolve(promise)
        .then(resolve) // 任何一个成功就 resolve
        .catch(error => {
          errors[index] = error
          rejectedCount++
          if (rejectedCount === promises.length) {
            reject(new AggregateError(errors, 'All promises were rejected'))
          }
        })
    })
  })
}
```

## 实际应用示例

### 网络请求封装

```javascript
class HttpClient {
  constructor(baseURL = '') {
    this.baseURL = baseURL
  }
  
  request(url, options = {}) {
    return new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest()
      const method = options.method || 'GET'
      const fullURL = this.baseURL + url
      
      xhr.open(method, fullURL)
      
      // 设置请求头
      if (options.headers) {
        Object.keys(options.headers).forEach(key => {
          xhr.setRequestHeader(key, options.headers[key])
        })
      }
      
      xhr.onload = () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          try {
            const data = JSON.parse(xhr.responseText)
            resolve(data)
          } catch (error) {
            resolve(xhr.responseText)
          }
        } else {
          reject(new Error(`HTTP Error: ${xhr.status}`))
        }
      }
      
      xhr.onerror = () => reject(new Error('Network Error'))
      xhr.ontimeout = () => reject(new Error('Request Timeout'))
      
      if (options.timeout) {
        xhr.timeout = options.timeout
      }
      
      xhr.send(options.body)
    })
  }
  
  get(url, options) {
    return this.request(url, { ...options, method: 'GET' })
  }
  
  post(url, data, options) {
    return this.request(url, {
      ...options,
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...options?.headers
      },
      body: JSON.stringify(data)
    })
  }
}
```

### 重试机制

```javascript
function retryPromise(promiseFactory, maxRetries = 3, delay = 1000) {
  return new Promise((resolve, reject) => {
    function attempt(retriesLeft) {
      promiseFactory()
        .then(resolve)
        .catch(error => {
          if (retriesLeft === 0) {
            reject(error)
          } else {
            console.log(`重试剩余次数: ${retriesLeft}, 错误:`, error.message)
            setTimeout(() => {
              attempt(retriesLeft - 1)
            }, delay)
          }
        })
    }
    
    attempt(maxRetries)
  })
}

// 使用示例
retryPromise(
  () => fetch('/api/data'),
  3,
  1000
).then(response => {
  console.log('请求成功:', response)
}).catch(error => {
  console.log('请求最终失败:', error)
})
```

### 并发控制

```javascript
class PromisePool {
  constructor(concurrency = 3) {
    this.concurrency = concurrency
    this.running = []
    this.queue = []
  }
  
  add(promiseFactory) {
    return new Promise((resolve, reject) => {
      this.queue.push({
        promiseFactory,
        resolve,
        reject
      })
      this.tryNext()
    })
  }
  
  tryNext() {
    if (this.running.length >= this.concurrency || this.queue.length === 0) {
      return
    }
    
    const { promiseFactory, resolve, reject } = this.queue.shift()
    const promise = promiseFactory()
      .then(resolve)
      .catch(reject)
      .finally(() => {
        const index = this.running.indexOf(promise)
        this.running.splice(index, 1)
        this.tryNext()
      })
    
    this.running.push(promise)
  }
}

// 使用示例
const pool = new PromisePool(2) // 最大并发数为 2

const urls = [
  '/api/data1',
  '/api/data2',
  '/api/data3',
  '/api/data4',
  '/api/data5'
]

const promises = urls.map(url => 
  pool.add(() => fetch(url).then(res => res.json()))
)

Promise.all(promises).then(results => {
  console.log('所有请求完成:', results)
})
```

### Promise 化工具

```javascript
// 将 Node.js 风格的回调函数转换为 Promise
function promisify(fn) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      fn.call(this, ...args, (err, result) => {
        if (err) {
          reject(err)
        } else {
          resolve(result)
        }
      })
    })
  }
}

// 使用示例
const fs = require('fs')
const readFileAsync = promisify(fs.readFile)

readFileAsync('file.txt', 'utf8')
  .then(content => console.log(content))
  .catch(error => console.error(error))
```

## 总结

Promise 的核心原理包括：

1. **状态管理**：三种状态的不可逆转换
2. **回调存储**：使用数组存储待执行的回调函数
3. **异步执行**：所有回调都在微任务队列中异步执行
4. **链式调用**：每个 `.then()` 返回新的 Promise
5. **错误传播**：错误会自动向下传播直到被捕获

理解这些原理有助于：
- 更好地使用 Promise API
- 避免常见的异步编程陷阱
- 编写更高效和可维护的异步代码
- 深入理解 async/await 的工作机制

Promise 是现代 JavaScript 异步编程的基础，掌握其原理对于成为一名优秀的 JavaScript 开发者至关重要。