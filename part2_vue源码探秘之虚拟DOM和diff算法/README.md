## `Vue`源码探秘之虚拟`DOM`和`diff`算法

#### `diff`的比较：

新**虚拟**`DOM`和老**虚拟**`DOM`进行`diff`(精细化比较)，算出应该如何最小量更新，最后再反映到真实的`DOM`上。

这里研究三个方面：

①虚拟`DOM`如何被渲染函数(h函数)产生？手写`h`函数

②研究`diff`算法原理

③虚拟`DOM`如何通过diff变为真正的DOM

` snabbdom `

h函数用来产生虚拟节点

虚拟节点的属性

```js
{
    children: undefined
    data: {}
    elm: undeefined // 元素对应的真正节点
    key: undefined // 元素的唯一标识，服务于最小量更新
    sel: 'div'
    text: '我是一个盒子'
}
```

h函数的作用

手写h函数（只实现三个参数的h函数）

```js
h('div', {}, [])
h('div', {}, '文字')
h('div', {}, h())
```

### diff算法的核心`patch函数`

#### patch函数的作用：

①将h函数返回的`vNode`挂载到DOM树上

②将新旧虚拟DOM进行最小量更新

#### patch使用示例：

```js
// 得到盒子和按钮
const container = document.getElementById('container');
const btn = document.getElementById('btn');
// 创建出patch函数
const patch = init([classModule, propsModule, styleModule, eventListenersModule]);
const vnode1 = h('ul', {}, [
    h('li', { key: 'A' }, 'A'),
    h('li', { key: 'B' }, 'B'),
    h('li', { key: 'C' }, 'C'),
    h('li', { key: 'D' }, 'D')
]);
patch(container, vnode1);
const vnode2 = h('ul', {}, [
    h('li', { key: 'D' }, 'D'),
    h('li', { key: 'A' }, 'A'),
    h('li', { key: 'C' }, 'C'),
    h('li', { key: 'B' }, 'B')
]);
// 点击按钮时，将vnode1变为vnode2
btn.onclick = function () {
    patch(vnode1, vnode2);
};
```

`diff`的注意事项：

- 为元素添加key时，key作为唯一标识，此时就是在告诉`diff`算法，他们是同一个DOM节点。

- 只有是同一个虚拟节点（选择器`sel`、key)，才进行精细化比较，否则就暴力删除旧的，插入新的。

- 只进行同层比较，不会进行跨层比较。像这种情况会暴力更新。虽然他的`li`相同，但是并非同层的，会直接插入新的。

  - ```js
    const vnode1 = h('ul', {}, [
        h('li', { key: 'A' }, 'A'),
        h('li', { key: 'B' }, 'B'),
        h('li', { key: 'C' }, 'C'),
        h('li', { key: 'D' }, 'D')
    ]);
    const vnode2 = h('ul', {}, h('section', {}, [
        h('li', { key: 'D' }, 'D'),
        h('li', { key: 'A' }, 'A'),
        h('li', { key: 'C' }, 'C'),
        h('li', { key: 'B' }, 'B')
    ]));
    ```

#### 手写第一次上树时：

首先我们先考虑最简单的情况，即；

```js
const container = document.getElementById('container')
const textNode = h('p', {} , 'HelloWorld')
patch(container, textNode)
```

此时不用考虑嵌套问题：

```js
function patch(oldVnode, newVnode) {
    // 判断传入的第一个参数，是DOM节点还是虚拟节点？
    if (oldVnode.sel == '' || oldVnode.sel == undefined) {
        // 传入的第一个参数是DOM节点，此时要包装为虚拟节点
        oldVnode = vnode(oldVnode.tagName.toLowerCase(), {}, [], undefined, oldVnode);
    }
    // 判断oldVnode和newVnode是不是同一个节点
    if (oldVnode.key == newVnode.key && oldVnode.sel == newVnode.sel) {
        console.log('是同一个节点');// 后面解决
        patchVnode(oldVnode, newVnode) // 第二步要写的函数
    } else {
        console.log('不是同一个节点，暴力插入新的，删除旧的');
        let newVnodeElm = createElement(newVnode);
        
        // 插入到老节点之前
        if (oldVnode.elm.parentNode && newVnodeElm) {
            oldVnode.elm.parentNode.insertBefore(newVnodeElm, oldVnode.elm);
        }
        // 删除老节点
        oldVnode.elm.parentNode.removeChild(oldVnode.elm);
    }
};
// 真正创建节点。将vnode创建为DOM，是孤儿节点，不进行插入
function createElement(vnode) {
    // console.log('目的是把虚拟节点', vnode, '真正变为DOM');
    // 创建一个DOM节点，这个节点现在还是孤儿节点
    let domNode = document.createElement(vnode.sel);
    // 有子节点还是有文本？？
    if (vnode.text != '' && (vnode.children == undefined || vnode.children.length == 0)) {
        // 它内部是文字
        domNode.innerText = vnode.text;
    } else if (Array.isArray(vnode.children) && vnode.children.length > 0) {
        // await code
    }
    // 补充elm属性
    vnode.elm = domNode;
   
    // 返回elm，elm属性是一个纯DOM对象
    return vnode.elm;
};
```

此时，就能顺利将文本节点插上去了。

那么如果插入节点是这种呢？

```js
const myVnode1 = h('ul', {}, [
    h('li', { key: 'A' }, 'A'),
    h('li', { key: 'B' }, 'B'),
    h('li', { key: 'C' }, 'C'),
    h('li', { key: 'D' }, 'D'),
    h('li', { key: 'E' }, 'E')
]);
```

这时就要考虑递归了。

手写创建递归创建子节点：让我们继续完善前面的`createElement`吧

```js
// 真正创建节点。将vnode创建为DOM，是孤儿节点，不进行插入
function createElement(vnode) {
    // console.log('目的是把虚拟节点', vnode, '真正变为DOM');
    // 创建一个DOM节点，这个节点现在还是孤儿节点
    let domNode = document.createElement(vnode.sel);
    // 有子节点还是有文本？？
    if (vnode.text != '' && (vnode.children == undefined || vnode.children.length == 0)) {
        // 它内部是文字
        domNode.innerText = vnode.text;
    } else if (Array.isArray(vnode.children) && vnode.children.length > 0) {
        // 它内部是子节点，就要递归创建节点
        for (let i = 0; i < vnode.children.length; i++) {
            // 得到当前这个children
            let ch = vnode.children[i];
            // 创建出它的DOM，一旦调用createElement意味着：创建出DOM了，并且它的elm属性指向了创建出的DOM，但是还没有上树，是一个孤儿节点。
            let chDOM = createElement(ch);
            // 上树
            domNode.appendChild(chDOM);
        }
    }
    // 补充elm属性
    vnode.elm = domNode;
    // 返回elm，elm属性是一个纯DOM对象
    return vnode.elm;
};
```

至此，我们就完成了第一步；

第二步：`diff`处理新旧节点是同一个节点时

```js
function patchVnode(oldVnode, newVnode) {
    // 判断新旧vnode是否是同一个对象
    if (oldVnode === newVnode) return;
    // 判断新vnode有没有text属性
    if (newVnode.text != undefined && (newVnode.children == undefined || newVnode.children.length == 0)) {
        // 新vnode有text属性
        console.log('新vnode有text属性');
        if (newVnode.text != oldVnode.text) {
            // 如果新虚拟节点中的text和老的虚拟节点的text不同，那么直接让新的text写入老的elm中即可。如果老的elm中是children，那么也会立即消失掉。
            oldVnode.elm.innerText = newVnode.text;
        }
    } else {
        // 新vnode没有text属性，有children
        console.log('新vnode没有text属性');
        // 判断老的有没有children
        if (oldVnode.children != undefined && oldVnode.children.length > 0) {
            // 老的有children，新的也有children，此时就是最复杂的情况。
            updateChildren(oldVnode.elm, oldVnode.children, newVnode.children);// 下面补充
        } else {
            // 老的没有children，新的有children
            // 清空老的节点的内容
            oldVnode.elm.innerHTML = '';
            // 遍历新的vnode的子节点，创建DOM，上树
            for (let i = 0; i < newVnode.children.length; i++) {
                let dom = createElement(newVnode.children[i]);
                oldVnode.elm.appendChild(dom);
            }
        }
    }
}
```

这时我们除了新旧节点均有`children`的情况没解决，其余情况基本已经解决。

划重点`diff`算法的子节点更新策略

接下来写，新旧节点均有孩子的情况。

四种命中查找：(diff算法)

```text
① 新前与旧前
② 新后与旧后
③ 新后与旧前（此种发生了，涉及移动节点，那么新前指向的节点，移动的旧后之后）
④ 新前与旧后（此种发生了，涉及移动节点，那么新前指向的节点，移动的旧前之前）
命中一种就不再进行命中判断了
如果都没有命中，就需要用循环来寻找了。移动到oldStartIdx之前
```



````js

````





