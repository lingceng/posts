### OPTIMIZE: 进入详情页返回没有保持原来的页面状态
TLDR: 使用 vue-navigation 可以实现功能。

方法一：使用 keep-alive 然后使用路由守卫刷新数据 #fix_expert_tab_alive_route
但这样需要对页面内的所有需要加载数据的组件都主动调用刷新
不够通用，需要获取数据，需要刷新页面内所有组件的状态
beforeRouteEnter 执行较早，但里面的next会在mounted之后才执行
https://segmentfault.com/a/1190000008923105

    watch: {
      expertId: 'getExpert',
    },
    beforeRouteEnter (to, from, next) {
      next(vm => {
        vm.expertId = to.query.id
      })
    }

方法二：使用集中的状态管理，保持Tab的选中状态
可能会有其他状态，例如滚动等，需要获取数据，这个方案不够通用。

方法三：使用 keep-alive 然后配合目标路由判断是否缓存 #fix_expert_tab_mata_keep
默认 keep-alive 为 true，当跳转到非子页面时就设置为 false，跳转到子页面设置为 true 。
发现缓存的那个页面不会刷新，从其他子页面跳转回来还是相同内容

    beforeRouteLeave(to, from, next) {
      from.meta.keepAlive = (to.name === 'Material')
      next();
    }

方法四：使用 keep-alive 的include #fix_expert_tab_with_alive_include
需要使用 vuex 管理缓存组件名字，还需要显示地给组件设置名称
https://juejin.im/post/5b407c2a6fb9a04fa91bcf0d

但发现通过路由的方向不能准确判断前进或后退。
例如，如果直接从 C 跳回 A，然后 A 再进入 B1，会显示到 B

    beforeRouteLeave (to, from, next) {
      if (to.name === 'Material') {
        this.$store.commit('alive/keepAlive', from.name)
      } else {
        this.$store.commit('alive/noKeepAlive', from.name)
      }
      next()
    }

尝试通过保持上一次路由，对比fullPath来判断是否是回退操作。
非回退操作时需要先清除掉缓存再缓存新内容，需要研究keep-alive组件主动清除缓存的机制。

通过 vue keep-alive destroy cache 找到主动清除缓存的方式
https://github.com/vuejs/vue/issues/6509

清除需要获取到Vue实例的引用，所以需要在 beforeRouteLeave 中把实例和from路由加入长度为2的队列。
从队列取出元素，如果是，全路径相同则不刷新缓存，全路径不同则刷新。

测试用例:

    回退场景   | B - C - B      | OK
    同组件跳转 | B - B1 - B     | 不支持，因为不会触发 beforeRouteLeave
    间隔同组件 | B - C - D - B1 | OK


    beforeRouteLeave (to, from, next) {
      store.commit('alive/pushRouterQuue', { router: from, instance: this })
      next()
    }

    paths.forEach((path) => {
      if (path.keepAlive === true || path.keepAlive === 'OnBack') {
        store.commit('alive/keepAlive', path.name)
      }
    })

    router.beforeEach((to, from, next) => {
      let preTwoPage = store.state.alive.routerQueue[1]
      let isOnBack = preTwoPage && preTwoPage.router.meta.keepAlive === 'OnBack'
      let isBack = preTwoPage && preTwoPage.router.fullPath === to.fullPath
      if (isOnBack && !isBack) {
        clearAliveCache(preTwoPage.instance)
      }

      next()
    })

同组件跳转的场景只会触发 beforeRouteUpdate，不管是否 keep-alive。
感觉 vue-router 和 vue 配合不太紧密，Router只负责把路由导航到某个组件
Vue负责呈现出这个组件，URL变化刷新数据需要用watch机制，感觉不够直观

#### 使用keep-alive实现回退不刷新总结
回退不刷新的功能上似乎很简单，Turbolink通过页面副本的方式实现，当回退时就访问副本。
Vue中类似的组件是 keep-alive，keep-alive 能使*组件保持活跃*，但不是对*页面做副本*。
Keep-alive 通过 include 等方式在一次渲染中能选择保持或者不保持组件的活跃。

使用 keep-alive 来实现回退不刷新需要这样处理。
+ 要每次进入都要缓存组件，为回退不刷新做准备。
+ 但在不是回退的场景，就需要刷新数据，并再次缓存。
第二点 keep-alive 默认是比较难实现的，它只能选择刷新或者缓存，不能刷新之后再缓存。

网上有通过 beforeRouteLeave 的路由方向来预测会不会有回退来设置活跃。
但预测很难稳定，在页面互相跳转且有参数变化时很容易出错，出现页面一直不刷新的情况。
是不是回退操作需要真的发生的时候才好判断，而发生的时候keep-alive不能做到先刷新再缓存。

我们不能通过减少再增加 include 来达到刷新的效果，和 keep-alive 使用 watch 的实现有关。
Vue的 watch 需要一次周期才会生效的，不会每一次修改都成效。
[Vue watch原理](https://juejin.im/post/5d629380518825121f661973)

所以比较直观的方式是提供一个主动刷新 keep-alive 的方式，让其进入之后再次缓存。
这里总结了方法，需要拿到页面组件的实例
https://gist.github.com/lingceng/c204bac2704d03db99932d5457e662e6

#### Related Info
The keep-alive 的组件有 activated 和 deactivated 方法，可以用来更新数据
[Vue keep-alive深入理解及实践总结](https://juejin.im/post/5d5a534351882568916523b7)

Vue Router的HTML5 History模式使用了 history.pushState API
进入子页面，当前页面会被销毁，从子页面返回时页面对象被重新创建
https://developer.mozilla.org/en-US/docs/Web/API/History_API/Working_with_the_History_API

Here talks some usage and introduct pjax and Histroy JS lib.
Turbolink is a more automatic solution than pjax.
https://css-tricks.com/using-the-html5-history-api/

Turbolink 缓存了页面使返回时可以快速响应
https://github.com/turbolinks/turbolinks#understanding-caching

#### 相关开源项目
vue-navigation 却少测试，有较多issue，没有解释原理 [USED]
https://github.com/zack24q/vue-navigation

Navigation.js 组件默认会缓存所有页面，通过增加 VNK 的参数来唯一标记页面
似乎也是用的 keep-alive 的特性来实现，代码中有 vnode.data.keepAlive = true

The index.js 中通过 router.beforeEach 通过记录历史路由，判断是否是回退等操作
有 VNK 参数的新路由则是回退操作，没有则是前进或替换
The navigator.js 回退操作怎会清除栈顶路由，下次进入就需要重新加载

会在浏览器地址增加 VNK 的参数来判断前进或后退
在导航标签页间切换，会造成多次前进，占用内存
可以通过清除的方式处理，或者对标签页做 keep-alive 并不放入 Naviation.
https://github.com/zack24q/vue-navigation/issues/8

开源项目，刚发布不久，项目的一篇文章很有益处
https://github.com/kallsave/vue-router-cache
https://juejin.im/post/5dccdb4a51882510ce752164

https://github.com/hezhongfeng/vue-page-stack
https://github.com/nearspears/vue-nav

如何判断使用了回退操作呢？
记录历史路由记录，当前页面是记录的页面栈的第二个则为回退
https://juejin.im/post/5dccdb4a51882510ce752164
https://github.com/zack24q/vue-navigation/issues/8

