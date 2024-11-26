
# 前言


watch这个API大家都很熟悉，今天这篇文章欧阳来带你搞清楚Vue3的watch是如何实现对响应式数据进行监听的。注：本文使用的Vue版本为`3.5.13`。


关注公众号：【前端欧阳】，给自己一个进阶vue的机会


# 看个demo


我们来看个简单的demo，代码如下：



```

  <button @click="count++">count++button>


<script setup lang="ts">
import { ref, watch } from "vue";
const count = ref(0);
watch(count, (preVal, curVal) => {
  console.log("count is changed", preVal, curVal);
});
script>

```

这个demo很简单，使用watch监听了响应式变量`count`，在watch回调中进行了console打印。如何有个button按钮，点击后会count\+\+。


# 开始打断点


现在我们第一个断点应该打在哪里呢？


我们要看watch的实现，那么当然是给我们demo中的watch函数打个断点。


首先执行`yarn dev`将我们的demo跑起来，然后在浏览器的network面板中找到对应的vue文件，右键点击**Open in Sources panel**就可以在source面板中打开我们的代码啦。如下图
![source](https://img2024.cnblogs.com/blog/1217259/202411/1217259-20241125232517396-2020390967.png)


然后给watch函数打个断点，如下图：
![debug](https://img2024.cnblogs.com/blog/1217259/202411/1217259-20241125232531347-2007949564.png)


接着刷新页面，此时代码将会停留在断点出。将断点走进watch函数，代码如下：



```
function watch(source, cb, options) {
  return doWatch(source, cb, options);
}

```

从上面的代码可以看到在watch函数中直接返回了`doWatch`函数。


将断点走进`doWatch`函数，在我们这个场景中简化后的代码如下（**为了方便大家理解，本文中会将scheduler任务调度相关的代码移除掉，因为这个不影响watch的主流程**）：



```
function doWatch(source, cb, options = EMPTY_OBJ) {
  const baseWatchOptions = extend({}, options);
  const watchHandle = baseWatch(source, cb, baseWatchOptions);
  return watchHandle;
}

```

从上面的代码可以看到底层实际是在执行`baseWatch`函数，而这个`baseWatch`就是由`@vue/reactivity`包中导出的watch函数。关于这个`baseWatch`函数的由来可以看看欧阳之前的文章： [Vue3\.5新增的baseWatch让watch函数和Vue组件彻底分手](https://github.com)


# `baseWatch`函数


将断点走进`baseWatch`函数，在我们这个场景中简化后的代码如下：



```
const INITIAL_WATCHER_VALUE = {}

function watch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb?: WatchCallback | null,
  options: WatchOptions = EMPTY_OBJ
): WatchHandle {
  let effect: ReactiveEffect;
  let getter: () => any;

  if (isRef(source)) {
    getter = () => source.value;
  }

  let oldValue: any = INITIAL_WATCHER_VALUE;

  const job = () => {
    if (cb) {
      const newValue = effect.run();
      if (hasChanged(newValue, oldValue)) {
        const args = [
          newValue,
          oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
          boundCleanup,
        ];

        cb(...args);
        oldValue = newValue;
      }
    }
  };
  effect = new ReactiveEffect(getter);
  effect.scheduler = job;

  oldValue = effect.run();
}

```

首先定义了两个变量`effect`和`getter`，`effect`是`ReactiveEffect`类的实例。


接着就是使用`isRef(source)`判断watch监听的是不是一个ref变量，如果是就将`getter`函数赋值为`getter = () => source.value`。这么做的原因是为了保持一致（watch也可以直接监听一个getter函数），并且后面会对这个getter函数进行读操作触发依赖收集。


我们知道watch的回调中有`oldValue`和`newValue`这两个字段，在`watch`函数内部有个字段也名为`oldValue`用于存旧的值。


接着就是定义了一个`job`函数，我们先不看里面的代码，执行这个`job`函数就会执行watch的回调。


然后执行`effect = new ReactiveEffect(getter)`，这个`ReactiveEffect`类是一个底层的类。**在Vue的设计中，所有的订阅者都是继承的这个`ReactiveEffect`类**。比如[watchEffect](https://github.com)、[computed()](https://github.com):[豆荚加速器官网PodHub](https://doujiaa.com)、render函数等。


在我们这个场景中`new ReactiveEffect`时传入的`getter`函数就是`getter = () => source.value`，这里的`source`就是watch监听的响应式变量`count`。


接着将`job`函数赋值给`effect.scheduler`属性，在`ReactiveEffect`类中依赖触发时就会执行`effect.scheduler`方法（接下来会讲）。


最后就是执行`effect.run()`拿到初始化时watch监听变量的值，这个`run`方法也是在`ReactiveEffect`类中。接下来也会讲。


# `ReactiveEffect`类


前面我们讲过了`ReactiveEffect`是Vue的一个底层类，所有的订阅者都是继承的这个类。将断点走进`ReactiveEffect`类，在我们这个场景中简化后的代码如下：



```
class ReactiveEffect implements Subscriber, ReactiveEffectOptions {
  constructor(fn) {
    this.fn = fn;
  }

  run(): T {
    const prevEffect = activeSub;
    activeSub = this;
    try {
      return this.fn();
    } finally {
      activeSub = prevEffect;
    }
  }

  trigger(): void {
    this.scheduler();
  }
}

```

在new一个`ReactiveEffect`实例时传入的getter函数会赋值给实例的`fn`方法。（实际的`ReactiveEffect`代码比这个要复杂很多，感兴趣的同学可以去看源代码）


我们回到前面讲过的`baseWatch`函数中的最后一块：`oldValue = effect.run()`。这里执行了`effect`实例的`run`方法拿到watch监听变量的值，并且赋值给`oldValue`变量。


因为我们如果不使用`immediate: true`，那么Vue会等watch监听的变量改变后才会触发watch回调，回调中有个字段叫`oldValue`，这个`oldValue`就是初始化时执行`run`方法拿到的。


比如我们这里`count`初始化的值是0，初始化执行`oldValue = effect.run()`后就会给`oldValue`赋值为0。当点击**count\+\+**按钮后，`count`的值就变成了1，所以在watch回调第一次触发的时候他就知道`oldValue`的值是0啦。


除此之外，在`run`方法中还有收集依赖的作用。Vue维护了一个全局变量`activeSub`表示当前active的订阅者是谁，在同一时间只可能有一个active的订阅者，不然触发get拦截进行依赖收集时就不知道该把哪个订阅者给收集了。


在`run`方法中将当前的`activeSub`给存起来，等下面的代码执行完了后将全局变量`activeSub`改回去。


接着就是执行`activeSub = this;`将当前的watch设置为全局变量`activeSub`。


接下来就是执行`return this.fn()`，前面我们讲过了这个`this.fn()`方法就是watch监听的getter函数。由于我们watch监听的是一个响应式变量`count`，在前面处理后他的getter函数就是`getter = () => source.value;`。这里的source就是watch监听的变量，这个getter函数实际就是`getter = () => count.value;`


那么这里执行`return this.fn()`就是执行`() => count.value`，将会触发响应式变量`count`的get拦截。在get拦截中会进行依赖收集，由于此时的全局变量`activeSub`已经变成了订阅者watch，所以响应式变量`count`在依赖收集的过程中收集的订阅者就是watch。这样响应式变量`count`就和订阅者watch建立了依赖收集的关系。关于Vue3\.5依赖收集和依赖触发可以看看欧阳之前的文章： [看不懂来打我！让性能提升56%的Vue3\.5响应式重构](https://github.com)


当我们点击**count\+\+**后会修改响应式变量`count`的值，就会进行依赖触发，经过一堆操作后最后就会执行到这里的`trigger`方法中。在`trigger`方法中直接执行`this.scheduler()`，在前面已经对`scheduler`方法进行了赋值，回忆一下`baseWatch`函数的代码。如下：



```
const INITIAL_WATCHER_VALUE = {}

function watch(
  source: WatchSource | WatchSource[] | WatchEffect | object,
  cb?: WatchCallback | null,
  options: WatchOptions = EMPTY_OBJ
): WatchHandle {
  let effect: ReactiveEffect;
  let getter: () => any;

  if (isRef(source)) {
    getter = () => source.value;
  }

  let oldValue: any = INITIAL_WATCHER_VALUE;

  const job = () => {
    if (cb) {
      const newValue = effect.run();
      if (hasChanged(newValue, oldValue)) {
        const args = [
          newValue,
          oldValue === INITIAL_WATCHER_VALUE ? undefined : oldValue,
          boundCleanup,
        ];

        cb(...args);
        oldValue = newValue;
      }
    }
  };
  effect = new ReactiveEffect(getter);
  effect.scheduler = job;

  oldValue = effect.run();
}

```

这里将`job`函数赋值给`effect.scheduler`方法，所以当响应式变量`count`的值改变后实际就是在执行这里的`job`函数。


在`job`函数中首先判断是否有传入watch的callback函数，然后执行`const newValue = effect.run()`。


执行这行代码有两个作用：


第一个作用是重新执行getter函数，也就是`getter = () => count.value;`，拿到最新`count`的值，将其赋值给`newValue`。


第二个作用是watch除了监听响应式变量之外还可以监听一个getter函数，那么在`getter`函数中就可以类似computed一样在某些条件下监听变量A，某些条件下监听变量B。这里的第二个作用是重新收集依赖，因为此时watch可能从监听变量A变成了监听变量B。


接着就是执行`if (hasChanged(newValue, oldValue))`判断watch监听的变量新的值和旧的值是否相等，如果不相等才去执行`cb(...args)`触发watch的回调。最后就是将当前的`newValue`赋值给`oldValue`，下次触发watch回调时作为`oldValue`字段。


# 总结


这篇文章讲了watch如何对响应式变量进行监听，其实底层依赖的是`@vue/reactivity`包的`baseWatch`函数。在`baseWatch`函数中会使用`ReactiveEffect`类new一个`effect`实例，这个`ReactiveEffect`类是一个底层的类，Vue的订阅者都是基于这个类去实现的。


如果没有使用`immediate: true`，初始化时会去执行一次`effect.run()`对watch监听的响应式变量进行读操作并且将其赋值给`oldValue`。读操作会触发get拦截进行响应式变量的依赖收集，会将当前watch作为订阅者进行收集。


当响应式变量的值改变后会触发set拦截，进而依赖触发。前一步将watch也作为订阅者进行了收集，依赖触发时也会通知到watch，所以此时会执行watch中的`job`函数。在`job`函数中会再次执行`effect.run()`拿到响应式变量最新的值赋值给`newValue`，同时再次进行依赖收集。如果`oldValue`和`newValue`不相等，那么就触发watch的回调，并且将`oldValue`和`newValue`作为参数传过去。


关注公众号：【前端欧阳】，给自己一个进阶vue的机会


![](https://img2024.cnblogs.com/blog/1217259/202406/1217259-20240606112202286-1547217900.jpg)


另外欧阳写了一本开源电子书[vue3编译原理揭秘](https://github.com)，看完这本书可以让你对vue编译的认知有质的提升。这本书初、中级前端能看懂，完全免费，只求一个star。


