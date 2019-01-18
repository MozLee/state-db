### 痛点是什么？

如图，这是我们没有 mobx/redux/vuex 时的纯组件化开发,  整个页面的状态分散在各个组件内部

1. 组件内部有自己的状态
2. 通过props把自己的状态带给子组件，这个过程是不可控的，有的同学可能把父组件的所有状态都透传下去，有的同学可能取子组件用到的部分透传下去
3. 子组件本身还会有自身的内部状态
4. 父组件可以通过context直接控制孙组件

![](/Users/xieyu/Desktop/企业微信截图_ac750825-f6d6-450e-a97f-81df4c067a14.png)



如图，这是我们使用 mobx/redux/vuex 时开发方式,  远程数据的拉取，操作方法都收敛到了Store内部。但是之前的组件用法仍然可能继续使用，所以如果不加控制，整个应用的状态会分的更散，出了问题更难以排查。

![企业微信截图_d2830699-0338-48bb-993e-5bc95e936a36](/Users/xieyu/Desktop/企业微信截图_d2830699-0338-48bb-993e-5bc95e936a36.png)

### why

1、将所有业务相关的状态收敛到store中，不论是远程拉取还是本地状态。

2、只允许业务组件与store建立关联

3、非业务相关的状态应存留在通用组件里

4、业务模板应该使用无状态的纯渲染组件

5、组合大于嵌套，推荐使用多个平行的业务组件进行组合，不建议过多组件层面的嵌套

6、禁用掉context



好处：

1、所有业务相关的状态都在store中进行排查，提升开发和debug效率。

2、状态处理逻辑和视图分离，视图可复用性更强

3、规范性更强

4、因为状态被收敛，绝大多数组件都可以做成函数式组件，省代码，消除副作用。

5、因为状态被收敛，可以通过devtool一目了然的进行开发

![企业微信截图_8a21a234-f643-41d2-866e-a7c3374a89cf](/Users/xieyu/Desktop/企业微信截图_8a21a234-f643-41d2-866e-a7c3374a89cf.png)

前端的分层与嵌套

本质上一个好的系统一定是存在嵌套的
比如我们了解到的网络协议

IP协议 => TCP协议 => HTTP协议 这是一个层层嵌套的关系，但是这三者又是严格分层的，一个工作在HTTP层的前端小哥是无需关心TCP和IP协议的内容的。

而前端组件化开发的复杂度来源在于我们的组件是可以嵌套的，但是总共有多少层？嵌套多深？每一层做什么事情？这些都是没有一个规范定义的。
我们整个业务的状态就会变得不可控：

即使我们完全按照React的规范来做业务, 会有以下的问题：

- 组件内可能持有业务状态
- redux store/mobx store 可能持有业务状态
- 业务状态可以随着props进行传递
- 业务状态随props传递过程中会进行删减
- 业务状态可以通过context跨层级传递

再加上可能我们还会有一些不那么规范的操作，那就更乱了：
- 原生事件绑定
- 状态直接跟DOM相关联

整个业务应该分为3层：
1、业务状态：动态可变部分，如从服务端动态加载数据
2、资源：固定不变部分，如模板，样式，组件等
3、修改状态的方法集

所有的组件应该处于资源层，不应允许持有任何业务相关的state, 所有组件应该划分为3层

1、基础组件：完全的业务无关，允许存在内部state，并接收props。（比如button， panel等）
2、业务展示组件：可以触发事件，纯展示，没有内部state, 只接收props进行展示，比如一个卡片，一个ListItem等。
3、业务组件：静态文字 + 原生html + 基础组件 + 业务展示组件 + store connection, 不允许存在内部state，不接收props, 业务组件只允许组合，不允许嵌套(循环中的组件除外)
4、入口组件：页面的初始化，DB表的添加与删除

```jsx
<Layout>
    <PageEntry>
        <Wegit1 />
        <Wegit2 />
        <Area>
            <Wegit3 />
            <Wegit4 />
            <Wegit5 />
        </Area>
    </PageEntry>
</Layout>
```


### 推荐开发模式
step1: 创建一个库，建议每个应用只有一个库, 并加入devtool;
```js
import DB from 'state-db.js';
import devtool from 'state-db.js/build/devtool.bundle.js';
const db = new DB();
devtool(db, 'html'); //第二个参数默认为console
export default db;
```

step2: 分析我们的单页APP，哪些状态是某个路由独有的，哪些是页面生命周期内持久存在的

step3.1: 对于持久存在的状态，比如我们从服务端拉一个配置列表下来, 这时我们的建表语句和操作方法定义要放在组件的定义阶段，然后就可以在组件中调用这些方法了，比如：

```js
import db from '../db.js' //刚才已经new好的DB实例
const configTable = db.createTable({
    name: 'configList'
})
const fetchConfigList = () => {
	$.ajax({
        url: url,
      	success: (res) => {configTable.init(res.data); }
    })
}

const getConfig = (confName) => {
    var arr = configTable.where('line.name == "'+ confName +'"').getValues;
    if (arr.length) {
		return arr[0].value;
    }
  	else{
        return none;
    } 
}
 export {fetchConfigList, getConfig}
```

step3.2: 对于某个路由或组件下独有的状态，比如一块独立的业务逻辑, 这时我们的建表语句和操作方法定义要放在组件的执行阶段 ，可以把model包装为一个函数，在组件入口执行处进行调用。`db.dbconnectReact('todos')`高阶组件会帮你进行组件和表之间的绑定和解绑。

```js
import db from '../db.js'
const model = () => {
    const todoTable = db.todoTable({name: 'todos' })
    const fetchTodos = () => {
        $.ajax({
             url: url,
             success: (res) => {todoTable.init(res.data);}
        })
    }
    const getTodos = () => todoTable.getValues();
    return { fetchTodos,  getTodos}
}

@db.dbconnectReact('todos')
class Todo extends Component {
    constructor() {
      this.model = model();
    }
    componentDidMount(){
        this.model.fetchTodos();
    }
  	render() {
        return(
        	return (<ul>
            	{this.model.getTodos().map(todo => <li>{todo.content}/</li>)
        	</ul>)
        )
    }
}
```




### 辅助工具
因为DB是非常结构化的并且能够反映全局的，可以有一个完整的视图来告知我们页面当前的状态，方便我们开发和debug。


### 结合模板使用

```js
const getArticals = articalTable.getValues();
const render = () => {
    str = `<ul>
		${getArticals().map(artical => `<li>${artical.title}/</li>`)}
	</ul>`
    $('#app').innerHTML = str;
}

articalTable.bindFn(render);
```

### 结合react使用

##### 数据表和组件进行绑定（表变化触发组件render）

```jsx
const getArticals = articalTable.getValues()

@db.dbconnectReact('artical')
class Artical extends Component {
    render() {
        return (<ul>
            {getArticals().map(artical => <li>{artical.title}/</li>)
        </ul>)
    }
}
```



### 结合vue使用

```js
const getArticals = articalTable.getValues()

new Vue({
  mixins: [db.dbconnectVue('artical')],
  //... your own logic
})
```




### API 文档

##### 创建数据库实例

```js
const db = new DB();
```

##### 创建一张表

可以通过schema字段限制字段的类型和是否必填

```js
db.createTable({
    name: 'articals', //required
  	schema: {
        id: {type: 'Number', reuqired: true}
      	title: {type: 'String', required: true},
        content: {type: 'String', required: false},
    },  //options
  	initValue:[{id: 1, title: "我的奋斗", content: "你好。。。"}], //options
    pramaryKey: "id" //options
})
```

##### 获取表

```js
const articalTable = db.table('artical');
```

##### 删除表

需要注意的是，drop和clear只是在库中删除表，但是如果表对象依然被应用引用，表对象实际在内存中并未被清空

```js
db.drop('artical'); //删除名为artical的表
db.clear();         //清除全部表
```

##### 监听库变化（增删表时）

```js
db.bindFn((changeInfo) => {
    console.log(changeInfo);
})
```

##### 监听表变化（增删查数据时）

```js
articalTable.bindFn((changeInfo) => {
    console.log(changeInfo);
})
```

##### 增

```js
articalTable.insert({ id: 2, name: "hi，你好"});

articalTable.insert([
    {id: 3, name: '21天精通C++'},
    {id: 4，name: '21天精通Java'}
])
```

##### 删

```js
articalTable.where('line.name=="我的奋斗"').delete()
```

##### 改

```js
articalTable.where('line.id==1').update({name: "你的奋斗"});
```

##### 查

```js
//指定条件的值
articalTable.where('line.name=="我的奋斗 && index !== 1"').getValues();
//查前三个
articalTable.first(3).getValues();
//查后三个
articalTable.last(3).getValues();
```

