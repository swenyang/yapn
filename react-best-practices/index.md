# React最佳实践

> Written at July 2017, need update

本篇文档旨在总结过去接近一年使用React重构项目的经验，得出我们的最佳实践。最佳实现可以作为我们的一个开发指引和代码规范，也希望对大家有所启发。

## 开发语言

- 使用ES6和ES7（包括[类成员](https://babeljs.io/docs/plugins/transform-class-properties/)，[对象扩展操作符](https://babeljs.io/docs/plugins/transform-object-rest-spread/)，[decorator](https://babeljs.io/docs/plugins/transform-decorators/)，[async](https://babeljs.io/docs/plugins/transform-async-to-generator/)等插件）作为开发语言。
- 使用React JSX编写组件的DOM标记。

## 组件设计

组件设计上很重要的一个原则是区分**展示组件**（Presentational Component）和**容器组件**（Container Component）。关于两者的概念可以参阅[Dan Abramov的文章](https://medium.com/@dan_abramov/smart-and-dumb-components-7ca2f9a7c7d0#.a08zftszf)。大体来说展示组件是处理UI展示的，而容器组件是处理业务逻辑的。

> **展示**组件：
> - 关注于**如何显示**。
> - 内部可能包含展示组件和容器组件，通常有自己的一些DOM标记和样式。
> - 一般允许通过`this.props.children`来被包含。
> - 不指定如何加载或修改数据。
> - 只通过`props`来接受数据和回调。
> - 很少会有自己的`state`（如果有，应当是UI的状态而不是数据）。
> - 除非需要`state`、生命周期钩子或性能优化，否则都实现为[函数组件](https://facebook.github.io/react/blog/2015/10/07/react-v0.14.html#stateless-functional-components)。
> - 样例：*Page, Sidebar, Story, UserInfo, List*。
> 
> **容器**组件：
> - 关注与**如何工作**。
> - 内部可能包含展示组件和容器组件，但通常除了一些包装用的`div`，没有任何自己的DOM标记，也没有任何样式。
> - 为展示组件和其他容器组件提供数据和行为。
> - 调用Flux动作，提供这些动作作为展示组件的回调。
> - 一般是有状态的，因为他们更多的是作为数据源。
> - 通常使用[高阶组件](https://medium.com/@dan_abramov/mixins-are-dead-long-live-higher-order-components-94a0d2f9e750)生成，比如React Redux的`connect()`，Relay的`createContainer ()`，或Flux Utils的`Container.create() `，而不是从零编写。
> - 样例：*UserPage, FollowersSidebar, StoryContainer, FollowedUserList*。

在项目中划分这两种组件非常有益。
- 关注点的分离。编写容器组件时，主要处理业务逻辑，而展示组件则主要处理UI显示。这也使我们获得更好的可维护性。
- 更好的复用性。不关心业务逻辑的展示组件可以复用在不同的容器组件里面。

> 记住一点，**组件不一定要输出DOM**。他们只需要提供UI问题间的组合边界。

## 组件编写

### ESLint

我们使用[基于AirBnB的代码检查](https://github.com/PALifeH5/eslint-standard)，其中包含了[eslint-plugin-react](https://github.com/yannickcr/eslint-plugin-react)，确保每个人写的代码风格大体一致。

### 组件样式

我们使用[CSS Modules](https://github.com/css-modules/css-modules)来编写组件样式。

**Example.css**

```css
.red-color {
    color: #f00;
}
```

**Example.js**

```js
import React, { Component } from 'react'
import styles from './Example.css'

export default class Example extends Component {
    render() {
        return <p style={styles['red-color']}>This is an example.</p>
    }
}
```

使用CSS Modules的主要优点是：
- 模块化的CSS，可复用。
- 没有样式命名冲突问题。
- 组件依赖的样式是显示指定的，可推断。
- 局部作用域。
- 更适于代码分块。

另外一方面，CSS Modules同样像LESS和SASS支持Import、嵌套、变量、mixins等灵活的功能，使用方便。配合[postcss](https://github.com/postcss/postcss)自动处理前缀等兼容性问题，使得开发过程中更关注于样式。

### JSX

JSX是在JS中表达组件树很好的形式，其中可以夹带逻辑判断，但是夹带多了组件树就会变得很难阅读，特别是三元操作符`?:`。

三元操作符可以用`&&`判断改写，使代码变得简洁一些，比如下面两种渲染`ComponentA`部分是一样的：

```js
render() {
    const { conditionA } = this.props
    return (
        <div>
            {conditionA ? <ComponentA /> : null}
            {conditionA && <ComponentA />}
        </div>
    )
}
```

不允许嵌套多层的三元操作符，如果需要多层判断，可以拆出为单独函数，也可以在JSX里面用[立即执行函数表达式（IIFE）](https://en.wikipedia.org/wiki/Immediately-invoked_function_expression)，比如下面的`renderButton()`和IIFE部分是等同的：

```js
renderButton(conditionA, conditionB) {
    if (conditionA && conditionB) {
        return <ComponentB />
    }
    else if (conditionA) {
        return <ComponentA />
    }
    return <ComponentC />
}

render() {
    const { conditionA, conditionB } = this.props
    return (
        <div>
            {this.renderButton(conditionA, conditionB)}
            {
                (() => {
                    if (conditionA && conditionB) {
                        return <ComponentB />
                    }
                    else if (conditionA) {
                        return <ComponentA />
                    }
                    return <ComponentC />
                })()
            }
        </div>
    )
}
```

另外，访问`props`和`state`中的属性使用解构赋值，使得代码简洁易读。

```js
const { propA, propB, propC, propD } = this.props
const { stateA, stateB, stateC } = this.state
```

### 基于类的组件

如前文所提，基于类的组件是有状态的或包含了生命周期方法的。

#### `propTypes`和`defaultProps`

我们通过静态类成员定义`propTypes`和`defaultProps`，让组件要求的`props`和默认值更合理地包含在类里面。

```js
import React, { Component, PropTypes } from 'react'

export default class Example extends Component {
    static propTypes = {
        title: PropTypes.string.isRequired,
    }

    static defaultProps = {
        title: 'Hello World!',
    }

    render() {
        return (
            <div>
                <h1>{this.props.title}</h1>
                <p>This is an example.</p>
            </div>
        )
    }
}
```

#### 初始化state

通过声明为类成员初始化组件的`state`：

```js
import React, { Component, PropTypes } from 'react'

export default class Example extends Component {
    state = { isCollapsed: false }

    render() {
        return (
            <div>
                <p>This is an example.</p>
                {
                    !this.state.isCollapsed && <p>More detail.</p>
                }
            </div>
        )
    }
}
```

#### 事件处理函数

组件的事件处理函数处理有几点原则：

- `handle`前缀表示组件内部的事件处理，`on`前缀表示暴露给父组件的事件处理（也即`props`）。
- 事件处理需要访问组件的`this`的，可以用箭头函数编写成员函数。
- 尽量使用成员函数，避免在JSX直接写箭头函数引入新的闭包。

样例阐释如下：

```js
import React, { Component, PropTypes } from 'react'

export default class Example extends Component {
    static propTypes = {
        title: PropTypes.string.isRequired,
        onCollapse: PropTypes.func.isRequired,
    }

    handleClick = () => {
        console.log(this.props.title)
    }

    render() {
        return (
            <div>
                <a onClick={this.handleClick}>Internal component.</a>
                <a onClick={this.props.onCollapse}>Exposed to parent.</a>
                <a onClick={() => console.log('new closure')}>Introduce a new closure.</a>
            </div>
        )
    }
}
```

#### 成员变量和refs

成员变量以`m`前缀形式命名，代表类的私有变量。如需要建立组件的refs，推荐用函数形式声明为成员变量，而不是用字符串定义`ref`然后通过`this.refs.xxx`访问。

```js
import React, { Component, PropTypes } from 'react'

export default class Example extends Component {
    mParagraph

    render() {
        return (
            <div>
                <p ref={(elem) => { this.mParagraph = elem }}>
                    This is an example.
                </p>
            </div>
        )
    }
}
```

### 无状态函数组件

无状态函数组件没有`state`，也不调用生命周期方法，是**推荐**的组件实现方式，因为其易读、易推断、易维护。

#### `propTypes`和`defaultProps`

无状态函数组件的`propTypes `和`defaultProps`不能像基于类的组件写在组件内部，需要写在定义之后。

```js
const Example = (props) => (
    <div>
        <h1>props.title</h1>
    </div>
)

Example.propTypes = {
    title: PropTypes.string,
}

Example.defaultProps = {
    title: 'Hello World'
}
```

使用解构赋值能使无状态函数组件的`props`和默认值更简洁：

```js
const Example = ({ title = 'Hello World', onCollapse, onExpand }) => (
    <div>
        <h1>props.title</h1>
        <a onClick={onCollapse}>Collapse</a>
        <a onClick={onExpand}>Expand</a>
    </div>
)

Example.propTypes = {
    title: PropTypes.string,
    onCollapse: PropTypes.func,
    onExpand: PropTypes.func,
}
```

### Decorator

ES7的Decorator可以用来灵活、静态地修改组件功能，在某些场景比如构建高阶组件时很有帮助。

使用Decorator有三种方式：

- 基于类的组件，用Decorator语法

	```js
    @deco
    class Example extends Component {
        // ...
    }
    ```

- 基于类的组件，不使用Decorator语法

    ```js
    class Example extends Component {
        // ...
    }
    
    export default deco(Example)
    ```

- 无状态函数组件

    ```js
    const Example = (props) => {
        // ...
    }
    
    export default deco(Example)
    ```

## 总结

本篇React最佳实践是在开发过程中慢慢完善的，当然也难免会有纰漏，如果你有更好的建议，欢迎提交PR给我们。

