---
title: react学习笔记一
date: 2018-08-19 22:40:27
tags: [react]
---
<!-- more -->
# helloworld
首先看一个最简单的hellowolrd：
```
<div id="example"></div>
<script type="text/babel">
  ReactDOM.render(
    <h1>Hello, world!</h1>,
    document.getElementById('example')
  );
</script>
```
React有一个render函数，传入的第一个参数为一个jsx对象，第二个参数为el的dom对象。react使用jsx来书写模板。
相较之下一个vue的helloworld这样写：
```
<div id="components-demo">
  <button-counter></button-counter>
</div>
Vue.component('button-counter', {
  template: '<h1>Hello, world!</h1>'
})
new Vue({ el: '#components-demo' })
```
这里用到了组件，关键是想比较一下模板，vue的模板语法更接近于html

# 列表渲染
vue需要用v-for指令，而react就牛x了
```
<div id="example"></div>
<script type="text/babel">
  var names = ['Alice', 'Emily', 'Kate'];
  ReactDOM.render(
    <div>
    {
      names.map(function (name, index) {
        return <div key={index}>Hello, {name}!</div>
      })
    }
    </div>,
    document.getElementById('example')
  );
</script>
```
使用一个map函数可以轻松搞定，从这里就可以看出jsx的灵活之处了。

# 组件
```
<div id="example"></div>
<script type="text/babel">
  var HelloMessage = React.createClass({
    render: function() {
      return <h1>Hello {this.props.name}</h1>;
    }
  });
  ReactDOM.render(
    <HelloMessage name="John" />,
    document.getElementById('example')
  );
</script>
```
组件的写法感觉和vue差不多，最大的差别还是模板的写法。所有组件类都必须有自己的 render 方法，用于输出组件。组件类的第一个字母必须大写，否则会报错（vue似乎没有这个要求），比如HelloMessage不能写成helloMessage。另外，组件类只能包含一个顶层标签，否则也会报错。（vue也是如此）

# 获取子元素
```
<script type="text/babel">
  var NotesList = React.createClass({
    render: function() {
      return (
        <ol>
          {
            React.Children.map(this.props.children, function (child) {
              return <li>{child}</li>;
            })
          }
        </ol>
      );
    }
  });
  ReactDOM.render(
    <NotesList>
      <span>hello</span>
      <span>world</span>
    </NotesList>,
    document.getElementById('example')
  );
</script>
```
这个方法也比较猎奇，这个this.props.children应该是来源于虚拟dom

# 属性验证
react用.string.isRquired连点的方式来表明这个属性必须，而且必须是字符串。感觉比较神奇，如果让我来实现，我会用split来分割'.'然后获取对应的约束吧
```
<div id="example"></div>
<script type="text/babel">
  var data = 123;
  var MyTitle = React.createClass({
    propTypes: {
      title: React.PropTypes.string.isRequired,
    },
    render: function() {
      return <h1> {this.props.title} </h1>;
    }
  });
  ReactDOM.render(
    <MyTitle title={data} />,
    document.getElementById('example')
  );
</script>
```
而vue使用一个对象来传递验证的配置,说不清孰优孰劣。
```
Vue.component('my-component', {
  props: {
    // 基础的类型检查 (`null` 匹配任何类型)
    propA: Number,
    // 多个可能的类型
    propB: [String, Number],
    // 必填的字符串
    propC: {
      type: String,
      required: true
    },
    // 带有默认值的数字
    propD: {
      type: Number,
      default: 100
    },
    // 带有默认值的对象
    propE: {
      type: Object,
      // 对象或数组且一定会从一个工厂函数返回默认值
      default: function () {
        return { message: 'hello' }
      }
    },
    // 自定义验证函数
    propF: {
      validator: function (value) {
        // 这个值必须匹配下列字符串中的一个
        return ['success', 'warning', 'danger'].indexOf(value) !== -1
      }
    }
  }
})
```

# 获取dom
react使用refs,vue使用ref

# 响应式
react使用setstate来更新状态，之后会重新执行render方法，个人觉得是setstate之后diff，diff完之后在render。setState会把状态放到一个更新队列，等待flush，react的这种更新机制不依赖于语言具体细节。
```
//将新的state合并到状态更新队列中
var nextState =  this._processPendingState(nextProps, nextContext);
//主要根据shouldComponent的状态来判断是否需要更新组件
var shouldUpdate = 
   this._pendingForceUpdate ||
   !inst.shouldComponentUpdate ||
   inst.shouldComponentUpdate(nextProps, nextState, nextContext);
```
而vue是通过defineProperty来进行进行属性劫持，实现响应式。另外vue的状态更新也依赖浏览器的环境，优先检查mirotask，然后是task。

单纯从数据绑定来看，React虚拟DOM没有数据绑定，因为setState()不维护上一个状态（状态丢弃），谈不上绑定
从数据更新机制来看，React类似于提供数据模型的方式（必须通过state更新）

# 表单
```
<div id="example"></div>
<script type="text/babel">
  var Input = React.createClass({
    getInitialState: function() {
      return {value: 'Hello!'};
    },
    handleChange: function(event) {
      this.setState({value: event.target.value});
    },
    render: function () {
      var value = this.state.value;
      return (
        <div>
          <input type="text" value={value} onChange={this.handleChange} />
          <p>{value}</p>
        </div>
      );
    }
  });
  ReactDOM.render(<Input/>, document.getElementById('example'));
</script>
```
表单事件中获取组件的值不能直接使用this.xxx,vue则可以，因为react不是双向绑定。

# 生命周期
react的组件生命周期分为三个状态
1. Mounting：已插入真实 DOM
2. Updating：正在被重新渲染
3. Unmounting：已移出真实 DOM

这三个状态又分别分为will和did状态。

# ajax
个人觉得react写ajax的方式非常的帅气
```
var UserGist = React.createClass({
  getInitialState: function() {
    return {
      username: '',
      lastGistUrl: ''
    };
  },

  componentDidMount: function() {
    $.get(this.props.source, function(result) {
      var lastGist = result[0];
      if (this.isMounted()) {
        this.setState({
          username: lastGist.owner.login,
          lastGistUrl: lastGist.html_url
        });
      }
    }.bind(this));
  },

  render: function() {
    return (
      <div>
        {this.state.username}'s last gist is
        <a href={this.state.lastGistUrl}>here</a>.
      </div>
    );
  }
});

ReactDOM.render(
  <UserGist source="https://api.github.com/users/octocat/gists" />,
  document.body
);
```
甚至可以直接把promise作为属性传入组件,这就更帅了。
```
ReactDOM.render(
  <RepoList
    promise={$.getJSON('https://api.github.com/search/repositories?q=javascript&sort=stars')}
  />,
  document.body
);
var RepoList = React.createClass({
  getInitialState: function() {
    return { loading: true, error: null, data: null};
  },

  componentDidMount() {
    this.props.promise.then(
      value => this.setState({loading: false, data: value}),
      error => this.setState({loading: false, error: error}));
  },

  render: function() {
    if (this.state.loading) {
      return <span>Loading...</span>;
    }
    else if (this.state.error !== null) {
      return <span>Error: {this.state.error.message}</span>;
    }
    else {
      var repos = this.state.data.items;
      var repoList = repos.map(function (repo) {
        return (
          <li>
            <a href={repo.html_url}>{repo.name}</a> ({repo.stargazers_count} stars) <br/> {repo.description}
          </li>
        );
      });
      return (
        <main>
          <h1>Most Popular JavaScript Projects in Github</h1>
          <ol>{repoList}</ol>
        </main>
      );
    }
  }
});
```

# 参考文献
[React 入门实例教程](http://www.ruanyifeng.com/blog/2015/03/react.html)
[setState详解](https://blog.csdn.net/sysuzhyupeng/article/details/63250741)
[关于React setState的实现原理（一）](https://www.cnblogs.com/jasonlzy/p/8046118.html)
[vue开发：vue,angular,react数据双向绑定原理分析](https://blog.csdn.net/generon/article/details/72835645)