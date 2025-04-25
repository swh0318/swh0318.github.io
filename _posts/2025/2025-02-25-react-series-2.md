---
layout: post
title: "[庖丁解牛]React系列(二)：状态管理工具mobx-react库的使用"
date: 2025-02-25
tags: [react, 前端]
comments: false
author: 小辣条
toc : false
---
本文是关于 mobx-react 的使用指南，结合 MobX 的核心概念和 React 组件的集成方法，帮助你更好地理解和使用 mobx-react 库。
<!-- more -->

---

### 一、安装与配置
1. **安装依赖**  
   ```bash
   npm install mobx mobx-react
   # 或
   yarn add mobx mobx-react
   ```

2. **启用装饰器语法（可选）**  
   如果使用装饰器语法（如 `@observable`, `@action`），需配置 Babel 或 TypeScript：
   - **Babel**：安装插件 `@babel/plugin-proposal-decorators`  
     ```bash
     npm install --save-dev @babel/plugin-proposal-decorators
     ```
     在 `.babelrc` 中添加配置：
     ```json
     {
       "plugins": [
         ["@babel/plugin-proposal-decorators", { "legacy": true }]
       ]
     }
     ```

   - **TypeScript**：在 `tsconfig.json` 中启用：
     ```json
     {
       "compilerOptions": {
         "experimentalDecorators": true,
         "emitDecoratorMetadata": true
       }
     }
     ```

---

### 二、核心概念与用法
#### 1. 创建 Store（状态管理）
使用 `observable`、`action` 和 `computed` 定义可观察状态和操作：
```javascript
import { observable, action, computed, makeObservable } from 'mobx';

class CounterStore {
  constructor() {
    makeObservable(this); // 非装饰器语法需调用
  }

  @observable count = 0; // 可观察状态

  @action // 定义动作（修改状态）
  increment = () => {
    this.count++;
  };

  @action
  decrement = () => {
    this.count--;
  };

  @computed // 计算属性
  get doubleCount() {
    return this.count * 2;
  }
}

const counterStore = new CounterStore();
export default counterStore;
```

---

#### 2. 将 Store 注入 React 组件
使用 `Provider` 和 `inject` 将 Store 传递给组件树。

**根组件**（`App.js`）：
```jsx
import { Provider } from 'mobx-react';
import CounterStore from './stores/CounterStore';

function App() {
  return (
    <Provider counterStore={CounterStore}>
      <CounterComponent />
    </Provider>
  );
}
```

**子组件**（类组件）：
```jsx
import { inject, observer } from 'mobx-react';

@inject('counterStore') // 注入 Store
@observer // 观察状态变化
class CounterComponent extends React.Component {
  render() {
    const { counterStore } = this.props;
    return (
      <div>
        <p>Count: {counterStore.count}</p>
        <button onClick={counterStore.increment}>+</button>
        <button onClick={counterStore.decrement}>-</button>
        <p>Double: {counterStore.doubleCount}</p>
      </div>
    );
  }
}
```

**函数组件**（Hooks）：
```jsx
import { observer } from 'mobx-react';
import { useStore } from './stores/CounterStore'; // 直接导入 Store 实例

const CounterComponent = observer(() => {
  const counterStore = useStore(); // 或通过 useContext 获取
  return (
    <div>
      <p>Count: {counterStore.count}</p>
      <button onClick={counterStore.increment}>+</button>
      <button onClick={counterStore.decrement}>-</button>
    </div>
  );
});
```

---

#### 3. 使用 `observer` 包裹组件
- **作用**：自动追踪组件内使用的可观察状态，并在状态变化时触发重新渲染。  
- **注意**：只有被 `observer` 包裹的组件才会响应状态变化。

---

### 三、异步操作处理
使用 `action` 包裹异步操作，或在回调中调用 `runInAction`：
```javascript
import { observable, action, runInAction } from 'mobx';

class UserStore {
  @observable users = [];
  @observable isLoading = false;

  @action
  fetchUsers = async () => {
    this.isLoading = true;
    try {
      const response = await fetch('/api/users');
      const users = await response.json();
      runInAction(() => { // 在异步回调中更新状态
        this.users = users;
        this.isLoading = false;
      });
    } catch (error) {
      runInAction(() => {
        this.isLoading = false;
      });
    }
  };
}
```

---

### 四、最佳实践
1. **最小化观察范围**  
   使用 `@observer` 仅包裹需要响应状态变化的组件，避免不必要的渲染。

2. **避免直接修改状态**  
   始终通过 `@action` 修改状态，启用严格模式确保规范：
   ```javascript
   import { configure } from 'mobx';
   configure({ enforceActions: "observed" }); // 强制通过 action 修改状态
   ```

3. **优化性能**  
   - 使用 `@computed` 缓存计算结果。  
   - 对于列表渲染，为列表项添加唯一 `key`，并使用 `observer` 包裹单个项组件。

---

### 五、常见问题
1. **装饰器语法报错**  
   - 确保 Babel/TypeScript 配置正确。  
   - 或改用非装饰器语法（如 `makeObservable`）。

2. **组件未更新**  
   - 检查是否遗漏了 `@observer`。  
   - 确认状态修改是通过 `@action` 进行的。

3. **循环依赖**  
   - 避免在 Store 之间直接相互引用，可通过依赖注入解决。

---

### 六、完整示例
**Store**（`CounterStore.js`）：
```javascript
import { observable, action, computed, makeObservable } from 'mobx';

class CounterStore {
  constructor() {
    makeObservable(this);
  }

  @observable count = 0;

  @action
  increment = () => {
    this.count++;
  };

  @action
  decrement = () => {
    this.count--;
  };

  @computed
  get doubleCount() {
    return this.count * 2;
  }
}

export default new CounterStore();
```

**组件**（`Counter.jsx`）：
```jsx
import { observer } from 'mobx-react';
import counterStore from './CounterStore';

const Counter = observer(() => {
  return (
    <div>
      <p>Count: {counterStore.count}</p>
      <button onClick={counterStore.increment}>+</button>
      <button onClick={counterStore.decrement}>-</button>
      <p>Double: {counterStore.doubleCount}</p>
    </div>
  );
});

export default Counter;
```

---
### 保持好奇，勇敢打破边界，做永远不被定义的人，祝你快乐❤️