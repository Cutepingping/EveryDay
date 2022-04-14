## reduce函数的特点
它接收一个函数，函数有4个参数，第1个是上次累计计算结果，第2个是遍历到的当前值，第3个是当前值的下标，第4个是调用它的数组。
它对数组中的从左到右的每一个数使用函数作累计操作，前一个函数累加结果作为后一个函数的第一个参数继续做累加，直到数组结束为止，它分两种情况，有初始值和没初始值两种：
- 当没有初始值时，数组中的第一个数会默认为第一个初始值pre，然后cur从第二个数开始，然后逐渐向后调用回调函数计算累计结果
- 当有初始值时，数组中第一个数pre会为初始值，cur从第一个数开始，然后逐渐向后调用回调函数计算累计结果

### 使用方式
```javascript
// 没设初始值，数组中第一个数会作为初始值，然后对剩下的数组做reduce操作，总共执行arr.length - 1次操作
res = [1, 2, 3, 4].reduce((pre, cur, curIndex, arr) => {
  console.log(pre, cur, curIndex, arr)
  // 第一次，pre为1，cur为2，curIndex为2的下标1
  // 1 2 1 [ 1, 2, 3, 4 ]
  // 第二次，pre为前两次累加结果3, cur为第三个数3，curIndex为3的下标2
  // 3 3 2 [ 1, 2, 3, 4 ]
  // 第三次，pre再次为前两数的累加结果6， cur为第四个数4，curIndex为4的下标3
  // 6 4 3 [ 1, 2, 3, 4 ]
  // 最后，输出最后一次pre和cur的和, 6 + 4 = 10
  return pre + cur
})

console.log(res)  // 10

```
```javascript
// 设有初始值，除初始值，要对整个数组做reduce操作，总共执行arr.length次操作
res = [1, 2, 3, 4].reduce((pre, cur, curIndex, arr) => {
  console.log(pre, cur, curIndex, arr)
  // 第1次，pre为初始值0，cur为第一个数1，curIndex为1的下标0
  // 0 1 0 [ 1, 2, 3, 4 ]
  // 第2次，pre为前两次累加结果1，cur为2，curIndex为2的下标1
  // 1 2 1 [ 1, 2, 3, 4 ]
  // 第3次，pre为前两次累加结果3, cur为第三个数3，curIndex为3的下标2
  // 3 3 2 [ 1, 2, 3, 4 ]
  // 第4次，pre再次为前两数的累加结果6， cur为第四个数4，curIndex为4的下标3
  // 6 4 3 [ 1, 2, 3, 4 ]
  // 最后，输出最后一次pre和cur的和, 6 + 4 = 10
  return pre + cur
}, 0)

console.log(res)  // 10
```

### 手写reduce
```javascript
// 注意，不要使用箭头函数，否则this会拿不到数组对象
Array.prototype.myReduce = function(fn, initData) {
  // 因为是数组调用reduce，所以this指向数组
  if(!this.length) return;
  
  let hasInit = initData !== undefined
  let total = hasInit ? initData : this[0]  // 初始值，如果函数有设置则直接使用；没有设置则使用数组第1位作为初始值
  // 遍历数组，如果有初始值，从数组第1位开始计算；如果没有，从数组第2位开始计算
  for(let i = hasInit ? 0 : 1; i < this.length; i++) {
    total = fn(total, this[i], i, this) 
  }
  return total
}

let noInitRes = [1, 2, 3, 4].myReduce((a, b) => a + b)
let hasInitRes = [1, 2, 3, 4].myReduce((a, b) => a + b, 0)
console.log("noInitRes:", noInitRes)  // 10
console.log("hasInitRes:", hasInitRes)  // 10
```
## Compose的实现
### 题目描述
实现一个compose函数，它接受任意个函数，这些函数只接受一个参数，然后compose返回也是一个函数，达到以下效果：
```javascript
const add1 = (x) => x + 1
const mul2 = (x) => x * 2
const div2 = (x) => x / 2
div2(mul2(add1(1))) // 2

// 实现一个compose函数，实现以下功能
let operate = compose(div2, mul2, add1)
operate(1)  // 2
```
### 解决方法
解法：柯里化+reduce
在了解了reduce的原理后，这道题就不难解出来了
```javascript
// 解题思路，柯里化+reduce
function compose(...funcs) {
  return function(...args) {
    // 如果没有传入函数，直接返回参数
    if(funcs.length == 0) return args;
    if(funcs.length == 1) return funcs[0](...args) 

    // 因为是要实现从右到左的计算，所以把数组逆序处理
    funcs = funcs.reverse()

    // 如果是多个，就是上个函数执行的输出结果是下个函数的输入
    return funcs.reduce((a, b, index) => {
      // console.log("index:", index)
      // 因为没有设初始值，所以方法1作为第一个a，数组从第2个方法下标1开始遍历，所以第一次结果应该返回a和b的计算结果，也就是 b(a(...args))
      // 此后，不用再做特殊处理，直接返回b(a)执行结果即可
      if(index === 1) {
        return b(a(...args))
      }
      else return b(a)
    })
  }
}
```
