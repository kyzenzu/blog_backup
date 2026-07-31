---
title: Promise大致实现
date: 2024-1-21
tags: 
  - JS
---

~~~javascript
class Promise {
    static PENDING = "待定"; static FULFILLED = "成功"; static REJECTED = "拒绝";
    constructor(func) {
        this.status = Promise.PENDING; // status作为Promise的状态
        this.result = null; // result作为Promise的结果值
        this.resolveCallbacks = [];
        this.rejectCallbacks = [];
        try {
            // 在通过执行func来调用resolve或reject方法时
            // 由于func定义在类的外面，所以func调用这两个方法后，
            // 这两个方法中的this不再是promise对象
            // 所以需要bind来生成一个绑定this的函数
            func(this.resolve.bind(this), this.reject.bind(this));
        } catch (error) { // 可能会在自定义的func函数中抛出错误
            this.reject(error);
        }
    }
    resolve(result) { // resolve方法是异步方法
        setTimeout(() => {
            if (this.status === Promise.PENDING) {
                this.status = Promise.FULFILLED;
                this.result = result;
                // 如果then方法先被先于resolve方法被调用
                // 那么待执行数组里的函数就是then要执行的函数
                // 由resolve方法代替then去逐个执行待执行数组中的函数
                this.resolveCallbacks.forEach(callback =>  {
                    callback(result);
                })
            }
        })
    }
    reject(result) { // reject方法是异步方法
        setTimeout(() => {
            if (this.status === Promise.PENDING) {
                this.status = Promise.REJECTED;
                this.result = result;
                // 如果then方法先被先于reject方法被调用
                // 那么待执行数组里的函数就是then要执行的函数
                // 由reject方法代替then去逐个执行待执行数组中的函数
                this.rejectCallbacks.forEach(callback =>  {
                    callback(result);
                })
            }
        })
    }
    then(onFULFILLED, onREJECTED) {  
        onREJECTED = typeof onREJECTED === 'function' ? onREJECTED : () => {};
        onFULFILLED = typeof onFULFILLED === 'function' ? onFULFILLED : () => {};
        if (this.status === Primise.PENDING) {
            // 如果执行then时,promise对象的的状态还没出来,
            // 就把要执行的两个自定义函数推入待执行数组中
            this.resolveCallbacks.push(onFULFILLED);
            this.rejectCallbacks.push(onREJECTED);
        };
        if (this.status === Promise.FULFILLED) {
            setTimeout(() => {
                onFULFILLED(this.result);
            })
        }
        if (this.status === Promise.REJECTED) {
            setTimeout(() => {
                onREJECTED(this.result);
            })
        }
        return this; // 这里还需要返回一个promise对象实现链式调用
                     // 但不是像这样直接返回自己需要新建一个promise对象
                     // 具体还有待开发
    }
}

let promise = new Promise((resolve, reject) => {
    // 在创建promise对象时，调用Promise类自带的resolve和reject两个方法
    // 来确定promise对象的状态和结果值
    resolve("一切正常");
})
promise.then( // then方法根据promise对象状态来选择要执行的自定义函数
    result => {console.log(result)},
    result => {console.log(result.message)}
);
~~~

