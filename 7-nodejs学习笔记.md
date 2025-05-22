## **一、 为什么要使用nodejs**

BFF—Backends For Frontends 服务于前端的后端的一种web架构。

最初设计的后端api只服务于某一个端，而后来需发展撑多端。但此时api的架构和数据结构不满足多端进行展示，此时需要在BFF层进行数据组装或抽离。



## **二、 node.js的特点**

1. node.js 内置http服务器软件，这意味着在使用 Node.js 开发时，开发者可以直接使用 Node.js 的 HTTP 模块来创建服务器，处理 HTTP 请求，并生成 Web 页面。

2. node.js采用单线程处理多个并发请求
3. JavaScript 引擎独立于托管它的浏览器。这一关键特性促成了 Node.js 的崛起。

## **三、如何使用node.js**

1. #### **非阻塞机制**

   *阻塞代码示例*

   创建一个文件 input.txt ，内容为：你好。

   ```javascript
   var fs = require("fs");
   
   var data = fs.readFileSync('input.txt');
   
   console.log(data.toString());
   console.log("程序执行结束!");
   
   // 你好
   // 程序执行结束
   ```

2. #### **非阻塞代码示例**

   ```js
   var fs = require("fs");
   
   fs.readFile('input.txt', function (err, data) {
       if (err) return console.error(err);
       console.log(data.toString());
   });
   
   console.log("程序执行结束!");
   // 程序执行结束
   // 你好
   ```

   执行机制：

   调用 fs.readFile() 时，异步 I/O 操作 会被启动。Node.js 不会等待文件读取完成，而是会继续执行后面的同步代码（即 console.log('程序执行结束!')）。一旦文件读取完成，Node.js 会把回调函数 放入事件队列（也叫消息队列）中，等待事件循环去处理它。

3. ####  **事件循环**

   **事件循环的阶段**

   事件循环分为多个阶段，每个阶段处理特定的任务。关键阶段如下：

   - **Timers**：执行 `setTimeout()` 和 `setInterval()` 的回调。
   - **I/O Callbacks**：处理一些延迟的 I/O 回调。
   - **Idle, prepare**：内部使用，不常见。
   - **Poll**：检索新的 I/O 事件，执行与 I/O 相关的回调。
   - **Check**：执行 `setImmediate()` 回调。
   - **Close Callbacks**：处理关闭的回调，如 `socket.on('close', ...)`。

   **事件循环的流程**

   - 任务进入事件循环队列。
   - 事件循环按照阶段顺序进行处理，每个阶段有自己的回调队列。
   - 事件循环会在 `poll` 阶段等待新的事件到达，如果没有事件，会检查其他阶段的回调。
   - 如果 `setImmediate()` 和 `setTimeout()` 都存在，`setImmediate()` 在 `check` 阶段先执行，而 `setTimeout()` 在 `timers` 阶段执行。

   **宏任务与微任务**

   - **宏任务**：`setTimeout`、`setInterval`、`setImmediate`、I/O 操作等。
   - **微任务**：`process.nextTick`、`Promise.then`。

   **执行顺序：**微任务优先级高于宏任务，会在当前阶段的回调结束后立即执行。

   

4. #### **事件驱动程序**

   ![image-20241107095546462](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20241107095546462.png)

   

   **基本概念：**

- **事件**：在程序中发生的动作或状态改变，例如一个文件读取完成或一个 HTTP 请求到达。

- **事件触发器**：`EventEmitter` 是 Node.js 的内置模块，用来发出和监听事件。

- **事件处理器**：与事件关联的回调函数，事件发生时被调用。

  

  **事件驱动的流程：**

  - **注册事件**：在程序中通过 `EventEmitter` 实例注册事件和对应的处理器。
  - **触发事件**：当指定的事件发生时，`EventEmitter` 会触发该事件。
  - **处理事件**：事件循环会调度相应的回调函数来执行任务。

  EventEmitter 是 Node.js 中用于创建、注册和触发事件的核心模块。



5. #### Buffer(缓冲区)

JavaScript 语言自身只有字符串数据类型，没有二进制数据类型。

Node.js 中的 Buffer 类是用于处理二进制数据的核心工具，提供了对二进制数据的高效操作。

Buffer 类在处理文件操作、网络通信、图像处理等场景中特别有用。

**特性：**

- **二进制数据**：`Buffer` 对象是一个包含原始二进制数据的固定大小的数组。每个元素占用一个字节（8位），因此 `Buffer` 适合处理二进制数据，如文件内容、网络数据包等。
- **不可变性**：虽然 `Buffer` 对象的内容可以在创建后修改，但其长度是固定的，不能动态改变。

6. ####  Stream

Node.js 的 Stream 是一种处理流式数据的抽象接口，广泛应用于文件操作、网络通信等场景。

流式数据处理的一个主要优点是可以**在数据传输过程中就开始处理数据**，而不需要等待整个数据加载完毕，这使得 Node.js 能够高效地处理大量数据，而不会占用过多的内存。

![image-20241108134638448](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20241108134638448.png)

7. #### **module.exports 与 exports 的区别：**

   - `module.exports` 是导出对象的真正引用。
   - `exports` 是 `module.exports` 的快捷方式。不能直接赋值 `exports = ...`，否则会断开引用。

8. ####  require内部操作原理--加载过就缓存

   ![image-20241108150439353](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20241108150439353.png)

   
   
   4. #### node.js 路由
   
      在 Node.js 中，路由是处理 HTTP 请求的关键部分，它决定了如何根据不同的 URL 和 HTTP 方法（如 GET、POST、PUT、DELETE 等）来分发请求。
   
      Node.js 本身并没有内置的路由机制，但可以通过中间件库（如 Express）来实现。
   
      路由通常涉及以下几个方面：
   
      - **URL 匹配**：根据请求的 URL 来匹配路由规则。
      - **HTTP 方法匹配**：根据请求的 HTTP 方法（GET、POST、PUT、DELETE 等）来匹配路由规则。
      - **请求处理**：一旦匹配到合适的路由规则，就调用相应的处理函数来处理请求。