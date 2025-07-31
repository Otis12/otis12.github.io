---
title: JavaScript异步编程学习笔记
date: 2025-07-31 15:23:15
categories: [技术学习]
tags: [JavaScript, 异步编程, Promise, async/await]
---

# JavaScript异步编程全面解析

## 为什么需要异步编程？

JavaScript是单线程的，如果同步执行耗时操作会阻塞主线程，导致页面卡顿。异步编程可以让程序在等待操作完成时继续执行其他代码。

## 异步编程的发展历程

### 1. 回调函数 (Callback)
```javascript
// 传统回调方式
setTimeout(() => {
    console.log("1秒后执行");
    setTimeout(() => {
        console.log("2秒后执行");
        setTimeout(() => {
            console.log("3秒后执行");
        }, 1000);
    }, 1000);
}, 1000);
```

**问题**: 回调地狱 (Callback Hell)

### 2. Promise
```javascript
// Promise方式
function delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
}

delay(1000)
    .then(() => {
        console.log("1秒后执行");
        return delay(1000);
    })
    .then(() => {
        console.log("2秒后执行");
        return delay(1000);
    })
    .then(() => {
        console.log("3秒后执行");
    });
```

**优点**: 链式调用，避免回调地狱

### 3. async/await
```javascript
// async/await方式
async function asyncDemo() {
    await delay(1000);
    console.log("1秒后执行");
    
    await delay(1000);
    console.log("2秒后执行");
    
    await delay(1000);
    console.log("3秒后执行");
}

asyncDemo();
```

**优点**: 代码更像同步代码，易于理解

## 实际应用场景

### 网络请求
```javascript
// 使用fetch API
async function fetchUserData(userId) {
    try {
        const response = await fetch(`/api/users/${userId}`);
        const userData = await response.json();
        return userData;
    } catch (error) {
        console.error('获取用户数据失败:', error);
        throw error;
    }
}
```

### 并发处理
```javascript
// 并发执行多个异步操作
async function fetchMultipleData() {
    const [users, posts, comments] = await Promise.all([
        fetch('/api/users'),
        fetch('/api/posts'),
        fetch('/api/comments')
    ]);
    
    return {
        users: await users.json(),
        posts: await posts.json(),
        comments: await comments.json()
    };
}
```

## 常见陷阱和最佳实践

### 1. 错误处理
```javascript
// ❌ 错误的做法
async function badErrorHandling() {
    const data = await fetch('/api/data'); // 如果失败，会抛出未捕获的异常
    return data.json();
}

// ✅ 正确的做法
async function goodErrorHandling() {
    try {
        const response = await fetch('/api/data');
        if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
        }
        return await response.json();
    } catch (error) {
        console.error('请求失败:', error);
        return null;
    }
}
```

### 2. 循环中的异步操作
```javascript
// ❌ 错误的做法（顺序执行）
async function processItemsSequentially(items) {
    for (const item of items) {
        await processItem(item); // 一个一个处理，效率低
    }
}

// ✅ 正确的做法（并行执行）
async function processItemsConcurrently(items) {
    const promises = items.map(item => processItem(item));
    await Promise.all(promises); // 同时处理所有项目
}
```

## 学习总结

1. **理解异步的本质**: JavaScript的单线程特性
2. **掌握三种模式**: Callback → Promise → async/await
3. **注意错误处理**: 总是要有适当的错误处理机制
4. **性能优化**: 合理使用并发处理提高效率

## 下一步学习计划

- [ ] 深入学习Event Loop机制
- [ ] 了解Web Workers
- [ ] 学习RxJS响应式编程
- [ ] 实践Node.js异步编程

---
**学习时间**: 3小时  
**实践项目**: Todo应用的异步数据获取  
**掌握程度**: ⭐⭐⭐⭐
