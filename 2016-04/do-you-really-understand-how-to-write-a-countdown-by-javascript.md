你真的知道怎么用javascript来写一个倒计时吗 ?
===========================================================================

倒计时功能真的是前端开发中最日常遇到的一个特效了，但是你真的知道怎么写吗？就在上周，在团队中试水了一下，结果却真的非常有意思，特此总结为一篇博文分享出来。

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

setTimeout是延迟一定毫秒后执行，setInterval是每隔一定毫秒后执行。但是真相并不像这两句话一样简单。比如我们这个倒计时例子里该选哪一个作为定时调用呢？

首先我们都知道javascript是单线程执行的，所以以上这2个方法都是会被线程阻塞的。

比如setInterval延迟设置为1000，如果内部的执行是一个耗时超过1秒的操作，那么每次重复执行的时候会造成2个问题：

1，是执行被阻塞，预期是1000毫秒执行一次，实际上必须在阻塞结束后才执行。
2，阻塞时，setInterval回调会被堆积，当阻塞结束后，堆积只会被消费一个，那么之后的堆积等于浪费了性能和内存。

如何解决堆积问题呢？因为阻塞是不可能被解决的，那么最简单的方法就是把setInterval换成setTimeout，使用一个递归来造成每隔多久执行一次的功能。当然阻塞是无法被解决的，这里的阻塞不仅仅有回调中的，还有浏览器中的方方面面的阻塞，比如用户的一些操作行为，其他定时器等外部的阻塞，所以这也就是无论我们如何做，页面开久了，定时器都会不准，或者说，变慢的根本原因。

理解了以上，我们就知道该如何选择了。那么我们如何来编写一个高性能可扩展的倒计时呢？

我们下面就针对以上的分析继续review代码。

到底如何编写面向对象的javascript
-------------------

以下的几个进化过程，组内实现都一一包含了，希望通过分析可以提高一些你对设计模式在js中的灵活理解：

```js
var CountDown = {
	$ : function(id){/*id选择器*/},
	init :function(startTime,endTime,el){/*执行定时器入口，使用setTimeout调用_timer*/},
	_timer : function(startTime,endTime,el){/*私有方法，处理时间参数等具体业务*/}
}
CountDown.init("","2016,04,23 9:34:44","countdown1");
```

点评：单例模式，简单方便好理解，缺点是每次init都会拿一个新定时器，性能不好。继承和扩展能力一般，无法获取实例属性，导致了执行状态都是不可见的。

```js
function Countdown(elem, startTime, endTime) {
	this.elem = elem;
	this.startTime = (new Date(startTime).getTime()) ? (new Date(startTime).getTime()) : (new Date().getTime());
	this.endTime = new Date(endTime).getTime();
}
Countdown.prototype = {
	SetTime: function() {},
	leftPad: function(n) {},
	DownTime: function() {}
}
var test = new Countdown("time", "2016/1/30,12:20:12", "2017/1/30,12:20:12");
test.SetTime();
```

点评：标准的原型构造器写法，简单方便好理解，确定是每次都拿一个新定时器，实例增多后性能同样不好，按道理setTime，leftPad等方法都可以通过继承来实现，方便扩展和复用，prototype上的方法均为辅助方法，按理不应该被外部调用，这里应该封装为私有方法或者前缀+_，优点可以通过实例拿到相关倒计时属性，可以对实例再做扩展操作。

```js
var countdown = {};
countdown.leftPad = function(n, len) {};
countdown.timeToSecond = function(t) {};
/**
 * 倒计时工厂
 * @param  {[object]} obj 倒计时配置信息
 * @return {[object]}     返回一个倒计时对象
 */
countdown.create = function(obj) {
    var o = {};
    o.dom = document.getElementById(obj.id);
    o.startMS = +new Date(obj.startTime || 0);
    o.endMS = +new Date(obj.endTime || 0);
    obj.totalTime && (o.totalTime = countdown.timeToSecond(obj.totalTime));

    var newCountdown = new countdown.style[obj.style](o);

    newCountdown.go = function(callback) {
        callback && (newCountdown.callback = callback);
        newCountdown.render();
        clearInterval(newCountdown.timer);
        newCountdown.timer = setInterval(newCountdown.render, 1000);
    };
    return newCountdown;
};
countdown.style.style1 = function(obj) {
    this.dom = obj.dom;
    this.startMS = obj.startMS;
    this.endMS = obj.endMS;
    var _this = this;
    this.render = function() {
        var currentMS = +new Date();
        var diff = (_this.endMS - currentMS) / 1000;
        var d = parseInt(diff / 60 / 60 / 24);
        d = countdown.leftPad(d, 3);
        d = d.replace(/(\d)/g, '<span>$1</span>');
        _this.dom.innerHTML = '距离国庆节还有：' + d + '天';
        if (currentMS > _this.endMS) {
            clearInterval(_this.timer);
            if (_this.callback) {
                _this.callback();
            } else {
                _this.dom.innerHTML = '国庆节倒计时结束';
            }
        }
    };
};
countdown.style.style2 = function(obj) {};
countdown.style.style3 = function(obj) {};
countdown.create({id:"clock3",totalTime:'82:23',style:'style1'}).go(function(){alert('It is over');});
```

点评：我尽量的减少了无用的干扰，留下最关键的部分。

优点：这里的countdown是一个比较简单的工厂模式实现，实现了一个统一的create方法，create方法上调用了style这个属性上扩展的样式（style1-3）实现，create方法返回的是一个独立的新实例，并统一扩展了go方法，go方法里统一创建定时器并挂载到timer属性，在这里我们也就等同拥有了修改和控制每个工厂造出来的单例的能力，样式做到了可扩展，leftPad，timeToSecond也可以方便通过一个utils对象来进行继承。

缺点：没有考虑到上面提到的setTimeout和setInterval的区别，也没有时间校验机制，在性能方面考虑不多。

```js
var EventNotifys = [];
var Event = {
    notify:function(eventName, data){},
    subscribe: function (eventName, callback) {},
    unsubscribe: function (eventName, callback) {}
};
var timer = null;

$.countDown = function(deadline,domParam){
    var that = this,
        MILLS_OFFSET = 15;
    function CountDown(){
        this.deadline = deadline;
        this.domParam = domParam;
    };
    CountDown.prototype = {
        leftPad: function(n){},
        /**
        * 计算时差
        * @returns {{sec: string, mini: string, hour: string, day: string, month: string, year: string}}
        */
        caculate: function(){},
        /*刷新dom*/
        refresh: function(){}
        };
        var countDown = new CountDown();
        /**
         * 启动定时器
         * @param first 是否首次进入
         */
        function startTimer(first){
            !first&&Event.notify('TIMER');
            //若是首次进入，则根据当前时间的毫秒数进行纠偏，延迟1000-当前毫秒数达到整数秒后开始更新UI
            //否则直接1秒后更新UI
            //若当前毫秒数大于MILLS_OFFSET 15，则修正延时数值与系统时间同步
            mills = new Date().getMilliseconds();
            timer = setTimeout(arguments.callee,first?(1000 -mills):(mills>MILLS_OFFSET?(1000-mills):1000));
            console.log(new Date().getMilliseconds());
        }
        /**
         * 订阅一次事件
         */
        Event.subscribe('TIMER',countDown.refresh.bind(countDown));
        //首次初始化时启动定时器
        !timer && startTimer(true);
    };
    
/*dom结构和样式与js分离，这里指定倒计时的dom节点信息作为配置*/
$.countDown('20160517 220451',{
    sec: $("#seconds6"),
    mini: $("#minute6"),
    hour: $("#hour6"),
    day: $("#day6"),
    month: $("#month6"),
    year: $("#year6")
});
```

点评：这里也是因为篇幅问题，去掉了多余代码，只留下了核心思路实现。首先是实现了一个jquery的countDown的扩展方法，传入倒计时时间和需要操作的dom节点信息配置，然后实现了一个Event对象，俗称sub/pub观察者的实现，可以广播和订阅还有取消订阅，所有订阅回调存在EventNotifys数组中，之后每个$.countDown方法返回的都是一个标准的单例实例，原型上有一系列的辅助方法，这里的caculate返回的是格式化的时间方便渲染时分离操作，但是这个实例和上面的实例不一样的地方是不包含定时器部分，定时器部分的实现使用了一个timer来实现，然后通过广播的形式，让所有产生的实例共享一个定时器。这里就考虑到性能，页面无论多少个倒计时，都会保持最少的定时器实例。最后在每次订阅的回调中，刷新所有的实例dom，消耗也是最小，并且在每次递归后对阻塞进行了计算，如果误差超过一定阀值，则进行矫正。但是这里的矫正只是对阻塞时间差进行了矫正，并没有和系统时间矫正。

优点在上面的实现思路上已经大概说明白了，应该已经可以算是满分了。除了没有和系统时间做矫正，只是对阻塞做了矫正。

最后还有一个实现，思路上并没有上面这个一样做到定时器复用，但是优点是文档齐全，前后端（nodejs）通用，内部解耦做的也不错，但是性能上也没有考虑太多，还是使用了setInterval做了实现，比较出彩的是鲁棒性比较好，实现上也是先设计API后对库进行实现封装，最后也通过事件广播的方式对每个定时器做了外部事件解耦，具体参见： https://github.com/luoye-fe/countdown

API设计确实仁者见仁，智者见智，但是一般的开发者还是考虑不到的，这里值得表扬，最后三个都属于组内很优秀的例子了。

最后看看我的吧，毕竟题目是我出的。

https://gist.github.com/xiaojue/d42e69117b6d53ace81d56e454ce5141 (他们说gist需要翻墙才能看，那我还是以防万一，贴一下。。)

```js
/**
 * @author xiaojue
 * @date 20160420
 * @fileoverview 倒计时想太多版
 */
(function() {

  function timer(delay) {
    this._queue = [];
    this.stop = false;
    this._createTimer(delay);
  }

  timer.prototype = {
    constructor: timer,
    _createTimer: function(delay) {
      var self = this;
      var first = true;
      (function() {
        var s = new Date();
        for (var i = 0; i < self._queue.length; i++) {
          self._queue[i]();
        }
        if (!self.stop) {
          var cost = new Date() - s;
          delay = first ? delay : ((cost > delay) ? cost - delay : delay);
          setTimeout(arguments.callee, delay);
        }
      })();
      first = false;
    },
    add: function(cb) {
      this._queue.push(cb);
      this.stop = false;
      return this._queue.length - 1;
    },
    remove: function(index) {
      this._queue.splice(index, 1);
      if(!this._queue.length){
        this.stop = true;
      }
    }
  };

  function TimePool(){
    this._pool = {}; 
  }

  TimePool.prototype = {
    constructor:TimePool,
    getTimer:function(delayTime){
      var t = this._pool[delayTime];
      return t ? t : (this._pool[delayTime] = new timer(delayTime));
    },
    removeTimer:function(delayTime){
      if(this._pool[delayTime]){
        delete this._pool[delayTime];
      }
    }
  };

  var delayTime = 1000;
  var msInterval = new TimePool().getTimer(delayTime);

  function countDown(config) {
    var defaultOptions = {
      fixNow: 3 * 1000,
      fixNowDate: false,
      now: new Date().valueOf(),
      template: '{d}:{h}:{m}:{s}',
      render: function(outstring) {
        console.log(outstring);
      },
      end: function() {
        console.log('the end!');
      },
      endTime: new Date().valueOf() + 5 * 1000 * 60
    };
    for (var i in defaultOptions) {
      if (defaultOptions.hasOwnProperty(i)) {
        this[i] = config[i] || defaultOptions[i];
      }
    }
    this.init();
  }

  countDown.prototype = {
    constructor: countDown,
    init: function() {
      var self = this;
      if (this.fixNowDate) {
        var fix = new timer(this.fixNow);
        fix.add(function() {
          self.getNowTime(function(now) {
            self.now = now;
          });
        });
      }
      var index = msInterval.add(function() {
        self.now += delayTime;
        if (self.now >= self.endTime) {
          msInterval.remove(index);
          self.end();
        } else {
          self.render(self.getOutString());
        }
      });
    },
    getBetween: function() {
      return _formatTime(this.endTime - this.now);
    },
    getOutString: function() {
      var between = this.getBetween();
      return this.template.replace(/{(\w*)}/g, function(m, key) {
        return between.hasOwnProperty(key) ? between[key] : "";
      });
    },
    getNowTime: function(cb) {
      var xhr = new XMLHttpRequest();
      xhr.open('get', '/', true);
      xhr.onreadystatechange = function() {
        if (xhr.readyState === 3) {
          var now = xhr.getResponseHeader('Date');
          cb(new Date(now).valueOf());
          xhr.abort();
        }
      };
      xhr.send(null);
    }
  };

  function _cover(num) {
    var n = parseInt(num, 10);
    return n < 10 ? '0' + n : n;
  }

  function _formatTime(ms) {
    var s = ms / 1000,
      m = s / 60;
    return {
      d: _cover(m / 60 / 24),
      h: _cover(m / 60 % 24),
      m: _cover(m % 60),
      s: _cover(s % 60)
    };
  }

  var now = Date.now();

  new countDown({});
  new countDown({
    endTime: now + 8 * 1000
  });

})();
```

不要脸的自己点评一下自己：主要说思路，实现分成几个步骤，第一个类是timer，通过setTimeout创建一个倒计时实例，内部对阻塞误差做了矫正，但是缺少一步，如果阻塞大于延迟时间，那么应该下次递归直接执行，而我这里还是会延迟1s，只是相对会减少误差。然后这个timer通过add和remove方法管理一个回调队列，让所有通过这个timer实现的倒计时都共用一个定时器。第二个类TimePool，主要实现一个timer池子，我们之前只想到了倒计时都是1s的，那如果有500ms或者10s的倒计时，这里这个池子是可以装多个timer类的，保证不同延迟下，定时器最小。最后一个类就是countDown类，里面同样是默认配置起手，然后主要介绍`fixNowDate`这个参数，是用来进行系统时间修正的，这里默认的实现是浏览器端的一个修正技巧，主要参见`getNowTime`方法，通过一个xhr请求拿服务端headers中的Date来进行矫正。

之后摘出来了一个`template`，对输出的时间格式可以做一个简单的模板替换，抽象了3个公共类，最后的countDonw依赖其他2个类做了最终实现。最后，这段代码也可以复用在nodejs端，只需要修改`getNowTime`的方法为nodejs的即可。

我的这段实现，没有考虑太多样式的可扩展，更多是性能上的，但是真实情况下性能如何，还需要实际验证，这一步是没有做的，而且只适用页面多定时器和倒计时的场景，可能还存在一些隐藏的未知bug。所以在这里只是开了一下脑洞，做了一个初步的实现，拿走需谨慎。

最后，我不爱写注释，这里要严肃的批评一下我自己……

总结
-------------------

通过一个最简单的倒计时功能，review过团队所有的实现之后，个人感觉收获最多的是，每个工程师的思维和经验确实决定了编码的高度，但是这种思维蛮好培养的。总结一下的话，主要表现在程序的扩展性，性能，鲁棒性，复用性，维护性这几方面。所以以后写代码时只要想的足够多，足够细致，我相信所有人都可以写出非常优秀的好代码的。

看完这篇文章后，你还觉得你真的知道怎么写一个javascript倒计时了吗？希望对所有阅读的人都能带来收获。




