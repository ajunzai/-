fiber
    两大件 1.vdom数=》链表
                2.利用浏览器渲染的间隔时间 requestIdleCallback
    1.前端架构的迭代，本质上是数据结构和计算机基础逐渐深入的发展
        以前的vdom{type: , children, props}
        新的vdom {type: , child, return, props, slibing}
    2.16.6ms 一帧 任何占用主进程超过这个时间，都可能会卡顿，这种任务，我们基本都可以用fiber这个理念来解决 上传图片md5  有webworker和时间切片

父组件发生变化的时候，子组件不需要渲染？ 
    可以用shouldComponentUpdata，return true更新false不更新。则需要判断新老值的变化是否相等。
    React.PureComponent 中的 shouldComponentUpdate() 仅作对象的浅层⽐较

ref概念
1.ref在模板中用内联函数作为属性值传递的时候，在render渲染时会触发二次更新
2.class中创建ref的方式，React.createRef(),通过current拿到当前组件实例
3.函数式组件通过React.forwardRef((props, ref)=>{})进行转发
4.hoc需要用useRef定义

组件复合
与vue相似通过传入对象来实现具名插槽或不具名插槽，或者通过porps传递显示

hook（类似于class的函数组件，代码抽离易于复用，便于理解）
1.useState定义state。
2.useEffect副作用，（类似于watch）用于请求，类型与didMount和didUpdate。第一个参数为回调执行函数，第二个参数为执行依赖项（为数组，空就为didMount），可以返回一个回调函数用来清空定时器类似willUnmount。
使用规则以及自定义hook
1.在函数最外层调用hook，不能再条件、循环、嵌套函数中调用
2.只能在react函数组件中掉用
API——useMemo和useCallback
useMemo ！！和pureComponent一样作用与函数式组件  把’创建’函数（一个类似计算属性的方法）和依赖项数组作为参数传入useMemo，它仅会在某个依赖项改变时候重新计算
useCallback 把内联回调函数依赖项数组作为参数传入useCallback，它仅会在某个依赖项改变时候重新计算，当NIIT吧回调函数传递给你进过优化的子组件避免非必要的渲染有用

portal传送门——全局弹窗

context
1.创建通过createContext
1.class组件用static方式contextType去获取provide里value的值
2.函数组件用consumer
3.context传递value的时候不要直接写对象，会导致重复渲染

redux
reducer ——定义规则
createStore——创建一个store仓库
    getState()获取当前state
subscribe——订阅更新
dispatch——派发操作

provide
自身接收一个React.CreateContext(),进行连接通信
connect
通过contextType链接store返回的一个新的组件，
两种
mapstateToProps: Function
（state, ownProps）=> {
    !ownProps自己组件的值， ownProps发生变化，mapStateToProps会被重新执行,影响性能
    return {count:state}
}
mapDispatchToProps: Object | Function
    function加入其它action的时候需要通过bindActionCreators绑定
dispatch => {
    return {dispatch}
}

react-router  一切皆组件
<Router>
    <Link />
    <Switch>
        <Route></Route>
        <Redirect to />
    </Switch>
</Router>

route一定要包裹在router中,因为要使用history，location。这些都来自router
Route渲染优先级；chiildren > component > render只能使用一种

react-saga 用于管理应用程序的副作用的库，类比redux-thunk
调用异步操作 call
状态更新        put
监听              takeEvery


umi框架 写react dva类型于仓库 加antd ui框架
