# mobx使用总结

#### 写在前面的话
- 从7月份进入项目以来，开始接触mobx，希望能做个关于mobx的总结！

## 基本用法
### observable 和 autorun

```
import { observable, autorun } from 'mobx';

const value = observable.box(0);
const number = observable.box(100);
let notObservable = 200

autorun(() => {
  console.log(value.get());
  console.log(notObservable)
});

value.set(1);
value.set(2);
number.set(101);
notObservable = 201
```

控制台依次输出了

```
0
200
1
200
2
200
```
- observable可以用来观测一个数据，这个数据可以数字、字符串、数组、对象等类型，而当观测到的数据发生变化的时候，如果变化的值处在autorun中，那么autorun就会自动执行。
- 上例中的autorun函数中，只对value值进行了操作，而并没有number值的什么事儿，所以number.set(101)这步并不会触发autorun，只有value的变化才触发了autorun。
- 虽然最后notObservable发生了变化，但是这个变量并不是observable 的类型，所以不会触发autorun。  

#### 补充
- 通常支持被observable的类型有三个，分别是Object, Array, Map；
- 对于原始类型，可以使用Obserable.box；
- 当某一类型被observable包装后，返回的其实是observable包装后的类型

### 计算属性---computed

```
import { observable, autorun, computed } from 'mobx';

let number = observable.box(10);
let plus = computed(() => number.get() > 0);

autorun(() => {
  console.log(plus.get());
});

number.set(-19);
number.set(-1);
number.set(1);
```
输出

```
true
false
true
```
- 我们不关注number的值，只关注它是否大于0，这个时候我们就适合用computed属性了。
- 上述结果，第一个true是number初始化的时候，10 > 0 plus返回true，触发autorun。
- 第二个false，number被set为-19，plus < 0 plus返回false，它的值由true变为false，变化了，因此触发autorun，输出false。
- 然后将number set为-1，plus仍然是false，没有变化，不会触发autorun。
- 最后将其设置为1，plus为true，由变化，触发autorun。

### action，runInAction和严格模式（useStrict）

```
import React from 'react';
import ReactDOM from "react-dom";
import { observable, useStrict, action, configure } from 'mobx';
import { observer } from 'mobx-react';

// mobx >= 4 没有 useStrict 了
// 而是使用 `configure({ enforceActions: false })` 关闭 strict mode
// 而在mobx >= 5中，将true，false替换成了以下三种模式
configure({ enforceActions: 'always' })
// false => never  不使用严格模式
// true => observed  
// strict => always  使用严格模式

class MyState {
  @observable num = 0;
  @action addNum = () => {
    this.num++;
  };
}

const newState = new MyState();

@observer
export default class App extends React.Component {

  render() {
    return (
      <div>
        <p>{newState.num}</p>
        <button onClick={newState.addNum}>+1</button>
      </div>
    )
  }
}

var mountNode = document.getElementById("app");
ReactDOM.render(<App />, mountNode);
```

- 以上的例子使用了decorator，比较便捷（项目开发中常见的是这种写法）
- 在函数addNum中，修改了被观测的值，根据规范，我们需要action来修饰这个函数（在严格模式下会报错，非严格模式则不会）
- 建议开启严格模式，这样不至于让你在各个地方很轻易地去改变你所需要的值，降低不确定性。

#### 关于@observer的说明
- 通常，在和Mobx数据有关联的时候，你需要给你的React组件加上@observer；
- 你不必太担心性能上的问题，加上这个@observer不会对性能产生太大的影响；
- 而且@observer还有一个类似于pure render的功能，甚至能起到性能上的一些优化

```
import React from 'react';
import ReactDOM from "react-dom";
import { observable, useStrict, action, configure } from 'mobx';
import { observer } from 'mobx-react';

configure({ enforceActions: 'always' })

class MyState {
  @observable num = 0;
  @action addNum = () => {
    this.num++;
  };
}

const newState = new MyState();

@observer
export default class App extends React.Component {

  render() {
    return (
      <div>
        <p>{newState.num}</p>
        <button onClick={newState.addNum}>+1</button>
        <PureItem />
      </div>
    )
  }
}

@observer
class PureItem extends React.Component {
  render() {
    console.log('PureItem的render触发了');
    return (
      <div>你们的事情跟我没关系</div>
    );
  }
}

var mountNode = document.getElementById("app");
ReactDOM.render(<App />, mountNode);
```
- 在每次点击的时候，只有App的render重新触发，而PureItem只触发了一遍
- 如果去掉子组件PureItem的@observer，按钮每次点击，控制台都会输出 ‘PureItem的render触发了’ 这句话

### mobx的异步处理  
- 在严格模式下
- action只能影响正在运行的函数，而无法影响当前函数调用的异步操作
- 因此对于异步action里的回调，若该回调也要修改observable的值，那么该回调也需要绑定action

```
import React from 'react';
import ReactDOM from "react-dom";
import { observable, useStrict, action, configure, autorun } from 'mobx';
import { observer } from 'mobx-react';

configure({ enforceActions: 'always' })


class MyState {
  @observable num = 0;
  @action
  changeNum() {
    this.num = 100;
    setTimeout(this.anotherChangeNum, 1000)
  }

  // bound 用来解决setTimeout的作用域的问题
  // 这里也需要绑定action，
  // 因为这个回调函数也修改了num（observable）的值  
  @action.bound
  anotherChangeNum() {
    this.num = 300;
  }
}

const newState = new MyState();

autorun(() => {
  console.log(newState.num)
})

newState.changeNum()
```
输出  

```
0
100 
// 一秒后
300
```
还有一种简单的方法：直接包装action

```
import React from 'react';
import ReactDOM from "react-dom";
import { observable, useStrict, action, configure, autorun } from 'mobx';
import { observer } from 'mobx-react';

configure({ enforceActions: 'always' })


class MyState {
  @observable num = 0;
  @action
  changeNum() {
    this.num = 100;
    setTimeout(action('anotherChangeNum', () => {
      this.num = 300;
    }), 1000)
  }
}

const newState = new MyState();

autorun(() => {
  console.log(newState.num)
})

newState.changeNum()
```
- 如果不想到处写action，可以使用mobx提供的工具函数runInAction来简化操作
- 可以把runInAction有点类似action(fn)()的语法糖，调用后，这个action方法会立刻执行

## autorun基本原理  

- autorun是个神奇的函数，被他包装过的方法，就会变为观察者函数；
- 并且autorun有一个很重要的特性，这个函数只会观察自己依赖到的设为 observable 的值
- 它的核心思想是依赖收集  
![](http://pik.internal.baidu.com/2018/11/14/f6a4674704be51e43a7cbf9eaf741870.png)

- Mobx是通过代理变量的getter和setter来实现的变量更新功能；
- 首先先代理变量的getter函数，然后通过预执行一遍autorun中回调，从而触发getter函数，来实现观察值的收集，依次来代理setter。之后只要setter触发便执行收集好的回调就可以了


## mobx的一些坑
### 无法收集新增的属性

```
let ob = observable({ a: 1, b: 1 });
autorun(() => {
  if(ob.c){
    console.log("ob.c:", ob.c);
  }
});
ob.c = 1
```
- 如果本地运行的话，会发现，诶，新增的属性c还是收集到了
- 这是因为 proxy 的 get 会触发
- 而Object.defineProperty不行，是因为这个函数只能设置某个 key 的 getter setter（具体可看源码）

#### 解决方法（无法收集在某些情况下还是会出现的）
- 知道有哪些变量，可以先预定义好
```
let ob = observable({ a: 1, b: 1, c: 0 });
```
- 或者通过extendObservable(target, props)方法来实现
### 新的坑持续待更。。。

## 写在后面的话
- 这一篇混合着mobx-react，重点是介绍mobx的基础用法
- 下一篇会详细介绍mobx-react（立个flag吧 ==）


