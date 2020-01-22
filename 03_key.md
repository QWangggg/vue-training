## 你知道vue中key的作用和工作原理吗？说说你对它的理解。

### 作用
    1.Vue为了高效渲染，将会复用已有元素而不会进行重新渲染，这样速度是快了，但是有时候我们并不想要复用组件，而需要独立，此时可以通过添加唯一的key值，使两个元素各自渲染。
    2.主要用在 Vue 的虚拟 DOM 算法，在新旧 nodes 对比时辨识 VNodes。如果不使用 key，Vue 会使用一种最大限度减少动态元素并且尽可能的尝试就地修改/复用相同类型元素的算法。而使用 key 时，它会基于 key 的变化重新排列元素顺序，并且会移除 key 不存在的元素。
    有相同父元素的子元素必须有独特的 key。重复的 key 会造成渲染错误。
    3.它也可以用于强制替换元素/组件而不是重复使用它。当你遇到如下场景时它可能会很有用：
        1.完整地触发组件的生命周期钩子
        2.触发过渡

### 工作原理

在源码目录/src/core/vdom/patch.js，我们找到了diff算法执行的位置，在这个文件中有一个sameVnode方法，用来比较节点是否一样。

```
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```
从这里看出，该方法首先会比较两个key是否相同，如果不相同，则直接返回false，而不会继续下面的逻辑，大大提高了性能。

再然后我们来看下在updateChildren方法中,有这么一段：
```
    if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
    if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
    } else {
        vnodeToMove = oldCh[idxInOld]
        if (sameVnode(vnodeToMove, newStartVnode)) {
        patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldCh[idxInOld] = undefined
        canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
        } else {
        // same key but different element. treat as new element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        }
    }
```
当未定义oldKeyToIdx时，调用createKeyToOldIdx方法，如果新元素节点有key，直接从oldKeyToIdx获取，如果没有key，调用findIdxInOld方法。
如果没有找到idxInOld，则创建新元素。

下面来看看这两个方法做了什么：
```
function createKeyToOldIdx (children, beginIdx, endIdx) {
  let i, key
  const map = {}
  for (i = beginIdx; i <= endIdx; ++i) {
    key = children[i].key
    if (isDef(key)) map[key] = i
  }
  return map
}
```
```
function findIdxInOld (node, oldCh, start, end) {
    for (let i = start; i < end; i++) {
      const c = oldCh[i]
      if (isDef(c) && sameVnode(node, c)) return i
    }
  }
```
createKeyToOldIdx创建一个map，将元素上的key值做属性名，索引做属性值，然后返回map。
findIdxInOld遍历老元素上的节点，和新元素开始节点比较，如果找到了，则返回该节点在老元素中的索引。

### **总结一下：**

**首先在对比节点是否相同时，会先对比key值，key值不相同，不再对比之后的逻辑，提高了性能。**

其次在更新时会生成一个以key值为属性，index为值的map对象。当新开始节点有key值，更新时可通过key值直接找到相应的节点的索引，如果没有则会通过遍历找到对应的节点的索引，返回给idxInOld。**通过key值直接找到元素索引比通过遍历找更快，也提高了性能。**

