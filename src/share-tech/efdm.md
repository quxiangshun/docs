# Enfalme: EFDM经验总结 {#enflame-efdm}

## 编程思想-复用性 {#framework-design}

### 面临问题 {#framework-problems}

- **维护成本高**：需要同时维护web版本和PC版本；
- 基础环境版本低：最新相关插件版本不兼容；
- 可阅读性差：代码篇幅太长；

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
    对于PC端需要调用本地库的，增加了网络开销。(根据之前web评估，在可接受的范围内)。
  ```

### 插件优势 {#solution}
无论哪种编程语言都强调代码复用这一核心理念。

- 开发人员通过组件设计将功能模块化，有效地促进了代码的复用和扩展，从而提升了开发效率并降低了维护成本。
例如：EFDM一个register页面动辄两三千行代码，阅读或更改比较困难。好几个功能都能单独抽离成单个组件，减少单个页面的篇幅。

在修改过程中，我新增了`createNewView.vue`和`dumpDialog.vue`两个组件。 通过以下方式就可以在register页面实现新增view和dump的功能。

```vue
<script lang="ts">
import CreateNewViewDialog from "./creat~~~~eNewViewDialog.vue";
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

### 混合编码 {#mix-coding}

页面使用`.vue`文件，逻辑处理采用`.tsx`文件。
* 把所有的操作移到tsx文件中，简化页面逻辑。
```text
  新增hook.tsx文件、types.ts文件等  
```
* tsx更加灵活，写法更帖子js和函数式编程范式。譬如处理table字段展示时，可以通过js语法更好的控制显示。

```vue
<template>
  <pure-table
    ref="tableRef"
    class="register-table"
    row-key="addr"
    :data="tableData"
    :columns="columns"
    :expand-row-keys="expandRowKeys"
    stripe
    border
    :default-sort="{ prop: 'name', order: 'ascending' }"
    :pagination="showIpTable ? { ...pagination } : {}"
    :style="getPureTableStyle"
    @expand-change="expandChange"
    @page-size-change="handleSizeChange"
    @page-current-change="handleCurrentChange"
    @row-contextmenu="showMouseMenu"
  >
    <template #expand="{ row }">
      <pure-table :data="row.fieldArr" :columns="childrenTableColumns" />
    </template>
  </pure-table>
</template>
```
```tsx
  const columns: TableColumnList = [
    {
      type: "selection",
      align: "left"
    },
    {
      type: "expand",
      slot: "expand",
      align: "left"
    },
    {
      // 自定义表头，tsx用法
      headerRenderer: () => (
        <>
          <span>Name</span>
          <el-input
            v-model={search.value}
            class="ml-5"
            style={{ width: "300px", maxWidth: "600px", marginLeft: "5px" }}
            placeholder={filterByPlaceholder.value}
          >
            {{
              append: () => (
                <el-select
                  v-model={filterBy.value}
                  placeholder="N"
                  style={{ width: "56px" }}
                >
                  <el-option label="Name" value="name" />
                  <el-option label="Address" value="address" />
                </el-select>
              )
            }}
          </el-input>
          <el-checkbox
            v-show={showIpTable.value}
            v-model={globalSearch.value}
            style={{ marginLeft: "5px" }}
            label="Global Search"
          />
        </>
      ),
      prop: "name"
    },
    {
      label: "Address",
      prop: "addr",
      fixed: "right",
      width: 150,
      cellRenderer: ({ row }) => <>0x{row.addr.toString(16).padStart(8, "0")}</>
    },
    {
      fixed: "right",
      prop: "value",
      width: 250,
      headerRenderer: () => (
        <>
          <div style="display: flex; justify-content: space-between; align-items: center;">
            <div>Value</div>
            <div style="display: flex;">
              <el-tooltip
                content="Query current values of the selected registers (Shift + R)"
                placement="top"
              >
                <el-button
                  type="primary"
                  icon={useRenderIcon(LoadingIcon)}
                  circle
                  onClick={readSelValue}
                />
              </el-tooltip>
              <el-tooltip
                content="Submit all changes (Shift + W)"
                placement="top"
              >
                <el-button
                  type="primary"
                  icon={useRenderIcon(SubmitIcon)}
                  circle
                  onClick={batchWriteValue}
                />
              </el-tooltip>
            </div>
          </div>
        </>
      ),
      cellRenderer: ({ row, index }) => {
        const isValidValue = (value: string) => {
          // 示例逻辑：检查 value 是否为非空字符串且不是某个特定值
          return typeof value === "string" && value !== "some-invalid-value";
        };
        return isValidValue(row.value) ? (
          <div className="register-input">
            <iconify-icon-offline
              v-show={row.value !== "" && row.value !== row.srcValue}
              width="2em"
              height="2em"
              icon={DotIcon}
              style="color: var(--el-color-primary)"
            />
            0x
            {
              <div>
                <span
                  v-show={!showInput.value(index) && row.value === row.srcValue}
                  onClick={() => onClickValue(index)}
                  className="register-input-text"
                >
                  {row.value}
                </span>
                <span
                  v-show={!showInput.value(index) && row.value !== row.srcValue}
                  onClick={() => onClickValue(index)}
                  className="register-input-text"
                >
                  {row.value}
                </span>
                <el-input
                  v-show={showInput.value(index)}
                  id={"inputs" + index}
                  maxlength="8"
                  onBlur={() => onBlurValue(index)}
                  style="width: 65px; height: 28px"
                  v-model={row.value}
                />
              </div>
            }
          </div>
        ) : (
          <></>
        );
      }
    }
  ];

  const childrenTableColumns: TableColumnList = [
    {
      label: "Field",
      cellRenderer: ({ row }) => (
        <>
          <div style="display: flex; align-items: center">
            <el-tooltip content={row.desc} placement="top">
              <iconify-icon-offline icon={Information} />
            </el-tooltip>
            {row.field}
          </div>
        </>
      )
    },
    {
      label: "Bit",
      prop: "bit",
      width: 150
    },
    {
      label: "Attribute",
      prop: "attribute",
      width: 150
    },
    {
      label: "Reset",
      width: 150,
      cellRenderer: ({ row }) => <>0x{row.reset}</>
    },
    {
      label: "Value",
      prop: "value",
      fixed: "right",
      width: 250,
      cellRenderer: ({ row, index }) => {
        const isValidValue = (value: string) => {
          // 示例逻辑：检查 value 是否为非空字符串且不是某个特定值
          return typeof value === "string" && value !== "some-invalid-value";
        };
        return isValidValue(row.value) ? (
          <div className="register-input">
            <iconify-icon-offline
              v-show={showChildDot.value(row.parentIndex, index)}
              width="2em"
              height="2em"
              icon={DotIcon}
              style="color: var(--el-color-primary)"
            />
            0x
            {
              <div>
                <span
                  v-show={!showChildInput.value(row.parentIndex, index)}
                  onClick={() => onClickChildValue(row.parentIndex, index)}
                  className="register-input-text"
                >
                  {row.value}
                </span>
                <el-input
                  v-show={showChildInput.value(row.parentIndex, index)}
                  id={"inputs" + row.parentIndex + "_" + index}
                  maxlength={row.valueMaxLength}
                  onBlur={() => onBlurChildValue(row.parentIndex, index)}
                  style="width: 65px; height: 28px"
                  v-model={row.value}
                />
              </div>
            }
          </div>
        ) : (
          <></>
        );
      }
    }
  ];
```
![img/duty.jpg](img/duty.jpg)
```vue
<template>
  <el-table
    :border="true"
    row-key="rowKey"
    :expand-row-keys="expands"
    ref="multipleTable"
    :data="tableData"
    :height="tableHeight"
    tooltip-effect="dark"
    style="width: 100%;border: none;"
    :default-sort="sort"
    :row-class-name="handleRowClass"
    @select="handleCheckSel"
    @keydown.native="handleKeyDown"
    @mousedown.native="handleMouseDown"
    @selection-change="handleSelectionChange"
    @sort-change="handleSortChange"
    @cell-click="handleCellClick"
    @cell-mouse-enter="handleCellHover"
    @cell-mouse-leave="handleCellLeave"
    @row-contextmenu="handleContextmenu"
    @expand-change="handleExpandChange">
    <el-table-column
      type="selection"
      :reserve-selection="true"
      class-name="sel-check"
      width="65">
    </el-table-column>
    <el-table-column type="expand" width="35" class-name="expand-icon">
      <template slot-scope="props">
        <table style="width: 100%;">
          <thead>
          <tr>
            <th>Field</th>
            <th width="63">Bit</th>
            <th width="88">Attribute</th>
            <th width="88">Reset</th>
            <th width="88">Value</th>
          </tr>
          </thead>
          <tbody>
          <tr :class="{readonly:fields[3] == 'RO'}" v-for="fields in getFieldList(props.row)">
            <td>
              <el-tooltip v-if="fields[5]" placement="bottom-start" effect="light">
                <div slot="content" v-html="'&nbsp;&nbsp;' + fields[5].replace(/\n/g,'<br/>&nbsp;')"></div>
                <i class="el-icon-info"></i>
              </el-tooltip>
              {{fields[0]}}
            </td>
            <td>{{(fields[1] + fields[2] - 1) + ':' + fields[1]}}</td>
            <td>{{fields[3]}}</td>
            <td>{{'0x' + fields[4].toString(16)}}</td>
            <td width="120" style="padding:0;" @mouseenter="handleFieldHover" @mouseleave="handleFieldLeave"><div class="cell" v-if="props.row.value"><div class="r-value"><span><span v-if="isShowFieldEditMark(props.row, fields[1], fields[2])" class="mark-edit"></span>0x</span><span class="value-label" v-if="fields[3] == 'RO'" v-html="subROFromHex(props.row, fields[1], fields[2])"></span><span class="value-label" v-else v-html="subFromHex(props.row, fields[1], fields[2])"></span><el-input
              v-if="fields[3] !== 'RO'"
              type="text"
              size="mini"
              style="display: none;"
              :maxlength="Math.ceil(fields[2]/4)"
              :value="subFromHex(props.row, fields[1], fields[2])"
              @blur="handleFieldValueBlur(props.row, fields[0], fields[1], fields[2])"
              @focus="handleFieldFocus()"></el-input></div></div></td>
          </tr>
          </tbody>
        </table>
      </template>
    </el-table-column>
    <el-table-column
      prop="name"
      label="Name"
      min-width="300"
      sortable="custom"
      class-name="expand-td">
    </el-table-column>
    <el-table-column
      prop="addr"
      label="Address"
      sortable="custom"
      :formatter="addrFormatter"
      class-name="hex-value"
      min-width="110">
    </el-table-column>
    <el-table-column
      prop="value">
      <template slot-scope="scope">
        <div class="r-value" v-if="scope.row.value || cacheValue[scope.row.addr]">
          <span><span v-if="cacheEdit[scope.row.addr]" class="mark-edit"></span>0x</span><span class="value-label" v-html="getColoredValue(scope.row)"></span>
          <el-input v-model="scope.row.value" type="text" style="display: none" maxlength="8" @blur="handleValueBlur(scope.row)" />
        </div>
      </template>
      <template slot="header" slot-scope="scope">
        <span>Value</span>
      </template>
    </el-table-column>
  </el-table>
  <el-pagination
    class="reg-pagination"
    @size-change="handleSizeChange"
    @current-change="handleCurrentChange"
    :current-page="currentPage"
    :page-sizes="[10, 20, 50, 100, 200, 500, 1000]"
    :page-size="pageSize"
    background
    layout="total, sizes, prev, pager, next"
    :total="totalSize">
  </el-pagination>
  <el-select
    v-show="activeName === 'SSM'"
    v-model="ssmMethod"
    title="Access Method"
    size="mini"
    class="method-sel">
    <el-option
      v-for="item in options"
      :key="item.value"
      :label="item.label"
      :value="item.value">
    </el-option>
  </el-select>
</template>
```
![img/hushuobadao.png](img/hushuobadao.png)
