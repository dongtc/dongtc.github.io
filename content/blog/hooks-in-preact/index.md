---
title: React Hooks 在Preact中的实现
date: "2019-11-01"
description: ""
---

React Hooks已经发布一段时间，[Preact](https://github.com/preactjs/preact)在最新的版本10.x中也添加了Hooks的实现。

Preact是一个精简版(3k)的React实现，其对Hooks的实现代码也很简单，通过对源码的阅读，可以非常容易的理解Hooks的各种规则。

Hooks只能用在Function组件里，我们知道Class组件可以维持状态，而Funciton组件是没有状态的，那么Hooks的状态是存在哪里的呢？

这里就需要对React的实现有一定了解了，实际上一个Function组件，React内部还是会将其转化成Class，也就是通过 `new React.Component()` 来生成一个组件实例，
然后将Function函数赋值给实例的 `render` 方法。

所以一个Function组件也是有一个组件实例的，所有的Hooks状态都存在这个实例上。

```jsx
// 通过函数是否有原型方法render来判断是否是Class组件
if ('prototype' in newType && newType.prototype.render) {
    newVNode._component = c = new newType(newProps, cctx);
} else {
    newVNode._component = c = new Component(newProps, cctx);
    c.constructor = newType;
    c.render = doRender;
}
function doRender(props, state, context) {
	return this.constructor(props, context);
}
```

这样，一个组件无论是Class还是Function，都是通过调用实例的 `render` 方法来拿到它要渲染的内容


在调用 `render` 前我们拿到组件的实例，并存到当前正在执行的组件变量中

```jsx
if ((tmp = options._render)) tmp(newVNode);

tmp = c.render(c.props, c.state, c.context);
```

`options._render` 是对外提供的钩子函数，用来功能扩展，现在我们进入Hooks的代码去看下。

```jsx
let currentIndex;
let currentComponent;

options._render = vnode => {
	currentComponent = vnode._component;
	currentIndex = 0;
};
```
`currentComponent` 是一个全局变量，存放着当前正在执行(`render`)的函数实例，因为同一时间有且只有一个组件在执行，所以可以存在全局变量里

接下来就到`useState`了

```jsx
export function useState(initialState) {
	return useReducer(invokeOrReturn, initialState);
}
```
很明显，`useState`使用`useReducer`实现的，只是少了一个`reducer`处理的过程。

为了便于理解，改下`useReduce`，直接实现`useState`

```jsx
export function useState(initialState) {
    // 通过currentIndex获取当前hookState
    const hookState = getHookState(currentIndex++);
    // 如果是第一次执行
	if (!hookState._component) {
        hookState._component = currentComponent;
        // 把value和setValue赋值，并返回
		hookState._value = [
			initialState,  // _value[0]
			newValue => {
				if (hookState._value[0] !== nextValue) {
                    // 当执行setValue时，会先把newValue到_value[0]
                    hookState._value[0] = nextValue;
                    // 然后触发更新
					hookState._component.setState({});
				}
			}
		];
	}
	return hookState._value;
}

let currentComponent
function getHookState(index) {
    // 在组件实例上有一个list，每次执行到hook函数，就根据currentIndex拿到hook值
	const hooks =
		currentComponent.__hooks ||
		(currentComponent.__hooks = { _list: [], _pendingEffects: [] });

	if (index >= hooks._list.length) {
		hooks._list.push({});
    }
    // 取hook数据是通过数组下标，所以你的写的hook函数的顺序不能改变!
	return hooks._list[index];
}
```
如果我们这样定义一个`useState`:
```jsx
const [value, setValue] = useState(0)
```
当执行setValue时，实际上执行了`hookState._component.setState({})`，它会触发组件实例`render`方法的执行，下一次执行function代码时，
`hookState._value[0]`已经被改变，直接返回，这样拿到的就是新的value了。

接下来看下useEffect

```jsx
function useEffect(callback, args) {
    // 第一步依然是获取hookState
    const state = getHookState(currentIndex++);
    // 然后对传入的参数对比有没有变化
	if (argsChanged(state._args, args)) {
        // 有变化就存下来
		state._value = callback;
        state._args = args;
        // 添加到一个待执行队列里
		currentComponent.__hooks._pendingEffects.push(state);
	}
}
```
参数对比的函数是这样的

```js
function argsChanged(oldArgs, newArgs) {
	return !oldArgs || newArgs.some((arg, index) => arg !== oldArgs[index]);
}
```
我们知道useEffect的第二个参数可以控制更新，有三种情况：
1. 无参数，那就认为每次都变化，都执行
2. [], 一个空数组，只会执行一次，用来模拟componentDidMount
3. 非空数组，依次对比数组项，有变化就执行

当然看代码就知道2、3是一种情况

如果有变化，我们把state存入当前组件的待更新队列里，接下来就是要触发队列的执行了，什么时候执行呢？

useEffect的执行需要在组件render完毕，且浏览器下一帧渲染完毕，这里就可以用`requestAnimationFrame`

```jsx
// 这里存的是组件的队列，每个组件中有effect队列
let afterPaintEffects = [];
options.diffed = vnode => {
	const c = vnode._component;
	if (!c) return;

	const hooks = c.__hooks;
	if (hooks) {
        // 组件执行结束如果有effect，添加到组件队列中
		if (hooks._pendingEffects.length) {
			afterPaint(afterPaintEffects.push(c)); 
		}
	}
};
afterPaint = newQueueLength => {
    // 这里判断数组长度为1，也就是当afterPaintEffects 待执行组件队列第一次有值，就开始准备flush
    if (newQueueLength === 1 ) {
        requestAnimationFrame(flushAfterPaintEffects);
    }
};
```

如果看过`setState`的异步执行代码，会发现这里如出一撤，在队里长度为1时执行`requestAnimationFrame`,
`requestAnimationFrame`里的函数会延迟到下一帧才执行，在空闲的这一段时间，待执行队列里会被同步的push数据，所以多次执行setState，并不会多次触发更新，
useEffect也是如此。

接下来看flush方法
```jsx
flushAfterPaintEffects() {
	afterPaintEffects.some(component => {
		if (component._parentDom) {
			component.__hooks._pendingEffects.forEach(invokeCleanup);
			component.__hooks._pendingEffects.forEach(invokeEffect);
			component.__hooks._pendingEffects = [];
		}
	});
	afterPaintEffects = [];
}
```

一个循环遍历执行掉队列，就结束了

最后，useEffect接受的函数，可以返回一个销毁函数,
这里的`invokeCleanup`就是先执行销毁。

```jsx
function invokeCleanup(hook) {
	if (hook._cleanup) hook._cleanup();
}

function invokeEffect(hook) {
    const result = hook._value();
    // 如果有销毁函数，就存下来
	if (typeof result === 'function') hook._cleanup = result;
}
```
以上就是核心的两个Hooks实现了
