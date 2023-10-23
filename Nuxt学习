学习链接：https://juejin.cn/post/6997747254182281229

生命周期：

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b562a4e3a024240baa1399d40aaa879~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

  通过上面的流程图可以看出，当一个客户端请求进入的时候，服务端有通过 `nuxtServerInit` 这个命令执行在 `store` 的 `action` 中，在这里接收到客户端请求的时候，可以将一些客户端信息存储到 `store` 中，比如项目中的用户信息。`nuxtServerInit` 钩子只有 `store` 目录下的 `index.js` 有效，其他模块文件是无法调用的。