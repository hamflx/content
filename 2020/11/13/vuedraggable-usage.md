---
title: "`vuedraggable` 使用问题记录"
date: 2020-11-13
---

# `vuedraggable` 使用问题记录

- 本文引用的 `Sortable.js` 源码 `commit id` 为：`1dff5e1`
- 本文引用的 `vuedraggable` 源码 `commit id` 为：`2fd91d6`

`vuedraggable` 官方地址：<https://sortablejs.github.io/Vue.Draggable/>

`vuedraggable` 是一个封装了 `Sortable.js` 的 `vue` 组件，将拖动同步到对应的数据模型上，提供了 `Sortable.js` 的所有特性。

但是 `vuedraggable` 提供的 `vue` 组件对动态绑定的属性并不能较好的兼容，会引发一些莫名奇妙的问题。

考虑一个典型的拖动场景，有待办、进行及完成三个泳道，每个泳道里面有若干卡片，待办中的卡片可以拖动到进行的泳道，反之进行的卡片则无法拖动到待办的泳道，若要在拖动时，高亮显示可以放置的泳道，则可以在 `start` 事件记录下拖动的卡片，通过双向绑定设置是否可以放置的类名 `is-droppable`。

```html
<!-- is-droppable 用于设置拖动时，可以放置的样式，在拖动过程中，isLaneDroppable 将会返回 true，否则为 false -->
<div :class="{ 'is-droppable': isLaneDroppable(item) }">
    <Draggable :sort="false">
        <div></div>
    </Draggable>
</div>
```

在这种 `draggable` 组件的父级元素绑定了动态属性的情况下，会导致 `sort` 属性不生效，初步猜测是 `DOM` 结构更新后，`Sortable.js` 未能匹配上，导致 `sort` 的判断失效。

因此，在使用 `draggable` 时，**最好不要在拖动的过程中，修改动态绑定的属性的值，避免拖动时更新 `DOM`**。

避免使用双向绑定，同时再配合 `start` 事件，在该事件中找到所有可以放置的泳道，并通过直接操作 `DOM` 的形式，给可以放置的元素添加 `is-droppable` 类名可以解决该问题。

在该场景上考虑另一个问题，若需要在拖动时，当前鼠标悬停的、可放置的泳道上设置一个有别于可放置的样式（比方说，可放置是虚线边框，悬停的是实现边框且有背景色），基于先前的经验避开使用双向绑定的方法，直接使用 `move` 事件操作对应的 `DOM`，添加 `is-target` 类名。

此时会有一个新的问题，当 `:sort="false"` 触发时（即，鼠标移动到原泳道但非卡片原位置，将会触发拖动的重置，此时卡片的占位符将会回到拖动前的位置），将不会触发 `move` 事件，即，悬停高亮将没办法更新，还是为原先设置的元素高亮。

检查 `Sortable.js` 源码的 `1287-1306` 行：

```typescript
if (revert) {
    parentEl = rootEl; // actualization
    capture();

    this._hideClone();

    //@ts-ignore
    dragOverEvent("revert");

    //@ts-ignore
    if (!Sortable.eventCanceled) {
        if (nextEl) {
        rootEl.insertBefore(dragEl, nextEl);
        } else {
        rootEl.appendChild(dragEl);
        }
    }

    return completed(true);
}
```

可知，在拖动被重置时，会触发 `revert` 事件，检查 `dragOverEvent` 函数的定义可知，这是一个在插件上触发的事件，而非 `Sortable.js` 实例事件，因此，可以编写一个插件，监听该事件，并将该事件在实例上触发，便可以解决该问题。

检查 `vuedraggable` 源代码的第 `197-228` 行：

```javascript
mounted() {
    // ...

    !("draggable" in options) && (options.draggable = ">*");
    this._sortable = new Sortable(this.rootContainer, options);
    this.computeIndexes();
},
```

可以使用以下的方法获取内部的 `Sortable.js` 引用：

```javascript
import Vue from 'vue'
import Draggable from 'vuedraggable'

const Tmp = Vue.extend(Draggable)
const tmp = new Tmp().$mount()
const Sortable = tmp._sortable.constructor
```

参考 `on-spill` 插件，可以编写出如下的插件：

```javascript
function RevertEventPlugin () {}

RevertEventPlugin.prototype = {
  constructor: RevertEventPlugin,
  revertGlobal ({ dispatchSortableEvent }) {
    dispatchSortableEvent('revert')
  }
}

Object.assign(RevertEventPlugin, {
  pluginName: 'revertEventPlugin'
})
```

将以上两部封装为一个模块，用于替代 `vuedraggable` 模块：

```javascript
/*
 * 描述：返回一个可以触发 revert 事件的 vuedraggable 组件
 * 文件名：src/utils/vuedraggableWithRevert.js
 */

import Vue from 'vue'
import Draggable from 'vuedraggable'

function RevertEventPlugin () {}

RevertEventPlugin.prototype = {
  constructor: RevertEventPlugin,
  revertGlobal ({ dispatchSortableEvent }) {
    dispatchSortableEvent('revert')
  }
}

Object.assign(RevertEventPlugin, {
  pluginName: 'revertEventPlugin'
})

const getDraggableWithRevertEvent = () => {
  const Tmp = Vue.extend(Draggable)
  const tmp = new Tmp().$mount()
  const Sortable = tmp._sortable.constructor
  Sortable.mount(RevertEventPlugin)
  return Draggable
}

export default getDraggableWithRevertEvent()
```

在使用时，直接将 `import Draggable from 'vuedraggable'` 替换为 `import Draggable from './src/utils/vuedraggableWithRevert'` 即可，若需要监听 `revert` 事件，添加一个具有 `onRevert` 属性的 `options`。

```html
<Draggable
    :sort="false"
    :options="{ onRevert: handleCardItemRevert }"
/>
</Draggable>
```
