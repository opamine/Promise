# promise
基于 Promise/A+ 规范实现 Promise

```js
// 三个状态：PENDING、FULFILLED、REJECTED
const PENDING = 'PENDING';
const FULFILLED = 'FULFILLED';
const REJECTED = 'REJECTED';

const resolvePromise = (promise2, x, resolve, reject) => {
  // 自己等待自己完成是错误的实现，用一个类型错误，结束掉 promise - Promise/A+ 2.3.1
  if (promise2 === x) {
    return reject(new TypeError('Chaining cycle detected for promise #<Promise>'));
  }

  // Promise 的 executor 中只能调用一次 resolve 或 reject，再次调用无效 - Promise/A+ 2.3.3.3.3
  let called = false;

  // 判断 p1 的 then 方法定义的 onResolved 处理程序返回的是不是 Promise 实例
  // 这个 Promise 实例可能是本库创建的 Promise 实例，也可能是第三方 Promise 库实现的实例
  // 所以这样写是为了做兼容
  if ((typeof x === 'object' && x != null) || typeof x === 'function') {
    try {
      // try catch 是为了捕获 x.then 的异常，因为可通过 Object.defineProperty 重写 then 属性的 getter
      let then = x.then;
      if (typeof then === 'function') {
        // 这里调用 then 不要忘记 this 指回 x - Promise/A+ 2.3.3.3
        then.call(
          x,
          (y) => {
            if (called) return;
            called = true;
            // 递归解析的过程（因为可能 promise.then 的 onResolved 处理程序还可能返回 promise）- Promis /A+ 2.3.3.3.1
            resolvePromise(promise2, y, resolve, reject);
          },
          (r) => {
            // 只要内部 promise 失败，外部就算失败 Promise/A+ 2.3.3.3.2
            if (called) return;
            called = true;
            reject(r);
          }
        );
      } else {
        // 如果 x.then 是个普通值就直接返回 resolve 作为结果 - Promise/A+ 2.3.3.4
        resolve(x);
      }
    } catch (e) {
      // Promise/A+ 2.3.3.2
      if (called) return;
      called = true;
      reject(e);
    }
  } else {
    // 如果 x 是个非 Promise 对象，就直接返回 resolve 作为结果 - Promise/A+ 2.3.4
    resolve(x);
  }
};

class Promise {
  constructor(executor) {
    this.status = PENDING;
    this.value = undefined;
    this.reason = undefined;
    this.onResolvedCallbacks = [];
    this.onRejectedCallbacks = [];

    let resolve = (value) => {
      if (value instanceof Promise) {
        // 递归解析
        return value.then(resolve, reject);
      }

      if (this.status === PENDING) {
        this.status = FULFILLED;
        this.value = value;
        this.onResolvedCallbacks.forEach((fn) => fn());
      }
    };

    let reject = (reason) => {
      if (this.status === PENDING) {
        this.status = REJECTED;
        this.reason = reason;
        this.onRejectedCallbacks.forEach((fn) => fn());
      }
    };

    try {
      executor(resolve, reject);
    } catch (error) {
      reject(error);
    }
  }

  // Promise.prototype.then()
  then(onResolved_origin, onRejected_origin) {
    //解决 onFufilled，onRejected 没有传值的问题
    //Promise/A+ 2.2.1 / Promise/A+ 2.2.5 / Promise/A+ 2.2.7.3 / Promise/A+ 2.2.7.4
    const onResolved = typeof onResolved_origin === 'function' ? onResolved_origin : (v) => v;
    //因为错误的值要让后面访问到，所以这里也要抛出个错误，不然会在之后 then 的 resolve 中捕获
    const onRejected =
      typeof onRejected_origin === 'function'
        ? onRejected_origin
        : (err) => {
            throw err;
          };

    // 每次调用 then 都返回一个新的 promise  Promise/A+ 2.2.7
    let promise2 = new Promise((resolve, reject) => {
      // 若 p1 的状态为 FUlFILLED
      if (this.status === FULFILLED) {
        // Promise/A+ 2.2.2
        // Promise/A+ 2.2.4 -> Promise/A+ 3.1 --- setTimeout
        setTimeout(() => {
          try {
            // x 为 onResolved 处理程序真实返回的值 - Promise/A+ 2.2.7.1
            let x = onResolved(this.value);
            // x 可能是一个 promise
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            // Promise/A+ 2.2.7.2
            reject(e);
          }
        }, 0);
      }

      // 若 p1 的状态为 REJECTED
      if (this.status === REJECTED) {
        // Promise/A+ 2.2.3
        setTimeout(() => {
          try {
            // x 为 onRejected 处理程序真实返回的值 - Promise/A+ 2.2.7.1
            let x = onRejected(this.reason);
            // x 可能是一个 promise
            resolvePromise(promise2, x, resolve, reject);
          } catch (e) {
            reject(e);
          }
        }, 0);
      }

      // 若 p1 的状态为 PENDING
      // 将上面两种状态的处理程序进行收集，待 p1 落定后执行
      if (this.status === PENDING) {
        this.onResolvedCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onResolved(this.value);
              resolvePromise(promise2, x, resolve, reject);
            } catch (e) {
              reject(e);
            }
          }, 0);
        });

        this.onRejectedCallbacks.push(() => {
          setTimeout(() => {
            try {
              let x = onRejected(this.reason);
              resolvePromise(promise2, x, resolve, reject);
            } catch (e) {
              reject(e);
            }
          }, 0);
        });
      }
    });

    return promise2;
  }

  // Promise.prototype.catch()
  catch(onRejected) {
    return this.then(null, onRejected);
  }

  // Promise.prototype.finally()
  finally(callback) {
    return this.then(
      (value) => {
        return Promise.resolve(callback()).then(() => value);
      },
      (reason) => {
        return Promise.resolve(callback()).then(() => {
          throw reason;
        });
      }
    );
  }

  // Promise.resolve()
  static resolve = (value) => {
    return new Promise((resolve) => {
      resolve(value);
    });
  };

  // Promise.reject()
  static reject = (reason) => {
    return new Promise((_, reject) => {
      reject(reason);
    });
  };

  // Promise.all()
  static all = (values) => {
    if (!Array.isArray(values)) {
      const type = typeof values;
      return new TypeError(
        `TypeError: ${type} ${values} is not iterable (cannot read property Symbol(Symbol.iterator))`
      );
    }

    return new Promise((resolve, reject) => {
      let resultArr = [];
      let orderIndex = 0;
      const processResultByKey = (value, index) => {
        resultArr[index] = value;
        if (++orderIndex === values.length) {
          resolve(resultArr);
        }
      };
      for (let i = 0; i < values.length; i++) {
        let value = values[i];
        if (value && typeof value.then === 'function') {
          value.then((value) => {
            processResultByKey(value, i);
          }, reject);
        } else {
          processResultByKey(value, i);
        }
      }
    });
  };

  // Promise.race()
  static race = (promises) => {
    return new Promise((resolve, reject) => {
      for (let i = 0, len = promises.length; i < len; i++) {
        let val = promises[i];
        if (val && typeof val.then === 'function') {
          val.then(resolve, reject);
        } else {
          resolve(val);
        }
      }
    });
  };
}

Promise.defer = Promise.deferred = function () {
  let dfd = {};
  dfd.promise = new Promise((resolve, reject) => {
    dfd.resolve = resolve;
    dfd.reject = reject;
  });
  return dfd;
};

module.exports = Promise;

```
