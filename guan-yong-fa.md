# 惯用法

### 类型转换

##### bool

`!!`  可以将各种类型转换为 bool, 常见的几种结果如下:

```js
!!undefined === false
!!null === false
!!NaN === false
!!0  === false
!!(-0) === false
!!'' === false
```

但是注意几个**陷阱**:

```js
!![] === true
!!{} === true    // 可以理解为: 任何对象都是真值, 即便对象内容是空
!!new Boolean(false) === true
```

##### string

`''`  以及 `[]`  与任何值相加, 结果都是字符串

```js
'' + 123 === '123'
'' + (-123) === '-123'
'' + function() {} === 'function() {}'
'' + undefined === 'undefined'
'' + NaN === 'NaN'

// 同理, 可以用 [] 代替上述 '', 如:
[] + 123 === '123'
```

但是, 有一个**例外**:

```js
'' + [] === ''        // 注意, 这里并不是 '[]'
```

##### number

```js
+'123' === 123
-'123' === -123
-'0' === -0
```

##### 数字取整

```js
~~(123.45) === 123
~~(-123.45) === -123
```

更多关于 &lt;类型转换&gt; 的内容, 可以参考这里休闲一下: [https://github.com/walfud/zhuangbility](https://github.com/walfud/zhuangbility)



### 短路操作符

##### &&

一般用于判断前置条件

```js
condition && func()    // condition 为真才执行 func, 否则返回 condition 的值
```

##### \|\|

一般用于数据兜底

```js
mayEmptyValue || 123    // mayEmptyValue 为真则将其返回, 否则返回 123
```



