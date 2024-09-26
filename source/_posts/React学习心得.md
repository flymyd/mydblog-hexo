---
title: React学习心得
toc: true
date: 2023-05-22 00:00:00
tags: [React]
categories:
  - [前端技术, 学习笔记]
---
## 一、JSX

- jsx的声明很简单，形如：

  ```jsx
  const element = (
    <div>
      <h1>Hello world！</h1>
    </div>
  )
  ```

- 在jsx中，React组件的标签名必须要大写，而DOM标签名必须小写。

- jsx中的表达式模板为`{}`而非`{{}}`

- jsx中只能使用表达式，而不能用多行JS语句。这一点类似于vue中模板、属性等动态赋值时的写法

- 部分标签属性名称有变化，如class => className，onclick => onClick

- jsx中的注释为{/**/}，例如：

  ```jsx
  const element = (
  	<div>
  		{/* 这里是一个注释 */}
  		<span>React</span>
  	</div>
  )
  ```

- jsx在react中实质上只是React.createElement的语法糖。

## 二、组件

- 声明一个简单的React组件，render函数返回的jsx其实就相当于Vue中的template，有且只能有一个根节点

  ```jsx
  import React, { Component } from "react";
  class PostList extends Component {
    render() {
      return (
      	<div>
        	<p>Hello World!</p>
        </div>
      )
    }
  }
  export default PostList;
  ```

- 给上述组件设置两个Props：title和subtitle

  ```jsx
  import React, { Component } from "react";
  class PostList extends Component {
    render() {
      const { title, subtitle } = this.props;
      return (
      	<div>
          <h1>{title}</h1>
          <h2>{subtitle}</h2>
        	<p>Hello World!</p>
        </div>
      )
    }
  }
  export default PostList;
  ```

- 有状态组件：需要用到state保存变化；state其实就类似于Vue中的data（this.state.foo = data{ foo }）。

- 简单的计数器组件（有状态）：

  ```jsx
  class PostItem extends Component{
    constructor(props){
      super(props)
      this.state = {vote: 0}
    }
    handleClick(){
      let vote = this.state.vote;
      vote++;
      this.setState({vote: vote})
    }
    render(){
      return(
        <div>
          <button onClick={()=>{this.handleClick()}}>点击增加</button>
          <p>{this.state.vote}</p>
        </div>
      )
    }
  }
  ```

- 自定义组件的构造器中首先要调用super(props)，这一步是为了调用React.Component的构造器。

- 无状态组件的函数式声明

  ```jsx
  function Welcome(props) {
    return <h1>Hello, {props.name}</h1>;
  }
  ```

- Props的类型校验

  ```jsx
  import PropTypes from 'prop-types';
  class PostItem extends React.Component {
    PostItem.propTypes = {
      post: PropTypes.object,
      onVote: PropTypes.func
    }
  }
  ```

  这类似于Vue中的Props类型校验。但在React中除了基本类型外还多了Element和Node属性。

  当需要判断object和array内部数据类型的时候，可以如下操作：

  ```jsx
  PostItem.propTypes = {
      style: PropTypes.shape({
        color: PropTypes.string,
        fontSize: PropTypes.number
      }),
      sequence: PropTypes.arrayOf(PropTypes.number)
    }
  ```

  这有点像TypeScript中对于类型的声明。

  如果该Props是必传的，那要在类型属性上调用isRequired，例：

  ```jsx
  PostItem.propTypes = {
      style: PropTypes.shape({
        color: PropTypes.string,
        fontSize: PropTypes.number
      }).isRequired
    }
  ```

  如果需要为Props指定默认值，则可以使用defaultProps实现：

  ```jsx
  function Welcome(props) {
         return <h1 className='foo'>Hello, {props.name}</h1>;
  }
       Welcome.defaultProps = {
         name: 'Stranger'
  };
  ```

- 内联样式的使用

  ```jsx
  function Welcome(props) {
    return (
      <h1
        style={{
          backgroundColor: 'blue',
          fontSize: '20px'
        }}
        >Hello,{props.name}</h1>
    )
  }
  ```

  style后的双花括号实际上的含义是：一个JS表达式中包括一个对象。

  也可以把上面的代码改写一下，将样式类抽取出来单独声明：

  ```jsx
  function Welcome(props) {
    const txtStyle = {
      backgroundColor: 'blue',
      fontSize: '20px'
    };
    return (
      <h1 style={txtStyle}>Hello,{props.name}</h1>
    )
  }
  ```

  这类似于Vue中的

  ```html
  <h1 :style="txtStyle">Hello, {{name}}</h1>
  ```

- 引入外部CSS：

  ```jsx
import '../assets/css/common.css'
  ```

- 引入外部图片：

  ```jsx
import girl from '../assets/images/1.jpg'
  class Test extends Component {
  constructor(props) {
      super(props);
    this.state={
        img: '../assets/images/2.jpg'
    }
    }
  render(){
      return (
    	<img src={girl} alt="pic1" />
        <img src={require('../assets/images/2.jpg')} alt="pic2" />
    )
    }
}
  ```
  
  
  
- 组件的生命周期

  挂载阶段：

  constructor（调用React.Component的构造器，进行state的初始化及绑定事件处理方法等工作）

  componentWillMount（类似beforeMounted，在这里调用this.setState不会引起组件的重新渲染）

  render：根据props和state返回一个React Element。在这里不能调用this.setState，否则会改变组件的状态

  componentDidMount：组件挂载到DOM后调用。此时已经可以获取到DOM，类似于Mounted。在这里调用this.setState也会引起组件的重新渲染。

  componentWillReceiveProps(newProps)：父组件传给该组件新的props时触发的钩子，一般需要比较新props与this.props来决定是否执行props变化后的逻辑，例如根据新props调用this.setState触发重新渲染。

  shouldComponentUpdate(newProps, newState)：该方法决定组件是否继续更新。一般通过比较新props、state与组件当前的props、state来决定该方法返回true还是false，从而减少组件不必要的渲染，优化性能。

  componentWillUpdate(newProps, newState)：该方法在render调用前执行。

  componentDidUpdate(oldProps, oldState)：组件更新后的钩子，可以在此处操作更新后的DOM
  
  componentWillUnmount：卸载组件前调用，类似于beforeDestory。
  
- 循环的使用

  在Vue中循环渲染一个组件，类似下例：

  ```html
  <div>
  	<PostItem v-for="(item, i) in postItems" 
              :key="item.id" 
              :post="item" 
              @onVote="handleVote"></PostItem>
  </div>
  ```

  而在React中，可以如下操作：

  ```jsx
  render(){
    return (
    	<div>
      	{this.state.postItems.map((item,index)=>
        	return (
  					<PostItem key={item.id} post={item} onVote={this.handleVote}></PostItem>
        	)
        )}
      </div>
    )
  }
  ```

  或者使用自运行函数，去执行其他的操作，例如for：

  ```jsx
  const data = [{
      foo: 111,
      bar: 222
    },{
      foo: 333,
      bar: 444
    }]
    return (
      <div className="App">
        {
           (() => {
            let domArr = [];
            for(const item of data) {
                domArr.push(<li>{item.foo},{item.bar}</li>)
            }
            return domArr;
          })()
        }
      </div>
    );
  ```

  for也可以通过预先转换数组的形式往页面里回写：

  ```jsx
  render() {
    let domList = this.state.data.map((item,index)=>{
      return (
      	<li>{item.foo},{item.bar}</li>
      )
    })
    return (
      <div className="App">
      	{domList}
      </div>
    )
  }
  ```

  

- 

