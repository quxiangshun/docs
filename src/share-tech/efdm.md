# Enfalme: EFDM经验总结 {#enflame-efdm}

## 编程思想-复用性 {#framework-design}

### 面临问题 {#framework-problems}

- **维护成本高**：需要同时维护web版本和PC版本；
- 可阅读性差：代码篇幅太长；
- 基础环境版本低：最新相关插件版本不兼容；

### 设计方案 {#design-scheme}

- 把web和electron版本整合到一个框架

优点：
  ```text
    兼顾了electron的性能，同时保留了web版本的兼容。
  ```
缺点：
  ```text
    需要在代码中针对调用方式做不同的逻辑处理，增加了代码的耦合性；
  ```

- 开发一套web版本的代码，套一层electron的壳

优点：
  ```text
    前后端分离，松耦合，代码维护性较高。
  ```
缺点：
  ```text
    对于PC端需要调用本地库的，增加了网络开销。
  ```

### 解决思路 {#solution}
无论哪种编程语言都强调代码复用这一核心理念。

- 开发人员通过组件设计将功能模块化，有效地促进了代码的复用和扩展，从而提升了开发效率并降低了维护成本。
例如：EFDM一个register页面动辄两三千行代码，阅读或更改比较困难。好几个功能都能单独抽离成单个组件，减少单个页面的篇幅。

在修改过程中，我新增了`createNewView.vue`和`dumpDialog.vue`两个组件。 通过以下方式就可以在register页面实现新增view和dump的功能。

```vue
<script lang="tsx">
import CreateNewViewDialog from "./createNewViewDialog.vue";
import DumpDialog from "./dumpDialog.vue";
</script>
<template>
  <createNewViewDialog ref="newViewRef" @addToNewView="addToNewView" />
  <DumpDialog ref="dumpFormRef" @addToNewView="addToNewView" />
</template>
```
其实页面还有很多可以优化的地方，后续需要继续改进，使代码看起来更加优雅。

- 鲁迅说“要站在巨人的肩膀上开发”，
![img/lu_xun_said.png](img/lu_xun_said.png)

使用`mouse-menu`实现右击事件

```vue
<script lang="tsx">
function childrenOptions(multi = false) {
  const views = [];
  const tableRef = ref();
  const children = [
    {
      label: "New View",
      fn: row => {
        // newViewRef.value?.handleOpen(props.currentIp, rows, views);
      }
    }
  ];

  views.forEach(item => {
    children.push({
      label: item,
      fn: row => {
        // addToNewView({
        //   viewName: item,
        //   rows
        // });
      }
    });
  });
  return children;
}
function menuOptions() {
  const cs = tableRef.value.getTableRef().getSelectionRows();
  const noSelected = !(cs && cs.length > 0);

  return {
    menuList: [
      {
        label: ({ addr }) => `ID为：${addr}`,
        disabled: true
      },
      {
        label: "Add this to",
        children: childrenOptions(false)
      },
      {
        label: "Add selected to",
        disabled: noSelected,
        children: childrenOptions(true)
      },
      {
        label: "Export selected",
        disabled: noSelected,
        fn: () => exportSelData()
      },
      {
        label: "Monitor selected",
        disabled: noSelected,
        fn: row =>
          message(
            `您修改了第 ${
              tableData.value.findIndex(v => v.id === row.id) + 1
            } 行，数据为：${JSON.stringify(row)}`,
            {
              type: "success"
            }
          )
      },
      {
        line: true
      },
      {
        label: "Dump registers...",
        fn: () =>
          dumpFormRef.value?.handleOpen(
            activeName.value,
            ssmMethod.value,
            currentAddr.value
          )
      }
    ]
  };
}
</script>
```

之前通过标签判断实现右击事件，此处还需开发相关的逻辑，设置判断条件，设置样式等。

```vue
<template>
  <div
    class="context-menu"
    :style="{left:menuPoint[0]+ 'px',top:menuPoint[1]+ 'px'}"
    @mousedown.stop=""
    v-show="!menu.hide">
    <ul class="context-ul">
      <li v-show="!selViewKey">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              :class="{disabled: existViewMap[menuRow.name] && existViewMap[menuRow.name].indexOf(view.id) !== -1}"
              @click="addOneToView(view.id)">{{view.name}}</li>
            <el-button class="menu-button" size="mini" @click="addOneToNewView()">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"><i class="fa fa-plus"></i></span>Add this to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li v-show="!selViewKey" :class="{disabled:disableMenuAll}">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              @click="addSelToView(view.id)">{{view.name}}</li>
            <el-button class="menu-button" size="mini" @click="addSelToNewView()">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"></span>Add selected to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li v-show="selViewKey" @click="removeOneFromView()"><span class="menu-icon"><i class="fa fa-remove" aria-hidden="true"></i></span>Remove this</li>
      <li v-show="selViewKey" :class="{disabled:disableMenuAll}" @click="removeSelFromView()"><span class="menu-icon"></span>Remove selected</li>
      <li v-show="selViewKey">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              v-show="!(view.id === selViewKey)"
              :class="{disabled: existViewMap[menuRow.name] && existViewMap[menuRow.name].indexOf(view.id) !== -1}"
              @click="moveOneToView(view.id)">{{view.name}}
            </li>
            <el-button class="menu-button" @click="moveOneToNewView()" size="mini">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"><i class="fa fa-sign-out" aria-hidden="true"></i></span>Move this to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li v-show="selViewKey" :class="{disabled:disableMenuAll}">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              v-show="!(view.id === selViewKey)"
              @click="moveSelToView(view.id)">{{view.name}}
            </li>
            <el-button class="menu-button" @click="moveSelToNewView()" size="mini">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"></span>Move selected to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li v-show="selViewKey">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              v-show="!(view.id === selViewKey)"
              :class="{disabled: existViewMap[menuRow.name] && existViewMap[menuRow.name].indexOf(view.id) !== -1}"
              @click="addOneToView(view.id)">{{view.name}}
            </li>
            <el-button class="menu-button" @click="addOneToNewView()" size="mini">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"><i class="fa fa-copy" aria-hidden="true"></i></span>Copy this to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li v-show="selViewKey" :class="{disabled:disableMenuAll}">
        <el-popover
          placement="right-start"
          trigger="hover">
          <ul class="context-ul" @mousedown.stop="" v-show="!menu.hide">
            <li
              v-for="view in viewList"
              v-show="!(view.id === selViewKey)"
              @click="addSelToView(view.id)">{{view.name}}</li>
            <el-button class="menu-button" @click="addSelToNewView()" size="mini">New view</el-button>
          </ul>
          <span class="menu-span" slot="reference"><span class="menu-icon"></span>Copy selected to<i class="fa fa-angle-right"></i></span>
        </el-popover>
      </li>
      <li :class="{disabled:disableMenuAll}" @click="exportSelData()">
        <span class="menu-icon"><i class="fa fa-download" aria-hidden="true"></i></span>Export selected
      </li>
      <li :class="{disabled:disableMenuAll || !gdbClient.connected}" @click="monitorSelData()">
        <span class="menu-icon"><i class="fa fa-caret-square-o-right" aria-hidden="true"></i></span>Monitor selected
      </li>
      <hr>
      <li :class="{disabled:!gdbClient.connected}" @click="showDump = true;menu.hide = true">
        <span class="menu-icon"><i class="fa fa-cloud-download" aria-hidden="true"></i></span>Dump registers...
      </li>
    </ul>
  </div> 
</template>
 ```

