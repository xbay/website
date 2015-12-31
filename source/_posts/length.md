date: 2015-12-31 16:43:50
author: [hiddenlotus]
title: length的几种写法
tags: [javascript, 趣谈]
----

一个传统程序员是这么写的：
```
function length(array) {
  var sum = 0
  for(var i = 0; i < array.length; i++) {
    sum++
  }
  return sum
}
```

一个读过SICP的程序员是这么写的：
```
function length(array) {
  if(array.length === 0) {
    return 0
  } else {
    return 1 + length(array.slice(1))
  }
}
```

另一个读过SICP的程序员：
```
function length(array) {
  function helper(array, num) {
    if(array.length === 0) {
      return num
    } else {
      return helper(array.slice(1), num + 1)
    }
  }
  return helper(array, 0)
}
```

再函数式一点：
```
function length(array) {
  return array.reduce(function(prev, cur) { return prev + 1 }, 0)
}
```

一个我面试过的人...
```
function length(array) {
  return array.length
}
```
