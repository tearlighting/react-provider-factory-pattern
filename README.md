# react-provider-factory-pattern

🧪 Experimental pattern for React state management:  
**Factory (closure) + Provider + useSyncExternalStore = Polymorphic Store with fine-grained updates**

---

## Motivation

- React Context 默认全量 re-render，不适合大表单/大列表场景。  
- Zustand/Jotai 这类库解决了一部分，但抽象层次固定。  
- 我想尝试：**把 store 本身抽象为工厂**，然后用 Provider 注入，这样可以：  
  - 支持多个 store 实例（多态）  
  - 用 `useSyncExternalStore` 保证 selector 精细化更新  
  - store 内部实现可以自由切换（闭包 / reactive-core / 其他）

---

## Example

```tsx
import React, { createContext, useContext, useRef } from "react"
import { useSyncExternalStore } from "react"

// ---- Store Factory ----
function createCounterStore() {
  let state = { count: 0 }
  const listeners = new Set<() => void>()

  const getState = () => state
  const setState = (partial: Partial<typeof state>) => {
    state = { ...state, ...partial }
    listeners.forEach(l => l())
  }
  const subscribe = (listener: () => void) => {
    listeners.add(listener)
    return () => listeners.delete(listener)
  }

  return { getState, setState, subscribe }
}

// ---- Hook creator (selector support) ----
function createUseStore<T>(store: ReturnType<typeof createCounterStore>) {
  return function useStore<Selected>(selector: (s: T) => Selected): Selected {
    return useSyncExternalStore(
      store.subscribe,
      () => selector(store.getState()) // only re-render if selector result changes
    )
  }
}

// ---- Provider Pattern ----
const CounterContext = createContext<ReturnType<typeof createCounterStore> | null>(null)

export function CounterProvider({ children }: { children: React.ReactNode }) {
  const storeRef = useRef(createCounterStore())
  return <CounterContext.Provider value={storeRef.current}>{children}</CounterContext.Provider>
}

export function useCounter<Selected>(selector: (s: { count: number }) => Selected): Selected {
  const store = useContext(CounterContext)!
  return createUseStore(store)(selector)
}

// ---- Usage ----
function Display() {
  const count = useCounter(s => s.count)
  return <div>Count: {count}</div>
}

function Controls() {
  const store = useContext(CounterContext)!
  return (
    <button onClick={() => store.setState({ count: store.getState().count + 1 })}>
      Increment
    </button>
  )
}

export default function App() {
  return (
    <CounterProvider>
      <Display />
      <Controls />
    </CounterProvider>
  )
}
```
## Key Points

createCounterStore → 工厂函数，返回闭包 store。

useSyncExternalStore → 保证 selector 精细化更新。

Provider → 注入多态 store 实例，不同 Provider = 不同 store。

useCounter(selector) → 组件只关心自己订阅的片段。
## TODO

More demos (forms, nested providers).

Benchmark vs Zustand/Jotai.

Try swapping the factory core (e.g. Vue's reactive) inside.
