# State management with RxJS

RxJS comes with a reactive primitive called BehaviorSubject. BehaviorSubject is a wrapper object that keeps an internal state that you can update, read, and subscribe to, to run callbacks when the state is updated. It's framework agnostic, so can you can use it, not only with Angular, but with React, Vue, Svelte, Solid, or vanilla JS.

# Framework agnostic reactive state

New BehaviorSubjects can be instantiated like any other object, inside or outside of ui components. BehaviorSubject requires an initial state so that it can return the current state to any new subscribers immediately.

```ts
const state = new BehaviorSubject("");
```

We can subscribe to the state to run side effects, such as rerendering components, by passing the subscribe method an observer. Observers can be simple callback functions, or Observer objects, which have definitions for methods `next, error` and `complete`.
This can be thought of like adding an event listener that triggers whenever the state changes.

```ts
const subscription1 = state.subscribe((value) => {
  console.log("Observer 1: ", value);
});

const subscription2 = state.subscribe({
  next: (value) => {
    console.log("Observer 2: ", value);
  },
  error: (error) => {
    console.error("Observer 2 had an error: ", error);
  },
  complete: () => {
    console.log("BehaviorSubject disposed and won't be emitting new values.");
  },
});
```

To update the state and run all the observer callbacks, we can call the next method. This updates the internal state and propagates updates to all the observers.

```ts
state.next("Hello world");
// logs out Observer 1: Hello world
// logs out Observer 2: Hello world
```

Subscribing to an observable returns a Subscription object that can be used to remove the observer, for example when ui components unmount. This can be thought of like a clean up function for useEffect in React. Conveniently, Subscription objects can also be added to other Subscription objects to dispose all of them together.

```ts
subscription1.unsubscribe();
subscription2.unsubscribe();

const subscription = new Subscription();
subscription.add(subscription1);
subscription.add(subscription2);
subscription.unsubscribe();
```

# Framework integration

To integrate external state with frameworks you generally need only two things, some method of subscribing to the BehaviorSubjects and rerendering the ui component on changes. Angular provides out of the box integration with RxJS Observables with the async pipe. Other frameworks can use the following approach demonstrated with React.

## React integration

Basic framework integration requires subscribing to the BehaviorSubject and assigning the current value to the component state to make the component rerender when state changes. In React this can be done with `useState + useEffect` combination.

```ts
function useBehaviorSubject1<T>(subject: BehaviorSubject<T>): T {
  const [state, setState] = useState(subject.getValue());

  useEffect(() => {
    const subscription = subject.subscribe(setState);

    return () => {
      subscription.unsubscribe();
    };
  }, [subject]);

  return state;
}
```

React 18 also introduced a new hook specifically for external state management `useSyncExternalStore` which does basically the same thing. It subscribes to the provided source with the first callback and whenever the value changes it runs the second callback to get the most recent value, and rerenders the component if the returned value is different. This requires immutable state updates like React state usually does for change detection.

```ts
function useBehaviorSubject2<T>(subject: BehaviorSubject<T>): T {
  return useSyncExternalStore(
    (observer) => {
      const subscription = subject.subscribe(observer);

      return () => {
        subscription.unsubscribe();
      };
    },
    () => subject.getValue()
  );
}
```

Most frameworks can use this approach by subscribing to the BehaviorSubject and using the framework native reactive primitive as the trigger to rerender components.

# External state example (React)

Putting it all together, here's a minimal example of creating an integration hook, store with some simple logic, and using it in an ui component.

```tsx
type ReactObserver = () => void;
type CleanupSubscription = () => void;
type ReactSubscriber = (onStoreChange: ReactObserver) => CleanupSubscription;
type SnapshotGetter<T> = () => T;

function subscribeFunc<T>(subject: BehaviorSubject<T>): ReactSubscriber {
  return (onStoreChange: ReactObserver) => {
    const subscription = subject.subscribe(onStoreChange);
    return () => {
      subscription.unsubscribe();
    };
  };
}

function snapshotFunc<T>(subject: BehaviorSubject<T>): SnapshotGetter<T> {
  return () => {
    return subject.getValue();
  };
}

function useExternalState<T>(subject: BehaviorSubject<T>): T {
  return useSyncExternalStore(subscribeFunc(subject), snapshotFunc(subject));
}

type Todo = {
  id: string;
  task: string;
  done: boolean;
};

class TodoStore {
  public readonly todos = new BehaviorSubject<Todo[]>([]);

  public addTodo(task: string) {
    const newTodo = { id: createId(), task, done: false };
    const todos = this.todos.getValue();
    const newTodos = [...todos, newTodo];
    this.todos.next(newTodos);
  }

  public removeTodo(id: string) {
    const todos = this.todos.getValue();
    const newTodos = todos.filter((t) => t.id !== id);
    this.todos.next(newTodos);
  }

  public toggleTodo(id: string) {
    const todos = this.todos.getValue();
    const newTodos = todos.map((item) =>
      item.id === id ? { ...item, done: !item.done } : item
    );
    this.todos.next(newTodos);
  }
}

const todoStore = new TodoStore();

function TodoList() {
  const [task, setTask] = useState("");
  const todos = useExternalState(todoStore.todos);

  function taskChanged(e: React.ChangeEvent<HTMLInputElement>) {
    setTask(e.currentTarget.value);
  }

  function addClicked() {
    todoStore.addTodo(task);
    setTask("");
  }

  function toggleClicked(target: Todo) {
    return () => {
      todoStore.toggleTodo(target.id);
    };
  }

  function removeClicked(target: Todo) {
    return (e: React.MouseEvent<HTMLButtonElement>) => {
      e.stopPropagation();
      todoStore.removeTodo(target.id);
    };
  }

  return (
    <main>
      <h1>Todos</h1>

      <input
        placeholder="Add new todo..."
        value={task}
        onChange={taskChanged}
      />
      <button onClick={addClicked}>Add</button>

      <div>
        {todos.length === 0 && <p>No todos.</p>}
        {todos.map((item) => {
          return (
            <section
              key={item.id}
              aria-disabled={item.done}
              onClick={toggleClicked(item)}
            >
              <span>{item.task}</span>
              <button onClick={removeClicked(item)}>X</button>
            </section>
          );
        })}
      </div>
    </main>
  );
}
```
