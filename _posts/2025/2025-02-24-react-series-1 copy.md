---
layout: post
title: "[庖丁解牛]React系列(一)：React 16有哪些生命周期函数和作用"
date: 2025-02-24
tags: [react, 前端]
comments: false
author: 小辣条
toc : false
---
在 React 16 中，生命周期函数经历了重要调整（尤其是 React 16.3 引入了新的生命周期方法，并废弃了部分旧方法）。本文将介绍React16 的生命周期函数分类及其作用。
<!-- more -->

---

### 一、组件挂载阶段（Mounting）
1. **`constructor()`**  
   - **作用**：初始化组件的 `state` 和绑定事件处理函数。  
   - **注意**：必须调用 `super(props)`，否则 `this.props` 会未定义。

2. **`static getDerivedStateFromProps(props, state)`**（新增，React 16.3+）  
   - **作用**：在组件挂载和更新时，根据新的 `props` 计算并返回新的 `state`。  
   - **注意**：静态方法，无法访问 `this`；返回 `null` 表示不更新 `state`。

3. **`render()`**  
   - **作用**：返回 JSX 结构，描述组件 UI。  
   - **规则**：必须是纯函数，不能修改 `state` 或执行副作用（如 API 调用）。

4. **`componentDidMount()`**  
   - **作用**：组件挂载到 DOM 后触发，适合执行副作用操作（如 API 请求、订阅事件、操作 DOM）。  
   - **示例**：  
     ```javascript
     componentDidMount() {
       fetch('/api/data').then(res => this.setState({ data: res.data }));
     }
     ```

---

### 二、组件更新阶段（Updating）
1. **`static getDerivedStateFromProps(props, state)`**  
   - （同上，在更新阶段也会触发）

2. **`shouldComponentUpdate(nextProps, nextState)`**  
   - **作用**：决定组件是否需要重新渲染，默认返回 `true`。  
   - **优化**：通过比较 `nextProps` 和 `nextState` 避免不必要的渲染。  
   - **示例**：  
     ```javascript
     shouldComponentUpdate(nextProps) {
       return nextProps.value !== this.props.value; // 仅当 value 变化时重新渲染
     }
     ```

3. **`render()`**  
   - （同上，重新渲染 UI）

4. **`getSnapshotBeforeUpdate(prevProps, prevState)`**（新增，React 16.3+）  
   - **作用**：在 DOM 更新前捕获某些信息（如滚动位置），返回值会传递给 `componentDidUpdate`。  
   - **示例**：  
     ```javascript
     getSnapshotBeforeUpdate() {
       return this.listRef.scrollHeight; // 记录当前滚动高度
     }
     ```

5. **`componentDidUpdate(prevProps, prevState, snapshot)`**  
   - **作用**：DOM 更新完成后触发，适合执行依赖新 DOM 的操作。  
   - **注意**：可以比较新旧 `props`/`state`，避免无限循环更新。  
   - **示例**：  
     ```javascript
     componentDidUpdate(prevProps, prevState, snapshot) {
       if (this.props.userID !== prevProps.userID) {
         this.fetchData(this.props.userID); // 当 userID 变化时重新请求数据
       }
     }
     ```

---

### 三、组件卸载阶段（Unmounting）
1. **`componentWillUnmount()`**  
   - **作用**：组件销毁前执行清理操作（如取消定时器、移除事件监听、清理缓存）。  
   - **示例**：  
     ```javascript
     componentWillUnmount() {
       clearInterval(this.timerID); // 清除定时器
       window.removeEventListener('resize', this.handleResize); // 移除事件监听
     }
     ```

---

### 四、错误处理（Error Handling）
1. **`componentDidCatch(error, info)`**（新增，React 16+）  
   - **作用**：捕获子组件树中的 JavaScript 错误，用于记录错误或显示降级 UI。  
   - **注意**：只能捕获子组件的错误，无法捕获自身错误。  
   - **示例**：  
     ```javascript
     componentDidCatch(error, info) {
       logErrorToService(error, info.componentStack); // 上报错误
       this.setState({ hasError: true }); // 显示错误提示
     }
     ```

---

### 五、已废弃的生命周期方法（React 16.3+ 标记为废弃）
- **`componentWillMount()`**：不建议在此处执行副作用（如数据请求），改用 `componentDidMount`。  
- **`componentWillReceiveProps(nextProps)`**：改用 `getDerivedStateFromProps`。  
- **`componentWillUpdate(nextProps, nextState)`**：改用 `getSnapshotBeforeUpdate` + `componentDidUpdate`。

---

### 生命周期流程图（React 16+）
```plaintext
挂载阶段：
constructor → getDerivedStateFromProps → render → componentDidMount

更新阶段：
getDerivedStateFromProps → shouldComponentUpdate → render → getSnapshotBeforeUpdate → componentDidUpdate

卸载阶段：
componentWillUnmount

错误处理：
componentDidCatch
```

---

### 最佳实践
1. **避免在 `render` 和 `constructor` 中执行副作用**。  
2. **优先使用 `componentDidMount` 和 `componentDidUpdate` 处理数据请求**。  
3. **通过 `shouldComponentUpdate` 或 `React.PureComponent` 优化性能**。  
4. **用 `getDerivedStateFromProps` 替代 `componentWillReceiveProps`**，但需谨慎使用（通常应尽量避免派生状态）。  

如果需要更详细的代码示例或特定场景的解决方案，可以进一步说明！

---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️