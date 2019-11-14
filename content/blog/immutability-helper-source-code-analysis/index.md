---
title: immutability-helper 源码分析
date: "2019-11-14"
description: ""
---

[immutability-helper](https://github.com/kolodny/immutability-helper) 是React推荐的两个immutable库之一，另一个是[immer](https://github.com/immerjs/immer)，代码比较简洁，先看下简单的使用，然后分析源码实现。

```js
import update from 'immutability-helper';

const state1 = {a: [1], b: {}};
const state2 = update(state1, {a: {$push: [2]}});

expect(state1).not.toBe(state2);
expect(state1.a).not.toBe(state2.a);
expect(state1.b).toBe(state2.b);
```

immutability-helper使用类似MongoDB的语法来定义对对象的操作，除了`$push`还有`$set`、`$splice`...等等。

核心代码如下：

```js
update(object, $spec) {
	const spec = (typeof $spec === 'function') ? { $apply: $spec } : $spec;
  let nextObject = object;
	getAllKeys(spec).forEach(key => {
		if (hasOwnProperty.call(this.commands, key)) {
			const objectWasNextObject = object === nextObject;
			nextObject = this.commands[key](spec[key], nextObject, spec, object);
			if (objectWasNextObject && this.isEquals(nextObject, object)) {
				nextObject = object;
			}
		}
		else {
			const nextValueForKey = type(object) === 'Map'
				? this.update(object.get(key), spec[key])
				: this.update(object[key], spec[key]);
			const nextObjectValue = type(nextObject) === 'Map'
				? nextObject.get(key)
				: nextObject[key];
			if (!this.isEquals(nextValueForKey, nextObjectValue)
				|| typeof nextValueForKey === 'undefined'
					&& !hasOwnProperty.call(object, key)) {
				if (nextObject === object) {
					nextObject = copy(object);
				}
				if (type(nextObject) === 'Map') {
					nextObject.set(key, nextValueForKey);
				}
				else {
					nextObject[key] = nextValueForKey;
				}
			}
		}
	});
	return nextObject;
}
```
先把原始对象赋值给新变量`nextObject`，`getAllKeys`就是`Object.keys`只是添加了对`Symbol`的支持，如果key是一个操作符($push、$set...)那就执行这个操作方法，
我们以`$push`为例：
```js
$push(value, nextObject, spec) {
  return value.length ? nextObject.concat(value) : nextObject;
}
```
使用数组的`concat`方法会返回一个新的数组，而如果用`push`方法是不会改变原数组的，所有操作符方法都要做到这一点，返回新对象，新的对象赋值给nextObject。

而如果key不是一个操作符，那么就递归调用`update`，将调用update返回的子对象的值和直接取key得到的子对象的值对比，如果发生变化那就要把当前对象浅拷贝，因为子对象改变，它的所有父级对象都要改变，最后子对象赋值到当前对象上，并返回。

整个过程很简单就是一个递归

这里有两处不好理解的地方

```js
if (hasOwnProperty.call(this.commands, key)) {
  const objectWasNextObject = object === nextObject;
  nextObject = this.commands[key](spec[key], nextObject, spec, object);
  if (objectWasNextObject && this.isEquals(nextObject, object)) {
    nextObject = object;
  }
}
```
`isEquals`就是用`===`来判断的，如果两个对象相等了，为啥还要进行`nextObject = object`赋值呢。

原因是这里`isEquals`是可以扩展的，比如你可以用深拷贝来实现，自定义相等的逻辑。

还有一点就是
```js
const objectWasNextObject = object === nextObject;
```

$spec不支持多个操作符如`{a: {$push: [2], $set: [2]}}`，那什么时候`objectWasNextObject`为false，通过在这里打个条件断点，跑下测试用例，发现false的场景为：

```js
it('can handle nibling directives', () => {
  const obj = {a: [1, 2, 3], b: 'me'};
  const spec = {
    a: {$splice: [[0, 2]]},
    $merge: {b: 'you'},
  };
  expect(update(obj, spec)).toEqual({a: [3], b: 'you'});
});
```
对`a`属性修改后，还可以通过`$merge`对整个对象修改，这个时候整个对象已经被浅拷贝改过了，就不需要再判断一次了。

immutable要实现的基本功能是不改变原对象，它返回的对象中子对象修改，它的所有父级对象引用都要改变，而它的兄弟节点对象引用不变。这就是它比深拷贝性能高的原因。

`immutability-helper`通过声明修改路径和修改操作符的方式，利用递归，很简洁的实现了immutable。
