## React Fiber Architecture 

This document provides a deep dive into React Fiber, a significant reimplementation of React's core algorithm.

### Introduction

React Fiber aims to enhance React's suitability for animation, layout, and gestures. Its key feature is **incremental rendering**, allowing work to be split and spread across multiple frames, resulting in smoother performance. Additionally, Fiber enables features like:

* Pausing, resuming, or aborting work based on incoming updates.
* Assigning priorities to different types of updates.
* Utilizing new concurrency primitives.

This guide aims to explain Fiber's concepts clearly, avoiding jargon and providing definitions for key terms. External resources will be referenced for further exploration.

**Important Note:**

* This is not an official React document and is still under development. The information might change as Fiber evolves. 
* Improvements and suggestions are welcome!

**Prerequisites**

Understanding these resources is highly recommended before proceeding:

* **React Components, Elements, and Instances:** [https://github.com/goodmodule/react-facebook](https://github.com/goodmodule/react-facebook)
* **Reconciliation:** [https://github.com/rohan-paul/Awesome-JavaScript-Interviews/blob/master/React/Virtual-DOM-and-Reconciliation-Algorithm.md](https://github.com/rohan-paul/Awesome-JavaScript-Interviews/blob/master/React/Virtual-DOM-and-Reconciliation-Algorithm.md)
* **React Basic Theoretical Concepts:** [https://github.com/reactjs/react-basic](https://github.com/reactjs/react-basic)
* **React Design Principles:** [https://github.com/uxdom/react-design-patterns](https://github.com/uxdom/react-design-patterns) - Pay close attention to the scheduling section.

## Review

### Reconciliation Explained

* **Reconciliation:** The algorithm React uses to compare two trees (virtual DOM) and determine which parts need updates.
* **Update:** A change in the data used to render a React app, often triggered by `setState`.

React treats updates as if they cause the entire app to re-render. This allows for declarative development, but for real-world apps, complete re-rendering can be expensive. 

**Reconciliation** is the process that optimizes re-rendering:

1. When rendering, a tree representing the app is created and stored in memory (virtual DOM).
2. This tree is then "flushed" to the rendering environment (e.g., translated to DOM operations in browsers).
3. When an update occurs (usually via `setState`), a new tree is generated.
4. The new and old trees are compared (diffed) to determine the minimal set of operations needed to update the rendered app.

**Key Points:**

* Different component types might lead to entirely new trees being generated instead of diffs.
* Keys are crucial for efficient diffing of lists and should be "stable, predictable, and unique."

### Reconciliation vs. Rendering

The virtual DOM is a conceptual term; React can render to various environments, including native iOS/Android views via React Native.

**Separation of Concerns:**

* Reconciliation determines which parts of the tree have changed.
* The renderer uses this information to update the actual rendered app.

This separation allows React DOM and React Native to have their own renderers while sharing the same core reconciliation engine.

**Scheduling Explained**

Scheduling refers to determining when work should be performed. In a UI, immediate updates aren't always necessary, and prioritizing updates based on user experience is crucial.

React's Design Principles state the importance of scheduling:

> React currently walks the tree recursively and calls render functions of the whole updated tree during a single tick. However in the future it might start delaying some updates to avoid dropping frames.

**Key Points:**

* UI updates don't always need to be immediate; delays can prevent dropped frames and improve user experience.
* Different updates have different priorities (e.g., animations are more critical than background data updates).
* A pull-based approach (React's approach) allows the framework to make scheduling decisions, while a push-based approach requires manual scheduling by developers.

## What is a Fiber?

Fibers are a core concept in React Fiber's architecture. They represent **units of work** and are lower-level abstractions than what developers typically interact with. Understanding them might require some effort, but it's valuable for optimizing UI rendering.

**Understanding Fibers through Function Calls and Stacks:**

Traditional computers track program execution using a **call stack**. A new **stack frame** is added when a function is called. This stack frame represents the work performed by that function.

However, UI components have specific concerns compared to general functions. In UI rendering, too much work at once can cause choppy animations, and some work might be unnecessary due to more recent updates.

**Virtual Stack Frames with React Fiber:**

React Fiber addresses these issues by introducing **fibers**, which act as virtual stack frames. Fibers allow for:

* **Pausing work and resuming it later.**
* **Assigning priorities to different types of work.**
* **Reusing previously completed work.**
* **Aborting work if it's no longer needed.**

## Fiber Node Structure Breakdown
```js
function FiberNode(tag, key, pendingProps, memoizedProps, returnFiber, child, sibling,   alternate, effectTag, updateQueue) {
  this.tag = tag; // Type of node (Host, FunctionComponent, ClassComponent, etc.)
  this.key = key; // Unique identifier for the element
  this.pendingProps = pendingProps; // Incoming props before reconciliation
  this.memoizedProps = memoizedProps; // Props used for the previous render
  this.returnFiber = returnFiber; // Parent fiber in the tree
  this.child = child; // First child fiber
  this.sibling = sibling; // Next sibling fiber
  this.alternate = alternate; // Alternate fiber (used for work-in-progress)
  this.effectTag = effectTag; // Flags for side effects (placement, update, deletion)
  this.updateQueue = updateQueue; // Queue of pending state updates
}
```

Here's the revised README.md entry with main headings in `**` and other headings in `*`:

---


## Fiber Properties**

Each FiberNode represents a unit of work in React Fiber. Key properties include:

* **`tag`** (String): Identifies the type of React element the fiber represents. Possible values:
  - `Host`: Represents a DOM element.
  - `FunctionComponent`: Represents a functional component.
  - `ClassComponent`: Represents a class component.
  - Other types may exist depending on React's implementation.

* **`key`** (Optional String): A unique identifier for the element within lists. Essential for efficient reconciliation. Keys should be stable and predictable across renders.

* **`pendingProps`** (Object): The incoming props passed to the component before reconciliation occurs.

* **`memoizedProps`** (Object): The props used for the previous render of the component, helping to detect prop changes during reconciliation.

* **`returnFiber`** (FiberNode): Points to the parent fiber in the component tree, establishing the hierarchical relationship.

* **`child`** (FiberNode): Represents the first child fiber of the current fiber.

* **`sibling`** (FiberNode): Points to the next sibling fiber in the tree hierarchy, facilitating traversal of the component tree structure.

* **`alternate`** (FiberNode): Holds the previous version of the fiber for comparison during reconciliation, enabling efficient diffing. When updates occur, a new fiber is created with the updated state.

* **`effectTag`** (Number): A bitmask representing side effects that need to be performed after reconciliation. Examples include:
  - `PLACEMENT`: Adding a new element.
  - `UPDATE`: Updating an existing element.
  - `DELETION`: Removing an element.

* **`updateQueue`** (Object or Array): Holds pending state updates for the component represented by the fiber.

## Fiber Creation and Reconciliation**

* **`createFiber`**

Creates a new `FiberNode` instance with the following parameters:
- `tag`: The type of the fiber.
- `key`: The unique identifier for the element.
- `pendingProps`: The props passed to the component.
- `returnFiber`: The parent fiber.

* **`reconcileChildren`**

Responsible for reconciling the children of a component. It involves:
1. Iterating through the `children` prop (an array of elements).
2. Checking if an existing fiber can be reused for a child element.
3. Creating a new fiber with `createFiber` if no reusable fiber exists.
4. Recursively reconciling each child element.

The function returns the first child fiber created during reconciliation.

---


### Related Content

[Source on React Fiber explained ](https://blog.openreplay.com/react-fiber-explained/)
