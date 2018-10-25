
### 依赖库 导入有先后 
#### [Element-ui css](https://unpkg.com/element-ui/lib/theme-chalk/index.css)
#### [Mock](http://mockjs.com/dist/mock.js)
#### [Axios](https://cdn.bootcss.com/axios/0.19.0-beta.1/axios.js)
#### [Vue](https://cdn.jsdelivr.net/npm/vue/dist/vue.js)
#### [Vue-Router](https://unpkg.com/vue-router@3.0.1/dist/vue-router.js)
#### [Vuex](https://cdn.bootcss.com/vuex/3.0.1/vuex.js)
#### [Element-ui js](https://unpkg.com/element-ui/lib/index.js)

```
;(function (Vue, Vuex, Mock, axios, Router) {
    //TODO mock
    // 基于Mock 声明接口 创建模拟数据
    ;(function (Mock) {
        const Random = Mock.Random;
        const produceNewsData = function () {
            let articles = [];
            for (let i = 0; i < 10; i++) {
                let newArticleObject = {
                    id: i,
                    title: Random.csentence(5, 10), //  Random.csentence( min, max )
                    price: 20,
                    inventory: 10,
                }
                articles.push(newArticleObject)
            }

            return {
                articles: articles
            }
        }
        Mock.mock('/news/index', 'post', produceNewsData);
        return Mock
    })(Mock);

    //TODO API
    //定义接口
    const api = {
        getProducts: (args) => axios.post('/news/index', args),
    }

    //TODO util
    // 定义工具函数
    const util = (function () {
        const digitsRE = /(\d{3})(?=\d)/g
        return {
            currency: (value, currency, decimals) => {
                value = parseFloat(value)
                if (!isFinite(value) || (!value && value !== 0)) return ''
                currency = currency != null ? currency : '$'
                decimals = decimals != null ? decimals : 2
                let stringified = Math.abs(value).toFixed(decimals);
                let _int = decimals ? stringified.slice(0, -1 - decimals) : stringified
                let i = _int.length % 3
                let head = i > 0 ? (_int.slice(0, i) + (_int.length > 3 ? ',' : '')) : ''
                let _float = decimals ? stringified.slice(-1 - decimals) : ''
                let sign = value < 0 ? '-' : ''
                return sign + currency + head + _int.slice(i).replace(digitsRE, '$1,') + _float
            }
        }
    })();

    //TODO Library
    //注册Vue依赖
    Vue.use(Vuex);
    Vue.use(Router);
    Vue.filter('currency', util.currency)

    // TODO vuex
    //创建购物车状态
    const cart = {
        namespaced: true,
        state: {
            items: [],
            checkoutStatus: null
        },
        getters: {
            cartProducts: (state, getters, rootState) => {
                return state.items.map(({id, quantity}) => {
                    const product = rootState.products.all.find(product => product.id === id)
                    return {
                        title: product.title,
                        price: product.price,
                        quantity
                    }
                })
            },
            cartTotalPrice: (state, getters) => {
                return getters.cartProducts.reduce((total, product) => {
                    return total + product.price * product.quantity
                }, 0)
            }
        },
        actions: {
            checkout({commit, state}, products) {
                const savedCartItems = [...state.items]
                commit('setCheckoutStatus', null)
                commit('setCartItems', {items: []})
            },
            addProductToCart({state, commit}, product) {
                commit('setCheckoutStatus', null)
                if (product.inventory > 0) {
                    const cartItem = state.items.find(item => item.id === product.id)
                    if (!cartItem) {
                        commit('pushProductToCart', {id: product.id})
                    } else {
                        commit('incrementItemQuantity', cartItem)
                    }
                    commit('products/decrementProductInventory', {id: product.id}, {root: true})
                }
            }
        },
        mutations: {
            pushProductToCart(state, {id}) {
                state.items.push({
                    id,
                    quantity: 1
                })
            },
            incrementItemQuantity(state, {id}) {
                const cartItem = state.items.find(item => item.id === id)
                cartItem.quantity++
            },
            setCartItems(state, {items}) {
                state.items = items
            },
            setCheckoutStatus(state, status) {
                state.checkoutStatus = status
            }
        }
    }
    //创建产品状态
    const products = {
        namespaced: true,
        state: {
            all: []
        },
        getters: {},
        actions: {
            getAllProducts({commit}) {
                api.getProducts({keyword: 44}).then(res => {
                    commit('setProducts', res.data.articles)
                }).catch(err => {
                    console.info(err)
                })
            }
        },
        mutations: {
            setProducts(state, products) {
                state.all = products
            },

            decrementProductInventory(state, {id}) {
                const product = state.all.find(product => product.id === id)
                product.inventory--
            }
        }

    }
    //创建状态store
    const store = new Vuex.Store({
        modules: {
            cart, // 注册购物车模块
            products, // 注册产品模块
        }
    })

    // TODO components
    //创建购物车组件
    const ShoppingCart = {
        computed: {
            ...Vuex.mapState({
                checkoutStatus: state => state.cart.checkoutStatus
            }),
            ...Vuex.mapGetters('cart', {
                products: 'cartProducts',
                total: 'cartTotalPrice'
            })
        },
        methods: {
            checkout(products) {
                this.$store.dispatch('cart/checkout', products)
            }
        },
        template: `
            <div class="cart">
                <ul>
                  <li
                    v-for="product in products"
                    :key="product.id">
                    {{ product.title }} - {{ product.price | currency }} x {{ product.quantity }}
                  </li>
                </ul>
                <p>Total: {{ total | currency }}</p>
                <p><button :disabled="!products.length" @click="checkout(products)">Checkout</button></p>
                <p v-show="checkoutStatus">Checkout {{ checkoutStatus }}.</p>
             </div>
        `
    }
    //创建产品列表组件
    const ProductList = {
        computed: { // 计算属性
            ...Vuex.mapState({products: state => state.products.all}),
        },
        methods: { // 方法
            ...Vuex.mapActions('cart', ['addProductToCart']),
            indexMethod(index) {
                return index * 2;
            }
        },
        created() {
            this.$store.dispatch('products/getAllProducts')
        },
        template: `
                <el-table :data="products" style="width: 100%" border stripe>
                    <el-table-column type="index" :index="indexMethod" align="center"></el-table-column>
                    <el-table-column prop="title" label="标题" align="center">
                        <template slot-scope="scope">
                            <el-tag size="medium">{{scope.row.title}}</el-tag>
                        </template>
                    </el-table-column>
                    <el-table-column prop="price" label="价格" align="center">
                        <template slot-scope="scope">
                            {{scope.row.price | currency}}
                        </template>
                    </el-table-column>
                    <el-table-column label="操作" align="center">
                        <template slot-scope="scope">
                            <el-button size="mini" type="success" :disabled="!scope.row.inventory" @click="addProductToCart(scope.row)">添加到购物车</el-button>
                            <el-button size="mini"><router-link to="detail">明细</router-link></el-button>
                        </template>
                    </el-table-column>
                </el-table>
        `
    }
    //创建入口组件
    const Main = {
        components: {ProductList, ShoppingCart}, // 注册组件
        template: `
            <div id="app">
                <ProductList/>
                <ShoppingCart/>
             </div>
        `

    }
    const Detail = {
        template: `
            <div id="app">
               路由跳转
             </div>
        `
    }
    //创建App容器组件
    const App = {
        template: `
            <div id="app">
                <router-view/>
             </div>
        `
    };

    //TODO router
    // 配置路由
    const routes = [
        {
            path: '/',
            redirect: '/home'
        },
        {
            path: '/home',
            name: 'home',
            component: Main
        },
        {
            path: '/detail',
            name: 'detail',
            component: Detail
        }
    ]
    // 创建路由实例
    const router = new Router({
        routes,
        mode: 'hash', // history
    })

    //TODO main
    //初始化Vue实例
    const app = new Vue({
        el: '#app',
        store: store,
        router: router,
        render: h => h(App)
    })
})(Vue, Vuex, Mock, axios, VueRouter);


```
