# Javascript 中该如何使用 `new Date`

日期处理基本上是每一个工程师都会面临的问题，而在前端开发中由于对日期格式的不了解、或者浏览器的兼容性问题，容易踩到日期相关的坑。本文重点介绍一下 Javascript 中创建日期（即 `new Date`）的正确方式。

## 创建日期对象的四种方式

Javascript 的 `Date` 对象代表了某一个时间，时间的值是基于1970年1月1日（UTC时区）来计算的。根据 [MDN Date 文档](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date) ,合法的构造函数格式如下：

```js
new Date();
new Date(value);
new Date(dateString);
new Date(year, month[, date[, hours[, minutes[, seconds[, milliseconds]]]]]);
```

> 注意：Javascript `Date` 对象只能通过调用 `Date` 作为构造函数来实例化：如果是以普通函数调用（即没有 `new` 操作符），将会返回一个当前时间的字符串而不是 `Date` 对象；跟其他 Javascript 对象类型不同，`Date` 对象没有字面量格式。

总共有四种方式来实例化一个 `Date` 对象。如果参数合法，将返回一个新的 `Date` 对象，否则返回一个 `Invalid Date`（即 `NaN`）。

```js
> new Date()
< Mon Jul 31 2017 16:53:37 GMT+0800 (CST)

> new Date('invalid')
< Invalid Date

> isNaN(new Date('invalid'))
< true
```

### `new Date()`

没有提供任何参数的情况下，将会根据系统设置创建一个 `Date` 对象，代表当前的时间。

注意，由于是根据系统设置来创建的，所以用户可以通过篡改时区和时间来影响 `new Date()` 的值。如果应用对时间有防篡改要求，应从服务器取基准时间再进行 `new Date()`，可以封装成公共函数调用，避免直接使用 `new Date()`。

### `new Date(value)`

`value` 是一个整形数字，代表的是从1970年1月1日00:00:00（UTC 时间）开始过去的毫秒数。可正可负，如果是负数表示的是早于1970年1月1日00:00:00的时间。

```js
> new Date(86400000 * 365)
< Fri Jan 01 1971 08:00:00 GMT+0800 (CST)

> new Date(86400000 * 365)
< Wed Jan 01 1969 08:00:00 GMT+0800 (CST)
```

Javascript 的 `Date` 上下限为距离1970年1月1日 `100000000` 天的时间。

```js
> new Date(86400000 * -100000000)
< Tue Apr 20 -271821 08:00:00 GMT+0800 (CST)

> new Date(86400000 * 100000000)
< Sat Sep 13 275760 08:00:00 GMT+0800 (CST)
```

```js
> new Date(86400000 * 100000001)
< Invalid Date

> new Date(86400000 * -100000001)
< Invalid Date
```

### `new Date(dateString)`

`dateString` 方式创建 `Date` 对象是**最容易犯错的**，因为一是浏览器对其支持很不一致，二是大部分人对其格式要求不是很清楚。另可参阅 <http://dygraphs.com/date-formats.html> 看看 Chrome, Firefox, IE 等浏览器的历史版本支持情况。

`dateString` 是代表日期的一个字符串，应当能被 `Date.parse()` 方法识别（[IETF-compilant RFC 2822 时间戳](http://tools.ietf.org/html/rfc2822#page-14)和[ISO 8601 的一个版本](http://www.ecma-international.org/ecma-262/5.1/#sec-15.9.1.15)）。

> 注意：通过 `dateString` 来构造 `Date` 对象（或者通过 `Date.parse`，它们是相同的）是**强烈不推荐的**，因为浏览器间的差异性和不稳定性。传统上只支持 RFC 2822 格式。ISO 8601 格式的支持不统一，在字符串只有日期时（例如"1970-01-01"），有的处理成 UTC 时间，不是本地时间。

由于 `dateString` 接受的字符串格式有严格限制，容易碰到兼容性问题，可以用第三方的日期处理库 [Datejs](https://github.com/datejs/Datejs) 或者 [moment.js](https://github.com/moment/moment/) 来处理日期，都有专门的 `parse` 功能，可以接受更宽泛的日期字符串格式。

#### RFC 2822 格式

RFC 2822 格式是以英文为准的，文档也很难懂。为简便起见，把文档中分割不同字段的 FWS(Folding White Space) 和 CFWS(Comment Folding White Space) 简略为一个空格，不在下列示意图中展示，读者可以自行替换。有兴趣的可以参阅[WSP, FWS, CFWS 的解释](https://stackoverflow.com/questions/37170893/what-does-cfws-and-fws-mean-in-this-abnf)。

用一张图表示 RFC 2822 格式如下：

```
                        date-time
                            |
                            |
[ date-of-week "," ]-------date-------------------------------------time 
      |                     |                                         |
      |                     |                                         |
      |                     |                   time-of-day-----------+---------------+
      |                     |                        |                                |
      |                     |                        |                                |
      |                     |                        |                                |
      |                     |                        |                                |
      |                     |                        |                                |
  date-name           day month year      hour ":" minute [ ":" second ]            zone
```

其中的中括号表示可选的，叶子节点的值（简便起见，文档中的 `obs-*`【即obsolete，过时的】值也不做讨论）如下：

- `date-name`: 星期，取值为 `Mon/Tue/Wed/Thu/Fri/Sat/Sun` 之一
- `day`: 1-2位数字
- `month`: 取值为 `Jan/Feb/Mar/Apr/May/Jun/Jul/Aug/Sep/Oct/Nov/Dec` 之一
- `year`: 至多4位数字
- `hour`: 2位数字
- `minute`: 2位数字
- `second`: 2位数字
- `zone`: `"+"` 或者 `"-"`,接4位数字，例如 "+0000", "-8000"

比如，以下格式都是正确的 RFC 2822 格式：

```js
new Date('Mon, 31 Jul 2017 00:00:00 +0800')
new Date('Mon 31 Jul 2017 00:00:00 +0800')
new Date('31 Jul 2017 00:00:00 +0800')
new Date('31 Jul 2017 00:00 +0800')
```

由于这种格式还有兼容 `obs-*` 的值，比如 `zone`，可用字母缩写 `UT`, `GMT`, `EST`, `EDT` 等表示，总体支持格式感觉比较乱，不易掌握。

#### ISO 8601 格式

ISO 8601 格式的日期字符串为 `YYYY-MM-DDTHH:mm:ss.sssZ`，各部分解释如下：

- `YYYY`: 格里高利历的 0000-9999 年
- `-`: 字面量的连接符，出现两次
- `MM`: 月份，01（1月）-12（12月）
- `DD`: 日期，01-31
- `T`: 字面量的字符`T`，指示时刻片段的开始
- `HH`: 小时，00-24
- `:`: 字面量的冒号，出现两次
- `mm`: 分钟，00-59
- `ss`: 秒，00-59
- `.`: 字面量的点
- `sss`: 毫秒，0-999，可以1到3个字符。
- `Z`: 时区，"**Z**" 代表 UTC，或者 `+HH:mm`, `-HH:mm` 代表所在时区。比如东八区 `+08:00`

这个格式允许以下仅包含日期（date-only）的形式：

```
    YYYY
    YYYY-MM
    YYYY-MM-DD
```

也允许 date-time 形式，通过上面的 date-only 形式之一接上下面的时间形式（可选的接上后面的时区格式）：

```
    THH:mm
    THH:mm:ss
    THH:mm:ss.sss
```

所有的数字都是十进制的。`YY` 和 `MM` 的缺省值是 `01`；`HH`, `mm`, `ss` 的缺省值是 `00`；`sss` 的缺省值是 `000`；时区的缺省值是 `Z`。

时区缺省值实际上是有所不同的，例如在 Chrome 和 Safari 下缺省值不一样：

```js
// Chrome，缺省值当地时区
> new Date('1970-01-01T00:00:00')
< Thu Jan 01 1970 00:00:00 GMT+0800 (CST)

// Safari，缺省值 UTC
> new Date('1970-01-01T00:00:00')
< Thu Jan 01 1970 08:00:00 GMT+0800 (CST)
```

非法值（超过界限或者语法错误）都意味着该字符串不是 ISO 8601 格式的。

> 注意1：由于一天在子夜开始和结束，00:00 和 24:00 能用于区分一天开始的子夜和结束的子夜。这意味着下面两个标记指向的是同一个时刻：`1995-02-04T24:00` 和 `1995-02-05T00:00`。
> 
> 注意2：有一些非国际标准用城市缩写指定时区，像 CET, EST 等。有时候相同的缩写用在了完全不同的时区。由于这个原因，ISO 8601 指定用数字表示日期和时间。

### `new Date(year, month[, date[, hours[, minutes[, seconds[, milliseconds]]]]])`

- `year`: 代表年份的整形数字，0-99的值映射到1900-1999。
- `month`: 代表月份的整形数字，0-11分别对应1-12月。
- `date`: 可选的，整形数字，代表所在月份的日期。
- `hours`: 可选的，整形数字，代表一天内的小时。
- `minutes`: 可选的，整形数字，代表分钟。
- `seconds`: 可选的，整形数字，代表秒。
- `miliseconds`: 可选的，整形数字，代表毫秒。

`date` 参数缺省值为1，其他参数缺省值为0。

#### 参数值的范围

> 注意：`Date` 作为构造函数调用时且不止一个参数，如果参数值比对应的逻辑值大（例如 month 值为13，minute 值为70），那么相邻的值就会被调整。例如 `new Date(2013, 13, 1)` 等同于 `new Date(2014, 1, 1)`，都会创建一个日期 `2014-02-01`（month 是从0开始的）。对于其他值也类似，`new Date(2013, 2, 1, 0, 70)` 等同于 `new Date(2013, 2, 1, 1, 10)`，都会创建一个日期 `2013-03-01T01:10:00`。

我们可以注意到后面这个用例日期字符串是`2013-03-01T01:10:00`来表示的，即前面提到的 ISO 8601 格式。

#### 成功创建的 `Date` 对象时区问题

> 注意：当 `Date` 作为构造函数调用时且不止一个参数，参数值指定的是当地时区的值。如果想要 UTC 时区，可以使用 `new Date(Date.UTC(...))`，使用相同的参数。

## 创建方式总结

前面介绍了四种 `new Date` 的形式，优缺点如下：

- `new Date()`: 无兼容性问题，但时间是根据系统设置创建的
- `new Date(value)`: 无兼容性问题
- `new Date(dateString)`: 格式复杂，容易产生兼容性问题
- `new Date(year, month[, date[, hours[, minutes[, seconds[, milliseconds]]]]])`: 无兼容性问题

所以，最好优先使用一、三、四种方式创建日期，如果你的数据是一个日期字符串格式，你又不确定格式是否合法，可以把对应的参数值截取出来，再采用第四种方式。例如后台返回数据格式为 `2017/07/31`：

```js
const dateString = '2017/07/31'
const splitted = dateString.split('/')
const date = new Date(splitted[0], splitted[1], splitted[2])
```

或者直接集成使用第三方库 Datejs 或 moment.js。

## 日期的格式化

如果成功创建了 `Date` 对象，在前端使用时格式化也会有时区的问题。例如，后台返回来的时间为 `2017-07-31`（服务器所在东八区+08:00），但是用户系统所在时区为东五区+05:00，那么 `new Date('2017-07-31')` 将创建的日期会变成 `2017-07-30T21:00:00+05:00`。这样，如果你的代码用 `Date.getDate()` 方法就会出现日期错误的情况。

如果都以服务器所在时区为准，可以在 `new Date` 之后再加上时区的偏差，变成所在时区的值与服务器所在时区值一致：

```js
// 以服务器在东八区（getTimezoneOffset() = -480）为例
const date = new Date('2017-07-31')
date.setMinutes(date.getMinutes() + date.getTimezoneOffset() + 480)
```

### `Date.toJSON()`

注意，该方法返回的是 UTC 时区的时间，如果要获取当地时区的 date 请使用 `Date.getDate()`。

```js
> new Date('2017-07-31T00:00:00+08:00').toJSON()
< "2017-07-30T16:00:00.000Z"
```
