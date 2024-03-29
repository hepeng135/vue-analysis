Vue的核心插件Vuex，用来管理应用中的状态，下面我们将一起来看看Vuex的源码，并对一些常用的API进行讲解。

## 目录
Vuex的源码被托管在github上，我们首先通过git将代码clone下来，用心仪的IDE打开，文件目录如下。

![](./image/src.png)

如图所示，Vuex的主要代码都在src文件夹下，examples文件夹下放一些栗子，解析源码时，我们运行栗子进行断点调试进行跟进会更容易理解，首先npm install安装一下依赖包
在运行前我们需要修改一个地方，首先先找到目录中的package.json文件，注意scripts中的第一项 "dev": "node examples/server.js" ，我们运行
npm run dev 对应的就好走这行,既会自动 node examples/server.js 命令，node会启动example下的server.js,我们打开examples文件，看到一些栗子、
server.js 和 webpack.config.js文件，在server文件我们会启动一个node的本地服务器，在里面会用webpack进行打包，对应的webpack配置文件就是 webpack.config.js，
在这个配置文件中我们加上 devtool: 'source-map'，使得我们在打断点时可以打在源src文件上，不用在编译后的代码中进行调试。

## 源码解析

#### 从入口开始
```
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'
import createLogger from './plugins/logger'

export default {
    Store,
    install,
    version: '__VERSION__',
    mapState,
    mapMutations,
    mapGetters,
    mapActions,
    createNamespacedHelpers,
    createLogger
}

export {
    Store,
    install,
    mapState,
    mapMutations,
    mapGetters,
    mapActions,
    createNamespacedHelpers,
    createLogger
}

```
如上述代码所示，我们一目了然的看到Vuex对外暴露的API，在入口我们先导入主要Store，install，然后就是一些辅助方法，然后在将其用两种方法导出。下面我们先主要
分析一下Store和install

#### 安装Vuex插件的Install方法
在Vue安装插件需要调用Vue.use(plugin)来安装对应插件，在Vue中use的实现其实就是调用当前插件身上的install方法进行安装，
但是在Store对象上并没有install方法，那是怎么实现的，在store.js文件中我们需要注意如下代码。
```
//store.js文件中的部分代码
import applyMixin from './mixin'
//code...
let Vue
export class Store {
    constructor (options = {}) {
    
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        //code. ...
    }
}

```
在store.js中先设置一个全局变量Vue，在Store构造函数中判断当前是否拥有全局Vue，并且当前环境拥有全局引用Vue，也就是window上有Vue这个对象，
然后执行install，对全局变量Vue的判断主要保证install只执行一次，然后我们看看install方法
```
export function install (_Vue) {
  if (Vue && _Vue === Vue) {
    if (__DEV__) {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  Vue = _Vue
  // applyMixin 在2.0版本中：每个组件混入beforeCreate构造函数，在每个组件实例上挂载$store属性，使得在每个组件实例中我们都能访问$store  既this.$store
  applyMixin(Vue)
}

//mixin.js文件
//省略版本控制部分
export default function(Vue){
    Vue.mixin({ beforeCreate: vuexInit });

    function vuexInit(){
        const options = this.$options
        // store injection
        if (options.store) {
            this.$store = typeof options.store === 'function'
                ? options.store()
                : options.store
        } else if (options.parent && options.parent.$store) {
            this.$store = options.parent.$store
        }
    }
}
```
install调用时我们比较一下当前的Vue和window.Vue,确保install只执行一次，在将window.Vue赋值给Vue,最后调用applyMixin，
在applyMixin函数中我们用Vue.mixin混入beforeCreate钩子函数，该钩子函数确保我们在Vue的任何组件上我们都可以通过thus.$store
来访问当前的Store实例。

#### 认识Store构造函数
我们在使用Vuex的时候，通常需要传入一个对象去实例化Store类，这个对象包含state、mutations、actions、getters、modules,那么在实例化的过程中
到底做过什么，让我们来来看看下面的代码
```
export class Store {
    constructor (options = {}) {
        if (!Vue && typeof window !== 'undefined' && window.Vue) {
            install(window.Vue)
        }
        if (__DEV__) {
            assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
            assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
            assert(this instanceof Store, `store must be called with the new operator.`)
        }
        const {
            plugins = [],
            strict = false
        } = options

        this._committing = false
        this._actions = Object.create(null)
        this._actionSubscribers = []
        this._mutations = Object.create(null)
        this._wrappedGetters = Object.create(null)
        this._modules = new ModuleCollection(options)
        this._modulesNamespaceMap = Object.create(null)
        this._subscribers = []
        
        this._watcherVM = new Vue()
        this._makeLocalGettersCache = Object.create(null)
        
        // bind commit and dispatch to self
        const store = this
        const { dispatch, commit } = this
        this.dispatch = function boundDispatch (type, payload) {
            return dispatch.call(store, type, payload)
        }
        this.commit = function boundCommit (type, payload, options) {
            return commit.call(store, type, payload, options)
        }
        this.strict = strict
        const state = this._modules.root.state
       
        installModule(this, state, [], this._modules.root)
        resetStoreVM(this, state)
        plugins.forEach(plugin => plugin(this))
        const useDevtools = options.devtools !== undefined ? options.devtools : Vue.config.devtools
        if (useDevtools) {
            devtoolPlugin(this)
        }
    }
}
```
首先是我们之前说到的install函数的调动，然后在开发环境下Vuex进行了一些断言判断操作。

断言：必须保证当前Vue的存在，因为后面的时候我们会依赖他
```
assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
```
断言：必须保证当前浏览器环境必须兼容Promise，否则需要提供一个Promise polyfill
```
assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
```
断言：当前对象必须是Store的实例对象。既通过new Store创建的。
```
assert(this instanceof Store, `store must be called with the new operator.`)
```
然后我们再看看assert函数是怎么实现的,代码如下，实现非常简单，通过判断传进来的第一个参数（不为真），然后抛出第二个参数（错误描述）
```
export function assert (condition, msg) {
    if (!condition) throw new Error(`[vuex] ${msg}`)
}
```
回到Store构造函数中，接下来将看到一些属性初始化代码
```
//记录当前state的改变是否通过commit
this._committing = false
//存放模板所有的action
this._actions = Object.create(null)
//存放通过API subscribeAction 添加的订阅store的action
this._actionSubscribers = []
//存放所有模板的mutation
this._mutations = Object.create(null)
//存放所有模板的getter
this._wrappedGetters = Object.create(null)
//处理Store中的模块
this._modules = new ModuleCollection(options)
//存放当前命名空间模块 eg: {a:module,b:module,''a/c':module}  这个时候模板a,b,a/c的namespaced都是true
this._modulesNamespaceMap = Object.create(null)
//存放所有通过API subscribe 添加的订阅store的mutation。
this._subscribers = []
// 创建vue实例
this._watcherVM = new Vue()
```
在属性初始化中，我们先中点介绍this._modules，他通过创建ModuleCollection类的实例进行初始化，传入的options和创建Store类的实例options
一样，接下来我们看看这个ModuleCollection类。

##### ModuleCollection类详解
```
export default class ModuleCollection {
    constructor(rawRootModule){
        this.register([], rawRootModule, false)
    }

    register(path, rawModule, runtime = true){
        if (__DEV__) {
            // 验证getter  mutations actions ,getter、mutations 必须是函数,actions可以是函数、对象(则：actions.handler 必须是函数
            assertRawModule(path, rawModule)
        }
        // 实例化Module类
        const newModule = new Module(rawModule, runtime)
        // 通过当前的path判断当前是根模块还是 子集模块
        if (path.length === 0) {
            this.root = newModule// 根模板
        } else { // 子模板
            // 获取父级模板
            const parent = this.get(path.slice(0, -1))
            // 添加子模板
            parent.addChild(path[path.length - 1], newModule)
        }
        
        // register nested modules // 子集模板
        // 循环子集模板，然后注册
        if (rawModule.modules) {
            forEachValue(rawModule.modules, (rawChildModule, key) => {
                this.register(path.concat(key), rawChildModule, runtime)
            })
        }
    }
}
```
在ModuleCollection类的构造函数中，我们直接调用它的原型方法register，那么我们就来看下这个register方法,首先做的也是断言验证，断言当前
对应的module中的getter、mutations必须是函数，actions可以是函数、对象(则：actions.handler 必须是函数）

```
  assertRawModule(path, rawModule)
```
在register中接下来是实例化Module，传入当前的module，如果当前的path，既module的key路径为空数组，则代表当前是根模板，在赋值到this.root上，
如果当前的path不为空的的话，则代表当前的模板存在子集，则调用this.get获取父级模板，然后父级模板实例调用其原型方法addChild添加这个子模板
```
// 实例化Module类
const newModule = new Module(rawModule, runtime)
// 通过当前的path判断当前是根模块还是 子集模块
if (path.length === 0) {
  this.root = newModule// 根模板
} else { // 子模板
  // 获取父级模板
  const parent = this.get(path.slice(0, -1))
  // 添加子模板
  parent.addChild(path[path.length - 1], newModule)
}
```
register方法是个递归函数，我们需要监测每个module的子级，然后实例，然后做添加操作，使其形成一个链式结构。我们通过判断当前
模板是否存在modules属性，既是否存在子级模板，如果存在则循环子模板，然后调用register。
```
// register nested modules // 子集模板
// 循环子集模板，然后注册
if (rawModule.modules) {
    forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
    })
}
```

综上所属在register方法函数中，我们主要是将传进来的module模块进行实例化，然后根据其子父级的关系创建一个链式结构，并赋值到this.root上，下面
我们看看在register函数中调用的Module类的具体代码
##### Module类的详情
```
export default calss Module {
    constructor (rawModule, runtime) {
        this.runtime = runtime  // 当前是初始化  还是运行时
        // 创建_children属性
        this._children = Object.create(null) // 创建一个纯对象
        //当前的module
        this._rawModule = rawModule  
        //获取出当前模块的state
        const rawState = rawModule.state // 存储当前的state ,未加工的state
        this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
    }
}
```
在Module类的构造函数中，主要设置了一个_children属性和获取当前module的state。回到register函数中，通过path去判断当前是否根模板，是的话直接挂
到this.root上，否则我们调用ModuleCollection类的原型方法get，获取当前module的上一个module，既当前模块的父模块，我们来看看这个get方法
```
class ModuleCollection {
    
    //code...
    get (path) {
         // reduce 进行循环，初始值为当前的根对象模板。循环的为当前模块的名称
         // 返回对应moduleKey的modules项
         console.log(path)
         return path.reduce((module, key) => {
            console.log(module)
            return module.getChild(key)
         }, this.root)
    }
    //code...
}
```
循环传进来的path，以this.root(根模块)做个起点，依次调用当前模板实例的原型方法getChild,this.root指向通过Module类创建的实例，那我们
现在看看Module类的原型方法getChild
```
class Module {
    //code...
    getChild (key) {
        return this._children[key]
    }
    //code...
}
```
getChild函数接收一个key作为参数，返回当前模板的子模块集合中的具体一个子模块。回到ModuleCollection类的register方法，调用get是传入的参数
为path.slice(0,-1),既去掉当前最后一个元素（当前模板的key）去取当前模板的父级模拟。然后在调用父模板的addChild方法添加子级模块为当前模块.
```
class Module {
    //code
    addChild (key , module){
        this._children[key]=module;
    }
    //code
}
```
回调最开始的Store类的构造函数中this._modules = new ModuleCollection(options)做的作用我们可以这样来理解，创建一个this.root执行根
模板(执行rootModule)，根模板拥有_children，里面放子模板，依次类推，形成一个链式结构。
```
    new Store({
        state,
        mutations,
        actions,
        modules:{
            a:ModuleA,
            b:ModuleB,
            c:moduleC
        }
    })
   //经过ModuleCollection实例化以后，
    this._modules.root={
        _rawModule:{state,mutations,actions},
        state:state,
        _children:{
            a:{},
            b:{},
            c:{}
        }
    }
```
接下来我们来看Store构造函数中对dispatch和commit的处理
```
const store = this
// 从当前实例对象上解析出dispatch commit
const { dispatch, commit } = this
// 重新定义dispatch、commit属性
this.dispatch = function boundDispatch (type, payload) {
  return dispatch.call(store, type, payload)
}
this.commit = function boundCommit (type, payload, options) {
  return commit.call(store, type, payload, options)
}

```
如上所示，从当前实例上解析出dispatch与commit，然后重新在当前实例上定义dispatch、commit方法，后续我们将介绍这两个方法的具体实现。
下面我们将接着介绍Store构造函数中剩下的代码
```
// strict mode当前Vue Store是否严格模式，默认为false
this.strict = strict
// 获取当前的rootState
const state = this._modules.root.state

// 注册commit actions getter
installModule(this, state, [], this._modules.root)

//初始化state和getter
resetStoreVM(this, state)

// apply plugins// 安装插件
plugins.forEach(plugin => plugin(this))
```
在我们处理完模板关系后，接下来就是处理模板中 state、commit、actions、getters，在上述代码中，我们首先将在当前实例对象上添加strict属性，表示当前
Store是否严格模式，然后从this._modules.root.state上获取根模板的state既rootState,然后就是处理模板中 state、commit、actions、getters。

注册处理 state、commit、 actions、 getter
```
class Store {
    //code...
    installModule(this, state, [], this._modules.root)
    //code...
}
//installModule函数的具体代码
/**
*   接受参数如下
*   store:当前Stroe的实例对象
*   rootState:当前模板的根state
*   path:当前模板路径，有key组成的数组
*   module：当前模板
*   hot:[Boolean]
**/
function installModule (store, rootState, path, module, hot) {
    // 通过获取当前路径path，得知是否根模块
    const isRoot = !path.length
    // 获取当前模块的命名空间
    const namespace = store._modules.getNamespace(path)
    // register in namespace map
    if (module.namespaced) {
        if (store._modulesNamespaceMap[namespace] && __DEV__) {
            console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}`)
        }
        // 存放当前命名空间模块 eg: {a:module,b:module,''a/c':module}
        store._modulesNamespaceMap[namespace] = module
    }
    // 将所有字模板的state全部添加到rootState上
    // set state
    if (!isRoot && !hot) {
        // 获取上一个模板的state，既当前模板的父级state
        const parentState = getNestedState(rootState, path.slice(0, -1))
        const moduleName = path[path.length - 1]
        store._withCommit(() => {
            if (__DEV__) {
                if (moduleName in parentState) {
                    console.warn(
                        `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
                    )
                }
            }
            Vue.set(parentState, moduleName, module.state)
        })
    }
    // 根据namespace 对  dispatch commit  getter state处理函数进行包装
    const local = module.context = makeLocalContext(store, namespace, path)

    // module.forEachMutation:循环模板中的每个mutations,调用传入的函数
    // 调用当前模块的原型方法  forEachMutation
    // fn 回调函数 mutation：当前mutationd的具体某一项  key:当前mutation的名称
    module.forEachMutation((mutation, key) => {
        // 命名空间与具体mutations项的key联合一起
        const namespacedType = namespace + key
        // store实例   namespacedType：当前mutation对应的key（带命名空间） mutation：当前mutation的具体项  local
        // registerMutation 注册mutations到store._mutations对象中{key:[mutation]}
        registerMutation(store, namespacedType, mutation, local)
    })
    // 注册actions
    module.forEachAction((action, key) => {
        const type = action.root ? key : namespace + key
        const handler = action.handler || action
        registerAction(store, type, handler, local)
    })
    // 注册getter
    module.forEachGetter((getter, key) => {
        const namespacedType = namespace + key
        registerGetter(store, namespacedType, getter, local)
    })
    
    module.forEachChild((child, key) => {
        installModule(store, rootState, path.concat(key), child, hot)
    })
}
```
如上所示，需要代码比较多，但是总体看起来并不复杂，我们来一起看看。

##### installModule函数详情

首先根据当前参数path来确定当前是否根模板，因为path是当前模块的路径，根模板的话这个path就是个空数组
```
const isRoot = !path.length
```
然后获取当前模板对应的命名空间key，通过store._modules获取当前ModuleCollection类的实例，然后调用其原型方法getNamespace。
```
const namespace = store._modules.getNamespace(path)

class ModuleCollection{
    //code...
    getNamespace (path) {
        let module = this.root
        return path.reduce((namespace, key) => {
            // 获取当前模块的子模板实例
            module = module.getChild(key)
            return namespace + (module.namespaced ? key + '/' : '')
        }, '')
    }
    
    //code...
}
```
getNamespace函数作为ModuleCollection类的原型函数，接受一个参数path，Array类型，然后运用path.reduce，起始点位空字符串，每次获取
key对应的module，然后根据module是否带有命名空间，来确定当前key是否连接起来，最后返回完整的命名空间key，多个key中间用"/"号连接，否则直接
返回空字符串，表示当前模板不是命名空间模板。

拿到模板名称后，接下来我们先处理命名模板，将带有命名空间的模板统一挂载到store._modulesNamespaceMap这个集合上，代码如下。
```
if (module.namespaced) {
    if (store._modulesNamespaceMap[namespace] && __DEV__) {
        console.error(`[vuex] duplicate namespace ${namespace} for the namespaced module ${path.join('/')}`)
    }
    // 存放当前命名空间模块 eg: {a:module,b:module,''a/c':module}
    store._modulesNamespaceMap[namespace] = module
}
```
接下来我们判断不是根节点，并且不是热替换时执行以下代码，将所有模板的state全部添加到rootState上
```
if (!isRoot && !hot) {
    // 获取上一个模板的state，既当前模板的父级state
    const parentState = getNestedState(rootState, path.slice(0, -1))
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
        if (__DEV__) {
            if (moduleName in parentState) {
                console.warn(
                    `[vuex] state field "${moduleName}" was overridden by a module with the same name at "${path.join('.')}"`
                )
            }
        }
        Vue.set(parentState, moduleName, module.state)
    })
}
```
当不是根模板且不是热替换时，我们先获取当前模板的父state，在getNestedState函数中，我们获取的state的对象是rootState,然后通过path.reduce
去循环递归访问，获取到当前模板父级state，
```
function getNestedState (state, path) {
    return path.reduce((state, key) => state[key], state)
}
```
接下来取path的最后一项值，既当前模板的名称，作为key，判断一下这个moduleName是否已经存在于父state，然后调用Vue.set，将当前模板的state添加到父级模板的state上，项目中具体如下
```
new Store({
    state:{name:'hepeng'},
    //其他项
    modules:{
        a:{state:{age:'22'}},
        b:{
            state:{job:'66666'},
            //其他项
            b1:{
                state:{sex:'man'}
            }
        }
    }
})
//在整理处理后得到

rootState={
    name:'hepeng',
    a:{age:'22'},
    b:{
        job:'6666',
        b1:{sex:'man'}
    }
}

```
接下来，对当前模板挂载content({dispatch,commit,getters,state})属性，根据当前模板是否带有命名空间，makeLocalContext函数会处理当前访问的应该是根模板数据还是当前模板数据，后续我们
在调用getter，commit,dispatch回调函数的时候我们会提供参数既 当前模板的信息 （既模板上的local信息）或者根模板信息 store.state/getter/commit/dispatch。
```
//Store构造函数中调用makeLocalContext函数，从变量名上了解是：设置本地上下文对象
const local = module.context = makeLocalContext(store, namespace, path)

// Store.js文件中定义makeLocalContext函数
/**
*   接受的参数列表
*   store：当前store实例
*   namespace: 当前命名空间的模板名称，假如该模板不是命名空间，则为空
*   path：当前模板路径组成的数组
*
**/
function makeLocalContext (store, namespace, path) {
    // 根据namespace判断当前模板是否命名空间模块
    const noNamespace = namespace === ''

    const local = {
        //如果不是命名空间模板，则取store.dispatch,如果是，则重新定义dispatch
        dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
            //判断参数类型，重新赋值，因为调用dispatch时有两种传参方式 
            const args = unifyObjectStyle(_type, _payload, _options)
            const { payload, options } = args
            let { type } = args
            //在命名空间模板中是否全局分发action，如果不是，在添加模板路径，表示分发当前模板的actions
            if (!options || !options.root) {
                type = namespace + type
                if (__DEV__ && !store._actions[type]) {
                    console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
                    return
                }
            }
        
            return store.dispatch(type, payload)
        },
        //提交载荷，判断类容上面的dispatch一样
        commit: noNamespace ? store.commit : (_type, _payload, _options) => {
            //提交载荷的方式也有两种，这里做这个处理
            const args = unifyObjectStyle(_type, _payload, _options)
            const { payload, options } = args
            let { type } = args
            //在非命名空间模板中，和处理action类似
            if (!options || !options.root) {
                type = namespace + type
                if (__DEV__ && !store._mutations[type]) {
                    console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
                    return
                }
            }

            store.commit(type, payload, options)
        }
    }
    //定义getters和state属性，并且是只读的
    Object.defineProperties(local, {
        getters: {
            get: noNamespace
                ? () => store.getters
                : () => makeLocalGetters(store, namespace)
            },
        state: {
            get: () => getNestedState(store.state, path)
        }
    })

    return local
}
```
回到installModule函数中，下面我们将看到对mutations、actions、getters的处理，通过三个循环分别将这三个对象添加到对应的Store的属性中，
store._mutations、store._actions、store._wrappedGetters
```
//Store构造函数中的代码
module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
})
// 注册actions
module.forEachAction((action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
})
// 注册getter
module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
})
```
通过调用当前模板实例的forEachMutation，forEachAction，forEachGetter的实例方法，这三个方法比较类似，都是在里面调用forEachValue函数，
既用Object.keys(module._rawModule.mutations/module._rawModule.actions/module._rawModule.getters)获取对应模板里的mutations、actions、
getters的key集合，然后进行循环，在循环中调用传入的回调函数。在回调函数中我们做以下处理
* Mutations
```
(mutation,key)=>{
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
}
function registerMutation (store, type, handler, local) {
    const entry = store._mutations[type] || (store._mutations[type] = [])
    entry.push(function wrappedMutationHandler (payload) {
        handler.call(store, local.state, payload)
    })
}

```
在处理Mutations的回调函数中，先获取当前mutation的对应key和当前所在模板的路径，然后调用registerMutation函数，传入参数为store(当前store的实例对象)，
namespacedType：当前路径+key的组合(当前mutation的完整路径，取决于当前是否作用域模板)，mutation:当前函数，local:根据namespace 对  dispatch commit  getter state处理函数进行包装
后的对象。在registerMutation函数中，我们先获取store._mutations是否存在当前namespacedType，不存在则创建这个属性，并赋值空数组，然后push进一个函数，这个函数接受一个参数，既当前你通过
commit调用mutation时提交的载荷，在这个函数中以当前store实例作为上下文对象调用这个mutations，既我们定义mutation回调，第一个参数为当前的state，第二个参数为载荷。

* actions
```
(action, key) => {
    const type = action.root ? key : namespace + key
    const handler = action.handler || action
    registerAction(store, type, handler, local)
}
function registerAction (store, type, handler, local) {
    const entry = store._actions[type] || (store._actions[type] = [])
    entry.push(function wrappedActionHandler (payload) {
        let res = handler.call(store, {
            dispatch: local.dispatch,
            commit: local.commit,
            getters: local.getters,
            state: local.state,
            rootGetters: store.getters,
            rootState: store.state
        }, payload)
        if (!isPromise(res)) {
            res = Promise.resolve(res)
        }
        if (store._devtoolHook) {
            return res.catch(err => {
                store._devtoolHook.emit('vuex:error', err)
                throw err
            })
        } else {
            return res
        }
    })
}
```
在处理actions的回调函数中，大致处理和上面处理mutations的一样，有区别的是我们在命名空间模板中定义action时，可以设置他的root属性，既调用时
不存在命名空间的限制，上面代码的判断就是做这些的，在最新Vuex API 上并没有这个描述，通过源码我们可以看出来，对于registerAction函数，先在store._actions上获取或者定义对于key的数组，然后在push一个函数，因为在定义actions
的时候，第一个对象是一个参数，随意我们在调用action的时候需要解析local的 dispatch、commit、getters、state,然后解析store上的getters和state，由于我们调用action后返回的需要是个Promise对象，
所有我们在这里判断，如果这个返回值不是Promise对象，则我们定义一个Promise。

* getter
```
(getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
}
```
在处理getter的回调函数中，和上面的mutations原理几乎一样，getter是返回一个值，且对应的key是唯一的，所以我们在registerGetter中需要检查是否重复，
然后 store._wrappedGetters上定义这个属性，这个属性的值是一个函数，该函数在内部调用我们定义的getter，然后传入对应的参数。

注意：为什么在处理actions和mutations定义都是数组呢，因为在不同的模板（不管是否拥有命名空间）我们可以定义相同的action和commit，我们在触发
的时候，可以同时触发多个（这个有命名空间的区别）

installModule函数中的最后一段代码，执行递归操作，对每个模板都需要执行该操作
```
module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
})
```


接下来将介绍Store构造函数的state和getter的初始化。1：初始化Vuex的状态树，将state变成响应式数据，既与Vue的data遵循相同的规则。2：初始化getter，
Vuex文档上介绍getter可以认定是Store的计算属性，且getter的返回值会根据它的依赖被缓存起来，且只有当他的依赖值发生了改变才会被重新计算，
是不是和Vue中的计算属性的描述一模一样，下面我们看看Vuex中是怎么实现的。
```
//Store构造函数中的代码，
 resetStoreVM(this, state)
```
该函数接受两个参数，this:当前store实例对象。state：当前所有模板的state对象，前面提到过，在installModule函数中，我们有将所有的state的引用根据模板之间
的关系都整合到一个对象中。下面我们具体看看这个函数。
```
function resetStoreVM (store, state, hot) {
  // 获取上一次的Vue实例
  const oldVm = store._vm

  store.getters = {}

  store._makeLocalGettersCache = Object.create(null)
  // 当前定义的getter
  const wrappedGetters = store._wrappedGetters
  const computed = {}
  // fn：当前getter   key
  forEachValue(wrappedGetters, (fn, key) => {
    computed[key] = partial(fn, store)
    Object.defineProperty(store.getters, key, {
      get: () => store._vm[key],
      enumerable: true // for local getters
    })
  })

  const silent = Vue.config.silent
  Vue.config.silent = true
  store._vm = new Vue({
    data: {
      $$state: state
    },
    computed
  })
  Vue.config.silent = silent

  if (store.strict) {
    enableStrictMode(store)
  }
  if (oldVm) {
    if (hot) {
      store._withCommit(() => {
        oldVm._data.$$state = null
      })
    }
    Vue.nextTick(() => oldVm.$destroy())
  }
}
```
如上述代码所示，我们循环之前获取的的所有getter，通过partial函数包装一下，并添加到我们创建的computed的空对象中，并在store.getters对象上定义
这个getter，定义为只读属性，只读属性指向store._vm.xxx，既我们就可以通过store.getters.xxx去读取getter。这样我们需要在store实例上定义_vm属性，
在代码中使用建立Vue实例，将我们传入的data:state响应化，将之前我们定义的computed放在Vue的计算属性进行响应化，前面有提到getter与Vue中的computed，
现在看到其实就是用的Vue中的computed进行绑定响应式，接下来当前store是严格模式的时候，只能通过mutation函数改变state，否则报错。

Store类的构造函数的最后一段代码是处理Store插件的，既循环当前插件集合，然后调用每个插件，并传入当前store实例。后面我们介绍定义插件的时候在统一
进行接受。
```
plugins.forEach(plugin => plugin(this))
```

到这里整体就已经很清楚呢，我就不再介绍那些辅助函数了，看了下挺简单，挺好理解的，上面的介绍可以更加断点重新跑几遍更有助于去理解。



