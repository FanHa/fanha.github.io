### table 源码目录
```
src
  dropdown.js
  filter-panel.vue
  layout-observer.js
  table-body.js
  table-column.js
  table-footer.js
  table-header.js
  table-layout.js
  table-store.js
  table.vue.js
  util.js

index.js
```
### Element 官网示例中的最基础用法源码
`template`中主要的部分是`data` 和 `el-table`的`slot`中的一系列`el-table-column`
```html
  <template>
    <el-table
      :data="tableData"
      style="width: 100%">
      <el-table-column
        prop="date"
        label="日期"
        width="180">
      </el-table-column>
      <el-table-column
        prop="name"
        label="姓名"
        width="180">
      </el-table-column>
      <el-table-column
        prop="address"
        label="地址">
      </el-table-column>
    </el-table>
  </template>

  <script>
    export default {
      data() {
        return {
          tableData: [{
            date: '2016-05-02',
            name: '王小虎',
            address: '上海市普陀区金沙江路 1518 弄'
          }, {
            date: '2016-05-04',
            name: '王小虎',
            address: '上海市普陀区金沙江路 1517 弄'
          }, {
            date: '2016-05-01',
            name: '王小虎',
            address: '上海市普陀区金沙江路 1519 弄'
          }, {
            date: '2016-05-03',
            name: '王小虎',
            address: '上海市普陀区金沙江路 1516 弄'
          }]
        }
      }
    }
  </script>
```

### table源码

#### table 源码的template中主要的几个部分
1. 用于放`table-column`的`slot`,从class名字“hidden-columns”可看出这一部分只是为了传数据(列)的显示参数,而不是真正的显示.
2. 用于显示表头的 `table-header`,这是table里的另一个子组件.
3. 用于显示数据的`table-body`, 这是table里的子组件.
```html
<template>
  <div class="el-table">

    // slot
    <div class="hidden-columns" ref="hiddenColumns"><slot></slot></div>

    // header
    <div
      v-if="showHeader"
      v-mousewheel="handleHeaderFooterMousewheel"
      class="el-table__header-wrapper"
      ref="headerWrapper">
      <table-header
        ref="tableHeader"
        :store="store"
        :border="border"
        :default-sort="defaultSort"
        :style="{
          width: layout.bodyWidth ? layout.bodyWidth + 'px' : ''
        }">
      </table-header>
    </div>

    // body
    <div
      class="el-table__body-wrapper"
      ref="bodyWrapper"
      :class="[layout.scrollX ? `is-scrolling-${scrollPosition}` : 'is-scrolling-none']"
      :style="[bodyHeight]">
      <table-body
        :context="context"
        :store="store"
        :stripe="stripe"
        :row-class-name="rowClassName"
        :row-style="rowStyle"
        :highlight="highlightCurrentRow"
        :style="{
           width: bodyWidth
        }">
      </table-body>
      <div
        v-if="!data || data.length === 0"
        class="el-table__empty-block"
        ref="emptyBlock"
        :style="{
          width: bodyWidth
        }">
        <span class="el-table__empty-text">
          <slot name="empty">{{ emptyText || t('el.table.emptyText') }}</slot>
        </span>
      </div>
      <div
        v-if="$slots.append"
        class="el-table__append-wrapper"
        ref="appendWrapper">
        <slot name="append"></slot>
      </div>
    </div>
  </div>
</template>
```

#### table源码中script的主要部分

```js
export default {
    name: 'ElTable',

    props: {
      data: {
        type: Array,
        default: function() {
          return [];
        }
      },
      // ...
    },

    created() {
      this.tableId = 'el-table_' + tableIdSeed++;
      this.debouncedUpdateLayout = debounce(50, () => this.doLayout());
    },
    
    watch: {
      data: {
        immediate: true,
        handler(value) {
          this.store.commit('setData', value);
          if (this.$ready) {
            this.$nextTick(() => {
              this.doLayout();
            });
          }
        }
      },
      // ...
    },

    mounted() {
      this.bindEvents();
      this.store.updateColumns();
      this.doLayout();

      this.resizeState = {
        width: this.$el.offsetWidth,
        height: this.$el.offsetHeight
      };

      // init filters
      this.store.states.columns.forEach(column => {
        if (column.filteredValue && column.filteredValue.length) {
          this.store.commit('filterChange', {
            column,
            values: column.filteredValue,
            silent: true
          });
        }
      });

      this.$ready = true;
    },

    data() {
      const store = new TableStore(this, {
        rowKey: this.rowKey,
        defaultExpandAll: this.defaultExpandAll,
        selectOnIndeterminate: this.selectOnIndeterminate
      });
      const layout = new TableLayout({
        store,
        table: this,
        fit: this.fit,
        showHeader: this.showHeader
      });
      return {
        layout,
        store,
        // ...
      };
    }
```

#### 创建store 和 layout 组件
table组件创建时会先创建两个子组件,`TableStore`和 `TableLayout`,并放在data中的 `store` 和 `layout`变量中
```js
data() {
  const store = new TableStore(this, {
    rowKey: this.rowKey,
    defaultExpandAll: this.defaultExpandAll,
    selectOnIndeterminate: this.selectOnIndeterminate
  });
  const layout = new TableLayout({
    store,
    table: this,
    fit: this.fit,
    showHeader: this.showHeader
  });
  return {
    layout,
    store,
    // ...
  };
}
```

#### 初始化prop中的`data`
在watch中监测了data的变化,会触发data的handler,这里会调用store组件的commit函数,将table组件接受到的数据(即`:data`)初始化并按一定格式存好,但是因为`this.$ready`要在mounted阶段才被置为`true`,这里暂时不会执行`this.doLayout()`.  
这里的`$ready`是用于异步加载数据或者刷新数据的需要
```js
watch: {
  data: {
    immediate: true,
    handler(value) {
      this.store.commit('setData', value);
      if (this.$ready) {
        this.$nextTick(() => {
          this.doLayout();
        });
      }
    }
  },
}
```
#### setData

`store.commit('setData',...)`实际时封装调用的store中的setData函数
```js
// table-store.js
TableStore.prototype.commit = function(name, ...args) {
  const mutations = this.mutations;
  if (mutations[name]) {
    mutations[name].apply(this, [this.states].concat(args));
  } else {
    throw new Error(`Action not found: ${name}`);
  }
};

// ...
TableStore.prototype.mutations = {
  setData(states, data) {
    // 这里主要是根据数据表的各种选项(如排序,过滤等)对data里的数据进行清洗裁剪,然后在nextTick里回调table.updateScrollY()

    Vue.nextTick(() => this.table.updateScrollY());
  },
  // ...
}
```

table 的updateScrollY 调用了layout里的函数`updateScrollY` 和 `updateColumnsWidth`
>> 这里的函数都没有直接操作DOM,只是给要展示的数据提供格式,真正的展示比如表的宽,高的改变,数据的改变,势在nextTick时由`template`里的`table-header`和`table-body`根据改变好的数据及其格式调整表格的样子
```js
//table.vue
updateScrollY() {
  this.layout.updateScrollY();
  this.layout.updateColumnsWidth();
},
```

#### Do Layout
vue模板的created阶段,调用layout的初始化
```js
created() {
  this.tableId = 'el-table_' + tableIdSeed++;
  this.debouncedUpdateLayout = debounce(50, () => this.doLayout());
},
// ....
doLayout() {
  this.layout.updateColumnsWidth();
  if (this.shouldUpdateHeight) {
    this.layout.updateElsHeight();
  }
},
```

#### Mounted
在Mounted阶段,页面的DOM已经生成,可以对页面的原生事件绑定等操作,并置`$ready`属性为`true`
```js
mounted() {
  this.bindEvents();
  this.store.updateColumns();
  this.doLayout();
  this.resizeState = {
    width: this.$el.offsetWidth,
    height: this.$el.offsetHeight
  };
  // init filters
  this.store.states.columns.forEach(column => {
    if (column.filteredValue && column.filteredValue.length) {
      this.store.commit('filterChange', {
        column,
        values: column.filteredValue,
        silent: true
      });
    }
  });
  this.$ready = true;
},
```


