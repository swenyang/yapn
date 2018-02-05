# class 成员函数不同写法的区别

## 两种不同写法

ES6 新增了 class 定义，我们可以在 class 中定义成员函数。

```js
class Test {
    foo() {
        // ...
    }
}
```

但是，我们在写组件类时，经常要给元素绑定事件，这样就要给成员函数绑定 `this`，使得事件回调中能访问到组件本身：

```js
class Test {
    constructor() {
        this.foo = this.foo.bind(this)
    }

    foo() {
        // ...
    }
}
```

这样的写法很冗长，为了简便我们会使用箭头函数来替代绑定，如下。当然，这种写法其实已经不是成员函数的定义了，而是为 class 定义了一个成员变量。我们需要开启 [`babel-plugin-transform-class-properties`](http://babeljs.io/docs/plugins/transform-class-properties/) 来支持这种写法。

```js
class Test {
    bar = () => {
        // ...
    }
}
```

## 生成代码分析

很明显，这两种写法的生成代码是不一样的。成员函数是生成在 class 的 prototype 上的；而箭头函数是在 class 实例化之后添加的函数。比如下面的 Test 类：

**Test.js**

```js
export default class Test {
    constructor() {
        this.bind = this.bind.bind(this)
    }

    bar = () => {
        console.log('bar')
    }

    foo() {
        console.log('foo')
    }

    func() {
        console.log('func')
    }
}
```

babel 转译后生成代码：

```js
var Test = function () {
    function Test() {
        _classCallCheck(this, Test);

        this.bar = function () {
            console.log('bar');
        };

        this.foo = this.foo.bind(this);
    }

    _createClass(Test, [{
        key: 'foo',
        value: function foo() {
            console.log('foo');
        }
    }, {
        key: 'func',
        value: function func() {
            console.log('func');
        }
    }]);

    return Test;
}();
```

而 `_createClass` 是以 `Object.defineProperty` 形式写入 class prototype 的：

```js
var _createClass = function () {
    function defineProperties(target, props) {
        for (var i = 0; i < props.length; i++) {
            var descriptor = props[i];
            descriptor.enumerable = descriptor.enumerable || false;
            descriptor.configurable = true;
            if ("value" in descriptor) descriptor.writable = true;
            Object.defineProperty(target, descriptor.key, descriptor);
        }
    }
    return function (Constructor, protoProps, staticProps) {
        if (protoProps) defineProperties(Constructor.prototype, protoProps);
        if (staticProps) defineProperties(Constructor, staticProps);
        return Constructor;
    };
}();
```

两种不同写法会导致在写测试代码时有所区别。虽然箭头函数的形式比较简洁，但是会给测试代码造成一些麻烦，Enzyme 的维护者 ljharb 则表示[避免用箭头函数的形式](https://github.com/airbnb/enzyme/issues/365#issuecomment-362168115)：

> Spying on instance methods won’t work properly if it’s using arrow functions in class properties - avoid them; use constructor-bound instance methods instead.
