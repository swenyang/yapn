# 组件库的单元测试

单元测试是保证代码质量的一种重要手段。与一般库的单元测试有所不同，UI 组件库中的组件涉及到大量渲染和 DOM 交互。组件库的单元测试主要目标包括：

1. 保证组件的渲染稳定，如果有所更改，应当是预期中的。
2. 保证组件的功能正常，需要模拟事件、DOM 交互等等；
3. 保证组件代码被充分测试，代码覆盖率尽可能达到 100%；

测试主要用到以下几个工具：

1. [Jest](https://facebook.github.io/jest/): 测试框架，用以测试函数、组件的 snapshot 等；
2. [Enzyme](http://airbnb.io/enzyme/): 用以测试 DOM 交互、事件模拟等。
3. [React Test Utilities](https://reactjs.org/docs/test-utils.html): 用以测试 Enzyme 无法解决的一些场景（比如 Portal 类）。

## Jest

Jest 测试框架的配置大部分按照 [Jest 官网](https://facebook.github.io/jest/docs/en/getting-started.html) 即可，有几点需要特殊配置一下。pacakge.json 中的 Jest 配置：

```json
"jest": {
  "moduleNameMapper": {
    "^COMPONENTS(.*)$": "<rootDir>/src/components$1",
    "^ICONS(.*)$": "<rootDir>/src/components/icon/images$1",
    "\\.css$": "identity-obj-proxy"
  },
  "transform": {
    "\\.(png|PNG|jpe?g|svg|gif)$": "<rootDir>/test/__mocks__/imageTransformer.js",
    "^.+\\.js$": "babel-jest"
  }
},
```

### Babel

如果你的 `.babelrc` 配置比较简单，按照 <https://facebook.github.io/jest/docs/en/getting-started.html#using-babel> 操作即可，同时在 package.json 里面增加 jest 配置让 `babel-jest` 处理：

```json
"^.+\\.js$": "babel-jest"
```

如果比较复杂，比如里面分了 `development` 和 `production` 配置：

```json
{
  "env": {
    "development": {
      // ...
    },
    "production": {
      // ...
    }
  }
}
```

需要在测试指令上指定 Node 的环境，比如：

```bash
export NODE_ENV=development && jest --coverage
```

### Webpack

参考 <https://facebook.github.io/jest/docs/en/webpack.html>，处理 Alias, CSS Modules, Static Assets 等。

#### CSS Modules

文档给的是用 `identity-obj-proxy` 模拟，`styles.root` 生成出来 `className="root"`，这个方案不够完美，不能够按照 webpack 的 loaders 规则。特别是组件的 DOM 嵌套比较多时，很多类名重复，比较难辨别。

这一点暂时没找到地方优化，主要的问题是在 Proxy 不能读取到文件的 context。

```json
"\\.css$": "identity-obj-proxy"
```

#### Static Assets(图片等）

静态资源比较好处理一些，我们只保留引用路径：

```json
"\\.(png|PNG|jpe?g|svg|gif)$": "<rootDir>/test/__mocks__/imageTransformer.js",
```

imageTransformer.js

```js
const path = require('path')

module.exports = {
    process(src, filename, config, options) {
        return 'module.exports = "images/' + (path.basename(filename)) + '";'
    },
}
```

#### Alias

把 webpack 配置中的 alias 迁移过来，可以用 `()` 和 `$n` 组合来匹配文件名：

```json
"^COMPONENTS(.*)$": "<rootDir>/src/components$1",
"^ICONS(.*)$": "<rootDir>/src/components/icon/images$1",
```

### [`react-test-renderer`](https://www.npmjs.com/package/react-test-renderer)

Jest 中的 snapshot 测试依赖于 `react-test-renderer`，使用时需要和你的 React 版本对应。可以用 `npm view your-lib time` 查看其历史发布版本：

```bash
npm view react-test-renderer time
```

然后根据你的 React 版本安装：

```bash
npm i react-test-renderer@15.4.2
```

## Enzyme

Enzyme 使用的是 3.3.0 版本，对于低版本的 React，需要额外的 adapter。

```bash
npm i --save-dev enzyme enzyme-adapter-react-15.4
```

**Comopnent.test.js**

```js
import Enzyme, { shallow } from 'enzyme'
import Adapter from 'enzyme-adapter-react-15.4'

Enzyme.configure({ adapter: new Adapter() })

// your test cases
```

Enzyme 有三种形式来渲染组件，各不相同。

### `shallow`

将组件作为一个单元测试，不受其他子节点的影响，但没有 DOM API 和组件的生命周期。主要用以测试 Functional Components 或者没有使用生命周期的组件。

### `mount`

可以调用 DOM 的 API 和组件的生命周期，注意一个测试文件共用一个 DOM，一个测试案例后最好清除 DOM 里面的内容。不同的测试文件 DOM 不会互相影响。

使用 `mount` 虽然可以访问 DOM 的 API，但是发现一个奇怪的现象，`document.body` 无法访问到 `mount` 挂载的组件。例如 ActionSheet 会通过 Portal 使用 `document.body.appendChild` 添加元素，`document.body.innerHTML` 只能访问到这部分动态添加的元素：

```js
const wrapper = mount(
    <div>
        <Button>Sample</Button>
        <ActionSheet
            show
            data={['选项一', '选项二', '选项三']}
            onChange={changeCallback}
            onCancel={cancelCallback}
        />
    </div>
)

console.log(document.body.innerHTML)
```

如果需要同时操作 wrapper 和 DOM 中的元素，可以用 wrapper 来操作 `mount` 部分，而 DOM API 则操作 Portal 的这部分。

每个 test case 后清除 DOM：

```js
afterEach(() => {
    document.head.innerHTML = ''
    document.body.innerHTML = ''
})
```

### `render`

将组件渲染到静态的 HTML，用以分析静态 HTML 的结构。使用的比较少，对于测试整个页面比较有用。

## React Test Utilities

在 Enzyme 的 `mount` 部分提到，有些 Portal 的元素无法通过 wrapper 访问到，这部分 DOM 元素可以通过 [`react-addons-test-utils`](https://reactjs.org/docs/test-utils.html) 来操作和模拟事件。

```bash
npm i react-addons-test-utils@15.4.2
```

## 测试指令

package.json 添加测试命令：

```json
"test": "export NODE_ENV=development && jest --coverage"
```

测试所有：

```bash
npm test
```

npm 脚本可以通过 `npm run script -- -xx` 形式添加运行参数。

更新组件 snapshot：

```bash
npm test -- -u
```

测试指定文件（实质上是对文件名进行字符串匹配）：

```bash
npm test -- Button
```

## 类别测试

### Snapshot

Jest 自带了 Snapshot 功能，可以用来保证组件 DOM 的稳定性：

```js
import React from 'react'
import renderer from 'react-test-renderer'
import { Button } from '../src/components/button/index'

describe('button.js', () => {
    test('type="primary"', () => {
        const component = renderer.create(
            <Button>Sample</Button>
        )
        const tree = component.toJSON()
        expect(tree).toMatchSnapshot()
    })
})
```

如果更改了组件的 DOM，需要运行 `npm test -- -u` 更新 Snapshot 文件。Snapshot 文件位于 `<testDir>/__snapshots/`，需要提交 git。

### Function

函数的测试目的是保证正常调用，尽可能覆盖到函数的每一行。我们可以用 Jest 的 [`jest.fn()`](https://facebook.github.io/jest/docs/en/jest-object.html#jestfnimplementation) 和 [`jest.spyOn()`](https://facebook.github.io/jest/docs/en/jest-object.html#jestspyonobject-methodname) 来监听函数的调用情况。

#### 普通函数

例如，测试回调是否被正常调用：

```js
import React from 'react'
import Enzyme, { shallow } from 'enzyme'
import Adapter from 'enzyme-adapter-react-15.4'
import injectTapEventPlugin from 'react-tap-event-plugin'
import { Button } from '../src/components/button/index'

injectTapEventPlugin()
Enzyme.configure({ adapter: new Adapter() })

describe('button.js', () => {
    test('click', () => {
        const callback = jest.fn()
        const wrapper = shallow(
            <Button onTouchTap={callback}>Sample</Button>
        )
        wrapper.find('a').first().simulate('touchTap')
        expect(callback.mock.calls.length).toBe(1)
    })
})
```

#### 组件的成员函数（prototype）

组件的成员函数以 prototype 形式写的：

```js
export default class Button extends React.Component {
    constructor() {
        this.handleTouchTap = this.handleTouchTap.bind(this)
    }
    
    handleTouchTap() {
        // ...
    }

    render() {
        // ...
    }
}
```

可以用如下形式测试：

```js
import React from 'react'
import Enzyme, { shallow } from 'enzyme'
import Adapter from 'enzyme-adapter-react-15.4'
import { Button } from '../src/components/button/index'

Enzyme.configure({ adapter: new Adapter() })

describe('button.js', () => {
    test('render', () => {
        const handleTouchTap = jest.spyOn(Button.prototype, 'handleTouchTap')
        const render = jest.spyOn(Button.prototype, 'render')
        const wrapper = shallow(
            <Button>Sample</Button>
        )
        expect(handleTouchTap.mock.calls.length).toBe(0)
        expect(render.mock.calls.length).toBe(1)
        handleTouchTap.mockRestore()
        render.mockRestore()
    })
})
```

在 test case 最后需要通过 `mockFn.mockRestore()` 清除在 prototype 绑定的 mockFn。

#### 组件的成员函数（class properties）

组件的成员以箭头函数形式写的：

```js
export default class Button extends React.Component {
    handleTouchTap = () => {
        // ...
    }
}
```

可以用如下形式测试：

```js
import React from 'react'
import Enzyme, { shallow } from 'enzyme'
import Adapter from 'enzyme-adapter-react-15.4'
import injectTapEventPlugin from 'react-tap-event-plugin'
import { Button } from '../src/components/button/index'

injectTapEventPlugin()
Enzyme.configure({ adapter: new Adapter() })

describe('button.js', () => {
    test('click', () => {
        const wrapper = shallow(
            <Button>Sample</Button>
        )
        const handleTouchTap = jest.spyOn(wrapper.instance(), 'handleTouchTap')
        wrapper.instance().forceUpdate()
        wrapper.update()
        wrapper.find('a').first().simulate('touchTap')
        wrapper.find('a').first().simulate('touchTap')
        expect(handleTouchTap.mock.calls.length).toBe(2)
    })
})
```

#### Why Difference?

见 [class 成员函数不同写法的区别](/class-member-functions)。

### Event

要模拟组件的事件，需要通过 Enzyme 的 [Selector](http://airbnb.io/enzyme/docs/api/selector.html) 找到对应的元素，然后发送模拟事件。

如上一节函数展示的，点击事件是通过 Enzyme 的 `simulate` 函数模拟的。组件库中大部分组件的点击事件为 `onTouchTap`，模拟这个事件之前需要调用 `injectTapEventPlugin`，否则会报错（找不到 `touchTap` 事件）：

```js
wrapper.find('a').first().simulate('touchTap')
```

如果是普通的 DOM 事件，可以直接传递，无需额外配置：

```js
wrapper.find('a').first().simulate('click')
wrapper.find('a').first().simulate('touchstart')
```

### setTimeout

如果组件内用到了 `setTimeout` 函数进行延时的，可以通过 `expect.assertions` 和 `Promise` 结合保证延时部分被执行。

例如，Button 组件有一个延时操作：

```js
handleTouchTap = (e) => {
    // ...
    setTimeout(() => {
        this.mInternalLock = false
    }, 50)
    // ...
}
```

为保证 `this.mInternalLock = false` 能被测试到，可以写如下代码：

```js
import React from 'react'
import renderer from 'react-test-renderer'
import Enzyme, { shallow } from 'enzyme'
import Adapter from 'enzyme-adapter-react-15.4'
import injectTapEventPlugin from 'react-tap-event-plugin'
import { Button } from '../src/components/button/index'
import { timerPromise } from './utils/timerPromise'

injectTapEventPlugin()
Enzyme.configure({ adapter: new Adapter() })

describe('button.js', () => {
    test('click', () => {
        expect.assertions(2) // 指示 Jest 有两个期望断言
        const callback = jest.fn()
        const wrapper = shallow(
            <Button onTouchTap={callback}>Sample</Button>
        )
        const handleTouchTap = jest.spyOn(wrapper.instance(), 'handleTouchTap')
        wrapper.instance().forceUpdate()
        wrapper.update()
        wrapper.find('a').first().simulate('touchTap')
        wrapper.find('a').first().simulate('touchTap')
        return timerPromise(100).then(() => { // 延时100毫秒的 Promise
            expect(callback.mock.calls.length).toBe(1)
            expect(handleTouchTap.mock.calls.length).toBe(2)
        })
    })
})
```

### Portal

由于 Portal 会把通过 `document.body.appendChild` 将子元素渲染到新增节点上，通过 Enzyme wrapper 中不能获取子元素，`react-test-renderer` 也无法获取子元素的 snapshot。

#### 方案一: `ReactWrapper`

1. 找到渲染到 `document.body` 的子元素 DOM 节点
1. 根据 DOM 节点找到 React 节点
2. 把 React 节点重新封装为 ReactWrapper（即 `mount` 返回的 wrapper）
3. 调用 ReactWrapper 接口进行操作

这个方案的好处是能测试 DOM、模拟事件等。第二步在 Enzyme 3.0 以下是可以用 [`new ReactWrapper`](https://github.com/airbnb/enzyme/issues/252) 实现的；不幸的是，在Enzyme 3.0+ 就失效了，见 [Issue 1202](https://github.com/airbnb/enzyme/issues/1202)，暂时没找到解决方案。

#### 方案二： [`react-addons-test-utils`](https://reactjs.org/docs/test-utils.html)

1. 找到渲染到 `document.body` 的子元素 DOM 节点
2. 利用 `react-addons-test-utils` 进行 Simulate 操作。

这个方案虽然可以 work，但是要对比 DOM 就比较麻烦，在方案一不能用的情况下作为权宜之计。

```js
// ...
import ReactTestUtils from 'react-addons-test-utils'

const wrapper = mount(
    <Test />
)
wrapper.find('#button').first().simulate('touchTap')
expect(document.body.innerHTML).toBe('<div><!-- your DOM here --></div>')
ReactTestUtils.Simulate.touchTap(document.querySelector('.cancel'))
expect(document.body.innerHTML).toBe('')
```
