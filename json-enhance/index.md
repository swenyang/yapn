# json-enhance: 扩展 `JSON.stringify` 和 `JSON.parse`

## 引言

从 1995 年 Javascript 诞生到现在，Web App 已经蓬勃发展了约20年。起初 Web App 都是多页应用的形式，直到后来 AJAX 技术的诞生，单页应用开始占据统治地位。单页应用促进了前后端分离，使 Web App 有了持续的状态管理，随着交互和应用复杂性增长，单页应用的规模可能也越来越大。

如果我们把多页应用理解为多个入口的 Web App 而不仅仅是后端吐出的页面，实质上，单页与多页之间的界限逐渐模糊，因为业务很可能在多个单页应用间跳转。例如，利用第三方账户体系进行登录、注册，或者在购物网站完成后跳转至第三方支付网站。可能是一个域名跳转到另外一个域名，也有可能是同一个域名不同子目录。

这些场景给单页应用带来了问题：Web App 持续的状态丢失了。从第三方网站回来后，用户之前操作的数据如果没有存储在服务器将丢失；如果存储在服务器，有一些状态数据（比如组件状态，勾选状态等）是冗余的，另外让后端与前端逻辑耦合也会造成额外的工作量。

## 应用状态管理

得益于应用状态管理框架的发展，比如 Redux, Flux, MobX 等，我们可以把应用状态收敛到一个地方进行管理，"Single Source of Truth" 的思想让状态的改变、回退、甚至恢复都变得简单。我们可以在单页应用跳转之前把状态序列化后存储到 localStorage 或者 sessionStorage 中，等下次应用初始化时读取状态进行恢复。

如果你的应用状态管理、页面逻辑处理的足够好，是完全可以通过 `JSON.stringify` 和 `JSON.parse` 完成状态恢复工作。但是在一些情况下应用状态难免会包含一些特殊的数据：比如 Date 对象，正则表达式，`undefined`，甚至还有 getter/setter、函数、自定义的对象，`JSON.*` 就无法处理了，这些数据或者丢失，或者需要我们做额外的处理进行恢复。

## json-enhance

为了更方便的对应用状态进行暂存和恢复，我实现了 [json-enhance](https://github.com/swenyang/json-ext) 库扩展 `JSON.stringify` 和 `JSON.parse` 函数使其支持更多的数据类型。`stringify` 流程如下（`parse` 的流程相反）：

1. 遍历状态树的所有子节点，如果发现子节点是需要特殊数据类型，则增加一条特殊记录，包含路径、恢复用的数据等；
2. 把处理过的状态树和特殊记录重新用 JSON 对象封装；
3. 调用原生的 `JSON.stringify` 序列化数据。

这样，我们就能**自动把数据恢复到原始数据，不需要特殊处理，也不需要服务器配合。**

```js
import { stringify, parse } from 'json-enhance'

const obj = {
    a: 1,
    b: true,
    c: [
        '123',
        new Date(2018, 0, 1),
        { x: '123' },
        (a, b) => a - b,
    ],
    d: undefined,
    e: /\/(.*?)\/([gimy])?$/,
    f: (a, b) => a + b,
}
Object.defineProperty(obj, 'x', {
    get() {
        return this.mX
    },
    set(v) {
        this.mX = v
    },
})
const str = stringify(obj)
parse(str) // equals obj
```

为了正确比较序列化前和序列化后的对象，我扩展了 [deep-equal-ext](https://github.com/swenyang/deep-equal-ext) 来比较新增的特殊数据。

### 特殊数据处理

假设原始数据为 `obj`，恢复后的数据为 `newObj`，恢复数据时读取的字符串为 `str`

#### Date

- Identify:

    ```js
obj instanceof Date
    ```
    
- `stringify`:

    ```js
obj.getTime()
    ```

- `parse`:

    ```js
new Date(str)
    ```
    
- Compare:

    ```js
obj.getTime() === newObj.getTime()
    ```
    
#### `undefined`    

- Identify:

    ```js
obj instanceof Date
    ```
    
- `stringify`:

    ```js
obj.getTime()
    ```

- `parse`:

    ```js
new Date(str)
    ```
    
- Compare:

    ```js
obj === newObj
    ```
    
#### Function    

- Identify:

    ```js
typeof obj === 'function'
    ```
    
- `stringify`:

    ```js
obj.toString()
    ```

- `parse`:

    ```js
eval(str)
    ```
    
- Compare:

    ```js
obj.toString() === newObj.toString()
    ```
    
#### RegExp    

- Identify:

    ```js
obj instanceof RegExp
    ```
    
- `stringify`:

    ```js
obj.toString()
    ```

- `parse`:

    ```js
const fragments = str.match(/\/(.*?)\/([gimy])?$/)
return new RegExp(fragments[1], fragments[2] || '')
    ```
    
- Compare:

    ```js
obj.toString() === newObj.toString()
    ```
    
#### Getter/Setter    

- Identify:

    ```js
Object.getOwnPropertyNames(obj).forEach((key) => {
    const descriptor = Object.getOwnPropertyDescriptor(obj, key) 
    if (descriptor.get || descriptor.set) {
        // ...
    }  
}
    ```
    
- `stringify`:

    ```js
descriptor.get.toString()
descriptor.set.toString()
    ```

- `parse`:

    ```js
newObj.__defineGetter__(key, eval(str))
newObj.__defineSetter__(key, eval(str))
    ```
    
- Compare:

    ```js
obj.toString() === newObj.toString()
    ```

### 缺陷

1. 函数是通过 `toString()` 后比较的，不是功能意义上的比较。
2. 无法恢复函数内的闭包引用，请尽可能使用纯函数。
3. Getter/Setter 恢复有写法上的限制。
