你真的知道怎么用javascript来写一个倒计时吗 ?
===========================================================================

倒计时功能真的是前端开发中最日常遇到的一个特效效果了，但是你真的知道怎么写吗？就在上周，在团队中试水了一下，结果却真的非常有意思，特此总结为一篇博文分享出来。
首先我们明确需求，我给出的题目是，用js写一个倒计时，用你能想到的各种方式，不局限于性能，效率，展示形式，扩展性，API设计等。
最后我收上来了10几份作业，通过review代码，简单的提炼一下，大家可以看看自己处在哪一个级别。：）

基本实现
-------------------
基本的一个倒计时的原理非常简单了，使用setTimout或者setInterval来对一个函数进行递归或者重复调用，然后对DOM节点做对应的render处理，并对时间做倒计时的格式化处理。

例如下面这个样子：

```js
//使用setInterval实现一个5的倒计时。
(function() {
    var time = 5;
    var p = document.getElementsById("time");
    var set = setInterval(function() {
        time--;
        p.innerHTML = time;
        if(time === 0) {
            p.innerHTML = "";
            clearInterval(set);
        }
    }, 1000);
})()
```

我对上面的写法定义为仅仅考虑了实现功能，而写法是完全面向过程的，操作都放到了匿名函数中处理，而且也没有对时间做格式化处理，但是原理也很简单，使用setInterval不断的改变DOM节点，然后对time变量做--操作。

稍微好一点的写法：

```js
window.onload = function(){
    showtime();
    function addZero(i){
        if(i<10){
            i = "0" + i;
        }return i;
    }
    function showtime() {
        var nowtime = new Date();
        var endtime = new Date("2016/05/20,20:20:20");
        var lefttime = parseInt((endtime.getTime() - nowtime.getTime()) / 1000);
        var d = parseInt(lefttime / (24 * 60 * 60));
        var h = parseInt(lefttime / (60 * 60) % 24);
        var m = parseInt(lefttime / 60 % 60);
        var s = parseInt(lefttime % 60);
        h = addZero(h);
        m = addZero(m);
        s = addZero(s);
        document.getElementById("contdown").innerHTML = "倒计时    " + d + ":" + h + ":" + m + ":" + s;
        if(lefttime<=0){
            document.getElementById("contdown").innerHTML = "活动已结束";
            return;
        } 
        setTimeout(showtime,1000);
    }
}
```

这种实现方法比上面的写法好一些了，但也是面向过程的，无法扩展，实现上首先，定义了一个showtime方法，然后作为入口函数执行，有一个辅助方法addZero，作用是对数字补位比如9补成09。然后showtime函数中，对时间做了一个差值计算，然后换算了对应的倒计时天，时，分，秒。最后又对DOM做了渲染，这次不一样的地方是使用了setTimeout来进行递归渲染，没有使用setInterval。

那么问题出现了，这里涉及到几个点：

* 补0操作的实现方法，怎么写比较好一点？
* 为什么用setTimeout，又为什么要使用setInterval，到底选哪个好一点？

leftPad操作的扩展
-------------------

```js
//几个同学几组不同的补零方法实现：
function leftPad(i){
    if(i<10){
        i = "0" + i;
    }
    return i;
}
function leftPad(i){
  return i < 10 ? '0'+i : i+'';
}
function leftPad(n){
    var n = parseInt(n, 10);
    return n > 0 ? n <= 9 ? ('0'+n) : (n+'') :'00';
}
function leftPad(n, len){
    len = len || 2;
    n = n + '';
    var diff = len - n.length;
    if (diff > 0) {
        n = new Array(diff + 1).join('0') + n;
    }
    return n;
}
```

最后来一个之前一阵子曝光度非常高的npm的leftpad模块的实现。

```js
function leftpad (str, len, ch) {
  str = String(str);
  var i = -1;
  if (!ch && ch !== 0) ch = ' ';
  len = len - str.length;
  while (++i < len) {
    str = ch + str;
  }
  return str;
}
```

简单分析一下，我们自己写的和老外们写的有何区别：

* 第一种写法返回值有bug，小余10的返回的为number类型，一个js的坑。
* 第二种写法弥补了这个bug，知道转换类型，但是只考虑了一个参数，功能仅仅为补0。
* 第三种写法考虑了负数的情况，会自动转换成00。
* 第四种方法终于考虑到了，可能会补更多的0，用了一个创建指定长度数组的方法`[,,].join("0")`来避免了一次循环的使用。
* 第五种，npm中的leftpad模块，考虑了第三种情况，那就是可能想补的不是0，而是空格啊，或者一些其他的什么别的东西，例如`leftPad("foo",5); //"  foo"; leftPad("foo",5,0); //"00foo"` 当然他默认补的就是空格，补0要自己填第三个参数。

说到这里可见一斑，同样一个函数，不同的程序员考虑问题的方式方法和实现手段，真的是太奇妙了，当然这和经验，水平还有脑洞都有关系。

setTimeout和setInterval如何选择？
-------------------

这个问题分2个点来说明：

* 这2个函数的功能区别
* javascript单线程的理解



