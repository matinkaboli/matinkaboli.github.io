---
layout: post
title: Optional Chaining in JavaScript
date: 2018-05-21 00:49
categories: programming
minutes: 2
---

Before Introduction, you should know that Optional Chaining is currently in
[Stage 1](https://tc39.github.io/process-document/), here is
[the repository](https://github.com/tc39/proposal-optional-chaining).

## So what exactly is Optional Chaining?

Optional Chaining lets us check if an object exists before trying to
access its properties and methods. There are some other languages that have
something like Optional Chaining. Take C# as an example that has a
Null Conditional Operator.

This is what we do if we want to access `width` property of `size` in `car`
object.

#### Property Chaining

```javascript
  const car = { model: 'BMW' };

  console.log(car.size.width);

  // TypeError: Cannot read property 'width' of undefined
```
And as you can see, it throws an error. So let's fix it, shall we?

```javascript
  const car = { model: 'BMW' };

  console.log(car.size && car.size.width);

  // undefined
```
Now, it doesn't throw an error. It just returns `undefined`.

But this is not a proper way, sometimes objects get more complicated and we have
to write dozens of conditions.

There are definitely other ways to access properties without getting an
error, like using **Try/Catch**, **Ternary Operators**, **Libraries** etc..

## Optional Chaining

Optional Chaining is currently in stage 1 and there are 4 stages.

```javascript
  const car = { model: 'BMW' };

  console.log(car?.size?.width);

  // undefined
```
And that's it. The operator checks if whatever is on the left-hand side of
the *?.* is **null** or **undefined**. If it is, then it returns *undefined*
otherwise continues.

### Other cases

```javascript
  a?.[x];

  a == null ? undefined : a[x];


  a?.b();

  a == null ? undefined : a.b();


  a?.();

  a == null ? undefined : a();
```
