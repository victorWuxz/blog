# 背景
我们组三年前开发了一套NoCode平台命名为米鹿平台，旨在提升运营生产力，目前这个系统的用户数一直维持在一个相对较高的水准，并且有越来越多的新用户加入，所以对该平台开放的能力要求就变高了许多。经过多次会议后，我们要将这个平台改造一下，可以接受外部组件，任何业务方的开发人员都可以给我们平台提供自定义组件，这样可以进一步提升开发和运营效率，充分发挥这个平台的作用。由于即将接入我们平台的多个业务方已经拥抱Vue3，所以我们决定将米鹿平台升级到Vue3。

# 项目介绍
米鹿是一个拖拽搭建H5的工具，项目的生产端前端使用的vue-cli搭建而成，拖拽过程中的状态都存储在Vuex中，通过Vuex实现了了一套状态机模型，路由使用的是Vue-router，UI框架使用的element-ui，除此之外，还引入了很多基于vue2开发的的第三方组件。

# 升级步骤

## 新建一个仓库
米鹿系统已经维护了三年多，内置组件上百个，直接在原项目升级风险较高，所以一定要新建一个仓库，将原仓库中的功能逐渐迁移过来。

## 使用最新版本的Vue-cli生成新项目
这里大家可能会有疑问，为什么还是使用Vue-cli而不用Vite。因为我们的项目基于原有的vue-cli做了很多webpack配置，而且在打包时依赖vue-cli打包lib的功能，相对于vue-cli升级，直接替换为vite成本较高。

在这里直接使用vue-cli生成一个vue3项目，然后迁移老项目的一些工程化配置（eslint，gitHooks等）即可。

## 分析原项目的package.json，逐渐迁移依赖
首先我们要分析一下老项目中的打包依赖，比如babel，webapck相关的loader等，因为vue-cli升级后对应的webpack版本已经升级是5.x，一些想对应的loader也需要升级，不兼容的配置也需要处理一下，保证项目能够跑起来。

### 升级开发环境依赖（devDependencies）和项目配置

1.迁移并升级开发环境依赖。

2.根据[VUE-CLI升级文档](https://cli.vuejs.org/migrations/migrate-from-v4.html)修改vue.config.js配置，在这一步，我们主要是移除了一些不兼容webpack5的配置，例如devServer中的watchOptions和disableHostCheck，其中`disableHostCheck: true`要替换成`allowedHosts: 'all'`,还有其他不兼容的配置这里就不一一列举，大家去官网查看相关文档即可。

> 注意vue-cli中文文档已经好久不更新，需要查看英文文档进行升级

3.安装`@vue/compiler-sfc`，替换`vue-template-compiler`。

4.注释掉业务逻辑入口（后续逐渐打开），首先保证项目能跑起来。

### 修改Vue3不兼容的语法

根据文档[非兼容性变更](https://v3.cn.vuejs.org/guide/migration/introduction.html#%E9%9D%9E%E5%85%BC%E5%AE%B9%E7%9A%84%E5%8F%98%E6%9B%B4)修改即可，这里给大家列举一下此次升级遇到的一些常见的变更。

1.应用入口修改，Vue不能再通过import全局引入，Vue3全局API已经支持Treeshaking，且不再支持全局引入vue的方式，改动如下：

```js
import App from './App';
import router from './router';
import store from './store';
// 不再支持
// import Vue from 'vue';
// Vue3 导入方式
import { createApp } from 'vue';

// 不再支持
// new Vue({
//  路由和vuex的挂载方式也改变了
//  router,
//  store,
//  render: h => h(App),
//}).$mount('#app');

// 全局变量的挂载方式发生变化，不再支持
// Vue.prototype.$axios = httpClient;
// Vue.prototype.$isEditor = true;
// Vue.use(...)
// Vue.directive(...)

// VUE3
const app = createApp();
app.config.globalProperties.$axios = httpClient;
app.config.globalProperties.$isEditor = true;

// 挂载路由和vuex（vuex和vue-router也需要升级）
app.use(router);
app.use(store);
app.directive(...)
``` 

2.全局变量的修改，vue2我们可以通过`this.$nextTick`或者`this.$createElement`等全局api，Vue3需要单独引入使用。

```js
import { nextTick, h } from 'vue';

// nextTick(...)
// h(...)
```

3.slot插槽变更，3.x版本统一了普通插槽和作用域插槽。
。

- 在 3.x 中，将所有 `this.$scopedSlots` 替换为 `this.$slots`。
- 将所有 `this.$slots.defalut` 替换为 `this.$slots.defalut()`。
- 现在已经无法通过插槽获取到具体的dom对象，如果有此应用场景需要替换成ref获取的方式。

4.模板的v-for需要在模板上指定key，不需要在子元素中指定;模板中的子元素不再要求唯一。

```js
<templete v-for="item in list" :key="item.key">
    <div>1</div>
    <div>2</div>
</templete>
```

5.`v-bind:param.sync`替换成`v-model:param`。Vue3移除了`v-bind.sync`的语法，由v-model替换

6.组件生命周期变化

```js
// destroyed() {}
unmounted() {}

// beforeDestroy() {}
beforeUnmount() {}
```

7.移除了部分api。比如`$set`和`$destroy`等方法。vue2中不能监听对象属性的增加和删除，必须使用`$set`或者`Vue.set`来变更对象或者数组，Vue3中直接用原生语法改变对象数组即可，不需要再使用`$set`。

### 升级开发依赖（dependencies）

分析开发依赖，前面已经介绍过，此项目使用的是Vue全家桶+element-ui，所以vue全家桶和element-ui都需要升级，除此以外项目中有很多第三方组件都是基于vue2开发的，我们先找出这些组件标记出来，对应的代码入口先注释掉，后续一一替换成vue3版本的组件即可。

> 此部分是最费时间的一部分，虽然不难但比较繁琐且容易出bug，需要谨慎测试。

1.升级Vue-router。vue-router的升级改动并不大，此次升级只改动了定义和引入路由的方式，参照官方升级文档[从 Vue2 迁移 | Vue Router (vuejs.org)](https://router.vuejs.org/zh/guide/migration/index.html)升级即可。

2.升级Vuex。Vuex的改动也很小，此次升级只改动了定义store和引入store的方式，其余没有任何变化。参照官方升级文档[从 3.x 迁移到 4.0 | Vuex (vuejs.org)](https://vuex.vuejs.org/zh/guide/migrating-to-4-0-from-3-x.html)升级即可

3.升级element-ui为element-plus。此部分升级工作较为繁琐，可参考文档[Element Plus 不兼容变化（中文简体） · Discussion #5657 · element-plus/element-plus · GitHub](https://github.com/element-plus/element-plus/discussions/5657)进行升级。我们项目的具体改动如下：

- 删除emlemt-ui，引入element-plus，并替换项目中的import语句。
- 如果项目是全局引入，讲`Vue.use(Element)`替换为`app.use(Element)`,并且替换全局引入的样式和变量文件。
- 如果项目是按需引入，请参考[安装 | Element Plus (element-plus.org)](https://element-plus.org/zh-CN/guide/installation.html)文档修改。
- 按需引入有一定的bug，比如项目中如果有`v-loading`等指令就会报错，`this.$message()`等方法也会出现问题，修改方法是全局引入`ElLoading`、`ElMessage`、`ElMessageBox`,具体用法如下：

    ```js
    import { ElLoading, ElMessage, ElMessageBox } from 'element-plus';
    import 'element-plus/theme-chalk/el-loading.css';
    import 'element-plus/es/components/message/style/css';
    import 'element-plus/es/components/message-box/style/css';

    //...
    const app = createApp(App);
    app.use(ElMessage);
    app.use(ElMessageBox);
    app.use(ElLoading);
    ```
    同时修改vue.config.js
    ```js
    // ...
    configureWebpack: {
        plugins: [
          autoImport({
            resolvers: [elementPlusResolver({
              importStyle: 'css',
              exclude: new RegExp(/^(?!.*loading-directive).*$/),
            })],
          }),
          components({
            resolvers: [elementPlusResolver()],
          }),
        ],
    },
    // ...
    ```
- 修改各种情况下Icon的用法。element-plus已经将icon单独抽离出来，需要按需引入，所以Icon的改动量巨大，以前用`<i class="el-icon-edit" />`这种用法和`<el-input :pre-fix="el-icon-search" />`已经不起作用，都需要替换成对应的icon组件，示例如下：

    ```js
    <script setup lang="ts">
    import { Search } from '@element-plus/icons'
    </script>

    <template>
      <el-button type="primary" :icon="Search">Search</el-button>
    </template>
    ```
- 所有带有slot的组件修改，不再支持`slot="header`等语法，例如`ElCard`和`ElDialog`
    ```js
    <template>
        <el-card>
            <template #header>
                header
            </template>
        </el-card>
    </template>
    ```
4.升级其他第三方组件。第三方组件的升级依赖其官方是否支持Vue3，如果不支持，需要寻找替代品或者自己造轮子。这里简单列举一下我们升级遇到到一些第三方组件。

- `vue-awesome-swiper`替换为`swiper/vue`，新版本的swiper改为插件模式，每个模块都需要按需引入，具体改动可参照[Swiper Vue.js Components (swiperjs.com)](https://swiperjs.com/vue)。
- `vue-cropper`升级到1.x版本，参考官网进行修改。
- `vue-json-editor`替换为`vue3-json-editor`，参考官网进行修改。
- `vuedraggable`升级到4.x版本，参考官网进行修改。
- `qrcode.vue`升级到3.x版本，参考官网进行修改（升级后样式会发生一定变化，需要留意）。
- `vue-wechat-title`不支持vue3，查看其源码很简单，参考源码维护一个Vue3版本的即可。
- `handsontable`Vue3版本不支持免费使用，我们替换成了`ag-grid-vue3`并变更了一下代码逻辑。

5.升级团队自己维护的vue组件。团队内部维护的组件直接去相对应的代码仓库修改即可，要注意的是一定要升级一个大版本，保留vue2的版本，此次升级我们也升级了接近10个内部组件。


# 踩坑记录

1. 通过以CDN的方式引入Vue3的umd版本，有些组件的语法在模板中不被编译，查找了各种文档也没有找到原因，所以我们的解决方案是不通过CDN的方式引入Vue3，因为Vue3已经支持treeShranking，打包体积已经缩小很多，没有必要再通过CDN的方式引入。

2. Mixin混入的data中的变量变为浅拷贝，如果用到mixin，得额外注意这一点。

3. 自己维护的组件，经过webpack打包后的组件引入项目中不能被正确编译，且样式丢失。和vue-loader和vue升级相关，具体原因还在查找，因为时间较紧，我们将组件改为通过vue-cli打包后可正常编译。

# 总结

此次升级大概用时14人天，主要耗时的工作还是在解决非兼容性语法和第三方组件升级，此外，测试工作是非常重要的，升级之后要进行一次全量的测试，以下是这次升级的一点总结：

- 升级时先升级脚手架，注释业务逻辑，之后逐渐放开，有限保证项目可以运行起来。
- 大部分API在Vue3中仍然可用,只有部分不兼容的变更，参照官方文档即可。
- 第三方组件的升级最为繁琐，依赖第三方组件是否支持Vue3，不支持就需要自己造轮子。
- 新语法如`setup`在接下来写新组件的时候再尝试使用，老组件在功能升级是可顺带修改，以保证项目代码风格统一。
- 测试工作十分重要，升级过程中可以逐步开始测试。