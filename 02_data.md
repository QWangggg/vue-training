## Vue组件data选项为什么必须是个函数而Vue的根实例则没有此限制？

### 官方文档描述：
    根据官方文档的说法，一个组件的data必须是一个函数，因此每个实例可以维护一份被返回对象的独立的拷贝。如果没有这条规则，操作一个组件实例时可能同时影响到其他组件实例。

### 从源码上来看：
首先从init方法开始，该方法中有mergeOptions方法，合并组件属性到vm。

```
Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    ...

    // merge options
    if (options && options._isComponent) {
      // optimize internal component instantiation
      // since dynamic options merging is pretty slow, and none of the
      // internal component options needs special treatment.
      initInternalComponent(vm, options)
    } else {
      vm.$options = mergeOptions(
        resolveConstructorOptions(vm.constructor),
        options || {},
        vm
      )
    }

    /* istanbul ignore else */
    ...
  }
```

从mergeOptions方法中我们可以找到合并各个属性的方法，其中合并data属性的方法源码如下：

```
export function mergeDataOrFn (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  if (!vm) {
    if (!childVal) {
      return parentVal
    }
    if (!parentVal) {
      return childVal
    }
    return function mergedDataFn () {
      return mergeData(
        typeof childVal === 'function' ? childVal.call(this, this) : childVal,
        typeof parentVal === 'function' ? parentVal.call(this, this) : parentVal
      )
    }
  } else {
    return function mergedInstanceDataFn () {
      // instance merge
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm, vm)
        : childVal
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm, vm)
        : parentVal
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```
上述源码体现了data选项为什么在组件中需要是函数，而在根目录中不需要。

对于子data，判断是否是函数，如果是函数则执行，返回一个对象，挂载到vm上。

**此时若data是对象形式，将该data直接挂载到了vm上，则另一个组件实例将和此组件实例共用一个data**

对于根目录来说，一个Vue实例只有一个根目录，它使用函数或是对象来定义data，都只有根目录使用，不会有第二个根目录而造成影响。