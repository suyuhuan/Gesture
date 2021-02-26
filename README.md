# 手势库说明

## 手势理论说明

### touchEvent VS mouseEvent

- touchEvent 没有像 mouseevent 在事件上的 x/y，而是在 touchlist(可能有多个 touch)，其中的每个 touch object 上
- touchevent 和 mouseevent 有不同的抽象，要对他们合理的统一抽象，才能使得组件同时支持 touch 和 mouse，不至于产生 touch 的场景下同时触发了 mouse 事件等的 bug
- 给组件写手势库来区分 touch 和 mouse
- 手机的系统手势可以关闭的，保护组件的手势不被识别成系统手势而造成退出 app
- 多指手势会产生 transform，rotate，scale
- 比较有名的手势库 hammerJS

### 手势行为

- Tap(手往屏幕上面点)
- Pan(手指拖拽)
- Flick(快速点)
- Press(点后较长时间离开屏幕)

### 行为说明

- start 之后很快的 end：tap 事件
- start 之后过了几秒：pressstart - 移动 10px（一般业界用的数字，但还是需要用 dpr 去算一下） ： panstart
- start 之后过了几秒：pressstart - end ： pressend
- start 之后移动了如 10px：panstart
- panstart - move :panmove - move ：panmove -end ：panend
- panstart - move :panmove - move ：panmove -end 且速度>XX ：panend + flick
- flick 是 panend 的一个变形，有可能是和 panend 一起同时触发
- 处理不同的 start 和 move 之间的关系-context：为了算出移动距离起点的距离，需要在 start 时候存储起点，因为 start 要处理 touch 和 mouse 两类事件，且可能被触发多次 start，所以不能仅从传入的 point 去存储起点，而应该同时传入 point 来自的 context，在 start 事件被触发时，把它的 context 传入全局的 contexts 对象中
- 在 context 中设置事件标志存储 isTap/isPan/isPress
- 考虑双击事件的话需要延迟 tap 才知道是不是双击，因为第一次触发的时候一定会识别成 tap。tap/singletap/doubletap 来识别双击
- Listener 还可以加入考虑 gesture 库的用户自定义的逻辑，如是否 capture 或者是否 passive 等
- flick 事件跟速度有关，有两种可能：快速的扫动，或者 press 时候，在 end 的时候已很快的速度扫出，看自己要不要处理后者。现在在 pan 之后去处理 flick 逻辑，看 end 之前 300ms 内的速度有多快，大于 2.5 就认为是触发了 flick 事件，不然就是 pan/press。
- 2.5 这个速度可根据具体情况/用研数据去做调整

### 事先声明

```
let element = document.body;
let contexts = Object.create(null);
let MOUSE_SYMBOL = Symbol('mouse');
```

### 鼠标事件监听模型

```
element.addEventListener("mousedown", (event)=>{
        contexts[MOUSE_SYMBOL] = Object.create(null);
        start(event, contexts[MOUSE_SYMBOL]);
        let mousemove = event => {
           move(event, contexts[MOUSE_SYMBOL]);
        }
        let mouseend = event => {
            end(event, contexts[MOUSE_SYMBOL]);
            document.removeEventListener("mousemove", mousemove);
            document.removeEventListener("mouseup", mouseend);
        }
        document.addEventListener("mousemove", mousemove);
        document.addEventListener("mouseup", mouseend);
    })
```

### touch 事件监听模型

```
// 触摸屏
element.addEventListener("touchstart", event => {
   for (let touch of event.changedTouches) {
    contexts[touch.identifier] = Object.create(null);
     start(touch, contexts[touch.identifier])
   }
})

element.addEventListener("touchmove", event => {
    for (let touch of event.changedTouches) {
        move(touch, contexts[touch.identifier])
    }
})

element.addEventListener("touchend", event => {
    for (let touch of event.changedTouches) {
        end(touch, contexts[touch.identifier]);
        delete contexts[touch.identifier];
    }
})

element.addEventListener("touchcancel", event => {
    for (let touch of event.changedTouches) {
        cancel(touch, contexts[touch.identifier]);
        delete contexts[touch.identifier];
    }
})
```

### 行为抽象

```
let start = (point,context) => {
    context.startX = point.clientX, context.startY = point.clientY;
    context.isTap = true;
    context.isPan = false;
    context.isPress = false;
    console.log("Tap");
    context.timeputHandler = setTimeout(()=>{
        if (context.isPan)
            return;
        context.isTap = false;
        context.isPan = false;
        context.isPress = true;
        console.log("pressstart");
    },500)

    // console.log("start",point.clientX,point.clientY);
}

let move = (point,context) => {
    let dx = point.clientX - context.startX, dy = point.clientY - context.startY;
    if (dx ** 2 + dy **2 > 100 && !context.isPan) {
        context.isTap = false;
        context.isPan = true;
        context.isPress = false;
        console.log("panstart");
    }
    if (context.isPan) {
        console.log("pan");
    }
    // console.log("move",dx, dy);
}

let end = (point,context) => {
    // console.log("end",point.clientX,point.clientY);
    if (context.isPan)
        console.log("panEnd")
    if (context.isTap)
         console.log("Tapend")
    if (context.isPress)
        console.log("PressEnd")
    clearTimeout(context.timeputHandler)
}

let cancel = (point,context) => {
    console.log("cancelend")
    clearTimeout(context.timeputHandler)
}
```
