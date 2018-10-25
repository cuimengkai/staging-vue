```
//依赖库 导入有先后
//<link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
//<script src="http://mockjs.com/dist/mock.js"></script>
//<script src="https://cdn.bootcss.com/axios/0.19.0-beta.1/axios.js"></script>
//<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
//<script src="https://cdn.bootcss.com/vuex/3.0.1/vuex.js"></script>
//<script src="https://unpkg.com/element-ui/lib/index.js"></script>

;(function (Vue, Vuex, Mock, axios,Router) {
    console.info(Router)
    // 创建数据
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
    //声明接口
    Mock.mock('/news/index', 'post', produceNewsData);

    //定义接口
    const api = {
        getProducts() {
            return axios.post('/news/index', {keywords: 4})
        },
    }

    //注册Vuex依赖
    Vue.use(Vuex);
    Vue.use(Router);
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
                api.getProducts().then(res => {
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


    const digitsRE = /(\d{3})(?=\d)/g
    const currency = (value, currency, decimals) => {
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

    // 注册过滤器
    Vue.filter('currency', currency)


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
    // 
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
    //创建App组件
    const App = {
        template: `
            <div id="app">
                <router-view/>
             </div>
        `
    };
    // 配置路由
    const routes = [
        {
            path:'/',
            redirect: '/home'
        },
        {
            path:'/home',
            name:'home',
            component: Main
        },
        {
            path:'/detail',
            name:'detail',
            component: Detail
        }
    ]
    // 创建路由实例
    const router = new Router({
        routes,
        mode: 'hash', // history
    })
    //初始化Vue实例
    const app = new Vue({
        el: '#app',
        store: store,
        router:router,
        render: h => h(App)
    })
})(Vue, Vuex, Mock, axios, VueRouter);


```
