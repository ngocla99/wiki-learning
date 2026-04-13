# Reconciliation and Diffing

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

React's reconciliation algorithm determines which components need to be re-rendered, unmounted, or mounted between state updates. It works by comparing element definition objects position-by-position in the tree and using the `type` property to decide whether to update in place or destroy and rebuild. Understanding this process explains several non-obvious bugs, reveals why defining components inside other components is destructive, clarifies how the `key` attribute works (both in and outside of lists), and provides techniques for resetting state or forcing component reuse.

## The Mysterious Bug: Conditional Rendering Preserving State

Consider a sign-up form where the user toggles between "company" and "person" tax ID fields:

```jsx
const Form = () => {
  const [isCompany, setIsCompany] = useState(false);

  return (
    <>
      <Checkbox onChange={() => setIsCompany(!isCompany)} />
      {isCompany ? (
        <Input id="company-tax-id-number" placeholder="Enter your company Tax ID" />
      ) : (
        <Input id="person-tax-id-number" placeholder="Enter your personal Tax ID" />
      )}
    </>
  );
};
```

Expected behavior: switching the checkbox resets the input field (different entity, fresh state). Actual behavior: the text you typed **persists** across the toggle. React reuses the same `Input` instance instead of unmounting one and mounting the other.

To understand why, we need to understand how diffing works.

## How Diffing Works

### The Virtual DOM Tree

When you write JSX, React builds a tree of description objects. Each element becomes an object with a `type` property:

```jsx
const Input = ({ placeholder }) => {
  return <input type="text" id="input-id" placeholder={placeholder} />;
};

// The element <Input placeholder="Text1" /> becomes:
{
  type: Input,       // reference to the function
  props: { placeholder: "Text1" }
}
```

For DOM elements, `type` is a string:

```js
// <input type="text" /> becomes:
{
  type: "input",
  props: { type: "text" }
}
```

A component returning multiple elements produces an array:

```jsx
const Input = () => {
  return (
    <>
      <label htmlFor="input-id">{label}</label>
      <input type="text" id="input-id" />
    </>
  );
};

// Becomes an array of objects:
[
  { type: 'label', /* ... */ },
  { type: 'input', /* ... */ }
]
```

For React components (functions), React puts the function reference as the `type`:

```js
// <Input /> becomes:
{
  type: Input,  // the actual function reference
  // ...
}
```

### Initial Mount

On the first render, React walks the tree. For each node:
- If `type` is a **string** (e.g., `"div"`), React creates the corresponding HTML element.
- If `type` is a **function** (a component), React calls it and recursively processes whatever it returns.

This continues until the entire tree resolves to DOM nodes, which React appends to the document.

### State Update: Position-by-Position Comparison

When state changes, React re-renders the component and compares the "before" and "after" trees. For each position in the returned array:

**Same type at same position** -- React marks the component as "needs update" and re-renders it. All state and DOM nodes are preserved:

```jsx
// Before and after state update, the component returns:
{ type: Input, props: { placeholder: "Text1" } }
// Same type (Input) at same position --> re-render, state preserved
```

**Different type at same position** -- React unmounts the old component (destroying state, DOM, everything) and mounts the new one from scratch:

```jsx
// Before (isCompany = true):
{ type: Input, /* ... */ }

// After (isCompany = false):
{ type: TextPlaceholder, /* ... */ }
// Different type --> unmount Input, mount TextPlaceholder
```

## The Conditional Rendering Gotcha

Now the mystery is clear. In the buggy form:

```jsx
{isCompany ? (
  <Input id="company-tax-id-number" placeholder="Enter your company Tax ID" />
) : (
  <Input id="person-tax-id-number" placeholder="Enter your personal Tax ID" />
)}
```

The ternary occupies a **single position** in the children array. Before and after:

```js
// Before (isCompany = true):
{ type: Input, props: { id: "company-tax-id-number", /* ... */ } }

// After (isCompany = false):
{ type: Input, props: { id: "person-tax-id-number", /* ... */ } }
```

The `type` is the same (`Input` function reference). React concludes this is the same component with different props. It re-renders it in place. The internal state (typed text) survives.

## Why Components Inside Other Components Break Everything

This is one of the biggest performance killers in React:

```jsx
const Component = () => {
  const Input = () => <input />;
  return <Input />;
};
```

The element object is:

```js
{ type: Input }
```

But `Input` is defined **inside** `Component`. On every re-render, a new `Input` function is created. When React compares `type` before and after, it compares two different function references:

```js
const a = () => {};
const b = () => {};
a === b; // always false
```

React sees a different type at the same position and **unmounts** the old component, then **mounts** a new one. This happens on every single re-render. The consequences:

- **Performance**: Re-mounting is at least twice as slow as re-rendering (everything is created from scratch).
- **Flickering**: Heavy components briefly disappear and reappear.
- **Lost state**: Any state inside the inner component (input text, selected options, scroll position, focus) resets on every re-render of the parent.
- **Hard to debug**: The component "works" but state randomly disappears, focus is lost, and performance is terrible.

In code: if the inner component triggers re-renders (e.g., via typing in an input), and a sibling component has state, that state vanishes on every keystroke.

Always define components at the module level, never inside other components.

## Solving the Mystery: Arrays and Position

The form has a Fragment with children, which is an array:

```jsx
const Form = () => {
  return (
    <>
      <Checkbox onChange={() => setIsCompany(!isCompany)} />
      {isCompany ? (
        <Input id="company-tax-id-number" />
      ) : (
        <Input id="person-tax-id-number" />
      )}
    </>
  );
};
```

React processes this as an array:

```js
[
  { type: Checkbox },
  { type: Input },  // the conditional, always in position 1
]
```

Since both branches of the ternary produce `{ type: Input }` at position 1, React reuses the instance.

### Fix 1: Separate Positions

Put each input in its own position, using two separate conditionals:

```jsx
const Form = () => {
  const [isCompany, setIsCompany] = useState(false);

  return (
    <>
      <Checkbox onChange={() => setIsCompany(!isCompany)} />
      {isCompany ? <Input id="company-tax-id-number" /> : null}
      {!isCompany ? <Input id="person-tax-id-number" /> : null}
    </>
  );
};
```

Now the array always has three items: `Checkbox`, `Input` or `null`, and `Input` or `null`.

When `isCompany` changes from `false` to `true`:

```js
// Before:
[{ type: Checkbox }, null, { type: Input }]

// After:
[{ type: Checkbox }, { type: Input }, null]
```

Position-by-position:
- Position 0: `Checkbox` before and after -- re-render.
- Position 1: `null` before, `Input` after -- mount new Input.
- Position 2: `Input` before, `null` after -- unmount Input.

The inputs are in different positions, so they are treated as different components. State is reset.

### Fix 2: The `key` Attribute

Add unique keys to force React to treat them as different components:

```jsx
{isCompany ? (
  <Input id="company-tax-id-number" key="company-tax-id-number" />
) : (
  <Input id="person-tax-id-number" key="person-tax-id-number" />
)}
```

Before (`isCompany` = false):

```js
[
  { type: Checkbox },
  { type: Input, key: 'person-tax-id-number' },
]
```

After (`isCompany` = true):

```js
[
  { type: Checkbox },
  { type: Input, key: 'company-tax-id-number' },
]
```

The keys are different. React drops the old `Input` and mounts a new one. State is reset.

## Arrays and Dynamic Lists

### How React Processes Arrays

When React encounters an array of children, it iterates and compares "before" and "after" elements position by position. With conditionals like `isSomething ? <A /> : null`, the array always has a stable number of slots -- sometimes a slot holds `null`:

```jsx
<>
  <Checkbox />
  {isCompany ? <Input id="company" /> : null}
  {!isCompany ? <Input id="person" /> : null}
</>

// Always 3 items: Checkbox, (Input or null), (Input or null)
```

### The Key Attribute: Why It Matters

Dynamic lists (`.map()`) create arrays where items can be added, removed, or reordered. Without keys, React matches by position and gets confused:

```jsx
const data = ['1', '2'];
const Component = () => {
  return data.map((value) => <Input key={value} />);
};

// Produces:
[
  { type: Input },  // "1" data item
  { type: Input },  // "2" data item
]
```

If the array is reordered, without keys React just matches position 0 with position 0:

```js
// Before:
[{ type: Input }, { type: Input }]  // "1" then "2"

// After reorder:
[{ type: Input }, { type: Input }]  // "2" then "1"
// React doesn't know they swapped!
```

It re-uses the first instance for the new first item's data. If you typed in the first input, the text stays in position 0 -- even though the data moved.

With keys, React tracks identity:

```js
// Before:
[{ type: Input, key: '1' }, { type: Input, key: '2' }]

// After reorder:
[{ type: Input, key: '2' }, { type: Input, key: '1' }]
// React knows key='1' moved from position 0 to position 1
```

React swaps the DOM nodes. Typed text moves with its element.

### What Happens Without Keys When Items Are Added

If a new item is added at the beginning and you use array index as key:

```js
// Before: key=0 has "Business Tax", key=1 has "Person Tax"
// After adding "New Tax" at beginning:
// key=0 now has "New Tax", key=1 has "Business Tax", key=2 has "Person Tax"
```

React thinks key=0 just changed its props from "Business Tax" to "New Tax". Both existing items re-render, and a new item is mounted at the end. With proper id-based keys, React sees the existing items have unchanged keys/props and only mounts the new item.

## Key and Memoized Lists

A common misconception: `key` prevents re-renders. It does not. `key` helps React identify which existing instance to reuse. Re-renders still happen for every item when the parent re-renders.

To prevent re-renders, you need `React.memo`:

```jsx
const data = [
  { id: 'business', placeholder: 'Business Tax' },
  { id: 'person', placeholder: 'Person Tax' },
];

const InputMemo = React.memo(Input);

const Component = () => {
  return data.map((value, index) => (
    <InputMemo
      key={index}  // fine for static arrays
      placeholder={value.placeholder}
    />
  ));
};
```

For **static arrays** (items never change position), index as key is fine. `InputMemo` will not re-render because its props have not changed.

For **dynamic arrays** (items can be reordered, added, removed), index as key **breaks memoization**:

```js
// Before reorder: key=0 has placeholder="Business Tax"
// After reorder:  key=0 has placeholder="Person Tax"
// React thinks key=0's prop changed --> re-renders it!
```

Use a stable identifier instead:

```jsx
const Parent = () => {
  return sortedData.map((value) => (
    <InputMemo
      key={value.id}  // stable: "business" stays with "Business Tax"
      placeholder={value.placeholder}
    />
  ));
};
```

Now after reorder, `key="business"` still has `placeholder="Business Tax"`. React swaps DOM nodes but skips re-rendering because props have not changed.

The same applies when adding items. With index keys, existing items get new props (because their index changed) and re-render. With id keys, existing items keep their props and are skipped.

## State Reset Technique

The `key` attribute is not limited to dynamic arrays. It works on any element. Changing a component's key forces React to unmount and remount it, resetting all internal state.

From our Form example:

```jsx
{isCompany ? (
  <Input id="company-tax-id-number" key="company-tax-id-number" />
) : (
  <Input id="person-tax-id-number" key="person-tax-id-number" />
)}
```

Different keys at the same position: React destroys the old and creates the new. You do not even need two components -- one component with a changing key works:

```jsx
const Component = () => {
  const { url } = useRouter();

  // Input resets whenever the URL changes
  return <Input id="some-id" key={url} />;
};
```

**Caution**: This forces a full unmount/mount, which is more expensive than a re-render. For large components, this can cause performance problems. The state reset is a by-product of total destruction and reconstruction.

## Using Key to Force Reuse of an Existing Element

The inverse technique: same key at different positions tells React to reuse the existing instance. From the array fix:

```jsx
<>
  <Checkbox onChange={() => setIsCompany(!isCompany)} />
  {isCompany ? <Input id="company-tax-id-number" key="tax-input" /> : null}
  {!isCompany ? <Input id="person-tax-id-number" key="tax-input" /> : null}
</>
```

Before (`isCompany` = false):

```js
[
  { type: Checkbox },
  null,
  { type: Input, key: 'tax-input' },
]
```

After (`isCompany` = true):

```js
[
  { type: Checkbox },
  { type: Input, key: 'tax-input' },
  null,
]
```

React sees an element with `type: Input` and `key: 'tax-input'` in both trees. It concludes the component just moved positions. It reuses the existing instance, preserving state. Typed text survives the toggle.

This is more of a curiosity for simple cases but can be useful for performance-tuning components like accordions, tabs, or galleries where you want to preserve an existing instance across layout changes.

## Why We Don't Need Keys Outside of Arrays

React only requires keys on elements produced by `.map()` (dynamic arrays). But the object structure is the same:

```jsx
// Dynamic array:
const Component = () => {
  return (
    <>
      {data.map((value) => <Input key={value} />)}
    </>
  );
};

// Static elements:
const Component = () => {
  return (
    <>
      <Input />
      <Input />
    </>
  );
};
```

Both produce `[{ type: Input }, { type: Input }]`. The difference: React knows static elements have fixed positions that never change. Dynamic arrays might be rearranged, so React forces you to add keys as a precaution.

You *can* add keys to static elements for special effects:

```jsx
const Component = () => {
  const [isReverse, setIsReverse] = useState(false);

  return (
    <>
      <Input key={isReverse ? 'some-key' : null} />
      <Input key={!isReverse ? 'some-key' : null} />
    </>
  );
};
```

Toggling `isReverse` causes the key `'some-key'` to jump between the two inputs, making React swap instances.

## Dynamic Arrays and Normal Elements Together

A potential concern: if a dynamic array and static elements are siblings, does adding an item to the array shift static elements and cause re-mounting?

```jsx
const data = ['1', '2'];

const Component = () => {
  return (
    <>
      {data.map((i) => <Input key={i} id={i} />)}
      <Input id="3" />  {/* Will this re-mount if the array grows? */}
    </>
  );
};
```

If this flattened into `[Input, Input, Input]` and a new dynamic item was added, the static input would shift from position 2 to position 3 and potentially re-mount.

React is smarter than that. It nests the dynamic array as a single child in the parent array:

```js
[
  // The entire dynamic array is one item
  [
    { type: Input, key: '1' },
    { type: Input, key: '2' },
  ],
  { type: Input },  // static input, always at position 1
]
```

The static `Input` stays at position 1 regardless of how many items the dynamic array contains. No re-mounting. No performance disaster.

## Key Takeaways

- React compares elements **position-by-position** in the returned array at every level. First with first, second with second, etc.
- **Same type + same position** = re-render (state preserved). **Different type + same position** = unmount old, mount new (state destroyed).
- A conditional ternary (`a ? <X /> : <Y />`) occupies a **single position**, even when one branch is `null`.
- If both branches of a ternary have the **same component type**, React reuses the instance and preserves state. This causes bugs when you expect independent state.
- **Never define components inside other components**. The inner component gets a new function reference on every render, causing unmount/remount (destroying state, losing focus, tanking performance).
- For dynamic arrays, `key` tells React which instance to reuse across re-renders. Without proper keys, reordering causes state mismatches and unnecessary re-renders of memoized items.
- Use **stable unique identifiers** for keys (database IDs), not array indices, when items can be reordered, added, or removed.
- `key` does **not** prevent re-renders -- it only helps React track identity. Use `React.memo` to prevent re-renders.
- **State reset technique**: Changing a key forces unmount/remount, resetting all state. Useful for resetting uncontrolled components on context changes (e.g., URL changes).
- **Force reuse technique**: Same key at different positions makes React treat elements as the same instance, preserving state across layout changes.
- Dynamic arrays and static elements can coexist safely -- React nests the dynamic array as a single child.

## See Also

- [React Re-renders](react-re-renders.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [Memoization in React](memoization.md)
