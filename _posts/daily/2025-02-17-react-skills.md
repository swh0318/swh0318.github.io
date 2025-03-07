---
layout: post
title: "[每日一文]前端组件使用小技巧：组件状态提升和父组件状态感知"
date: 2025-02-17
tags: [react, vue]
comments: false
author: 小辣条
toc : false
---
在react中，父组件如何触发子组件的提交操作？父组件想获取到子组件的内容或变动情况？本文进行简单的介绍。
<!-- more -->

# 1、父组件如何触发子组件的提交操作？
在 React 类组件中，父组件可以通过 ref 获取子组件实例，并直接调用子组件的方法来实现触发操作。具体步骤如下：
1. 子组件：定义提交方法（如 handleSubmit）
2. 父组件：创建 ref 并绑定到子组件
3. 父组件：通过 ref 调用子组件的提交方法

```
jsx
// 子组件
class ChildComponent extends React.Component {
  handleSubmit = () => {
    console.log('提交操作');
  }

  render() {
    return (
      <div>子组件</div>
    );
  }
}

// 父组件
class ParentComponent extends React.Component {
  constructor(props) {
    super(props);
    this.childRef = React.createRef();
  }

  handleClick = () => {
    this.childRef.current.handleSubmit();
  }

  render() {
    return (
      <div>
        <ChildComponent ref={this.childRef} />
        <button onClick={this.handleClick}>触发提交</button>
      </div>
    );
  }
}

ReactDOM.render(<ParentComponent />, document.getElementById('root'));
```

# 2、前端组件状态提升：父组件想获取到子组件的内容或变动情况？
场景：父组件需要获取子组件的内容或变动情况，可以通过将子组件的状态提升到父组件中，实现父组件状态感知；另外，父组件可以通过 props 将方法传递给子组件，实现子组件状态更新。状态提升也可以实现组件之间的通信。

```
jsx
// 子组件
class ChildComponent extends React.Component {
  state = {
    value: ''
  }

  handleChange = (e) => {
    this.setState({
      value: e.target.value
    });
  }

  render() {
    return (
      <input value={this.state.value} onChange={this.handleChange} />
    );
  }
}

// 父组件
class ParentComponent extends React.Component {
  state = {
    value: ''
  }

  handleChange = (value) => {
    this.setState({
      value
    });
  }

  render() {
    return (
      <div>
        <ChildComponent onChange={this.handleChange} />
        <p>父组件获取到的值：{this.state.value}</p>
      </div>
    );
  }
}
```

---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️