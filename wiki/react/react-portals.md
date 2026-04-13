# React Portals

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

React Portals allow rendering children into a DOM node that exists outside the parent component's DOM hierarchy, while preserving the React component tree. Contrary to popular belief, Portals are **not** primarily needed to escape `overflow: hidden` (pure CSS can handle that). They are needed to escape **Stacking Context** traps — an inescapable CSS mechanism that no `z-index` value can overcome. This article covers the full CSS positioning model, why absolute and fixed positioning fail in real applications, how Stacking Contexts accumulate silently, and how Portals solve the problem while introducing their own set of behavioral differences.

## CSS Absolute Positioning

### Basic Absolute Positioning

The `position` property supports two values that break away from normal document flow: `absolute` and `fixed`. With `absolute`, an element is removed from the layout and can be positioned using `top`, `left`, `right`, and `bottom`:

```css
.modal {
  position: absolute;
  width: 300px;
  top: 100px;
  left: 50%;
  margin-left: -150px;
}
```

```jsx
const ModalDialog = () => {
  return (
    <div className="modal">
      some additional info
    </div>
  );
};

const App = () => {
  const [isVisible, setIsVisible] = useState(false);

  return (
    <>
      <SomeComponent />
      <button onClick={() => setIsVisible(true)}>
        show more
      </button>
      {isVisible && <ModalDialog />}
      <AnotherComponent />
    </>
  );
};
```

This dialog appears in the middle of the screen with a 100px gap at the top.

### Absolute Is Not Truly Absolute

Despite its name, `position: absolute` is actually **relative** — relative to the **closest ancestor element** that has `position` set to any value (`relative`, `absolute`, `sticky`, or `fixed`).

In simple examples it works by accident: there are no positioned elements between the modal and the root. But if the dialog is rendered inside a `div` with `position: relative`, and that div is not in the middle of the page, the modal will be positioned in the middle of **that div**, not the screen.

```jsx
<div style={{ position: 'relative', marginLeft: '200px' }}>
  {/* Modal will be positioned relative to THIS div, not the viewport */}
  <ModalDialog />
</div>
```

For elements like tooltips or dropdown menus that should be positioned relative to their trigger, this behavior is actually useful — you can use `offsetLeft` and `offsetTop` to position them relative to the trigger element. But for modals that should be centered on screen, `absolute` positioning is unreliable.

## Understanding Stacking Context

### What Is Stacking Context?

The Stacking Context is a three-dimensional way of looking at HTML elements. In addition to X and Y dimensions (width and height), there is a **Z-axis** that determines what sits on top of what when elements overlap. If an element has a shadow that overlaps with surrounding elements, should the shadow render on top or underneath? The Stacking Context determines this.

### Default Stacking Rules

By default, elements are stacked in the order of their appearance in the DOM:

```html
<div>grey</div>
<div>red</div>
<div>green</div>
```

The green div is after the red, so it appears "in front" from the stacking perspective. Red is in front of grey. If you add a small negative margin so they overlap, you see this layering clearly.

However, elements with `position` set to `absolute` or `relative` are **always pushed forward** in the stacking order. Adding `position: relative` to the red div makes the green div appear under it, even though green comes later in the DOM:

```html
<div>grey</div>
<div style="position: relative">red (now on top of green)</div>
<div>green</div>
```

### z-index Within a Stacking Context

The `z-index` CSS property manipulates the Z-axis **within the same Stacking Context**. By default z-index is zero. A negative value pushes the element behind, a positive value brings it forward.

**"Within the same Stacking Context" is the key phrase.** If something creates a **new** Stacking Context, `z-index` becomes relative to that new, isolated context. It is a completely isolated bubble: the new context is controlled as a black box by the rules of the parent context, and what happens inside stays inside.

### What Creates a New Stacking Context?

The combination of `position` and `z-index` on the same element creates its own Stacking Context. This means:

```css
/* grey div — creates its own Stacking Context */
.grey {
  position: relative;
  z-index: 1;
}

/* red div — creates its own Stacking Context */
.red {
  position: relative;
  z-index: 2;
}
```

With this setup, the grey div and **everything inside it** (including a modal dialog with `z-index: 9999`) will appear **underneath** the red div. The modal's z-index only competes with siblings inside the grey Stacking Context — it can never escape.

Other properties that create a new Stacking Context include:
- `transform` (any value other than `none`) — widely used for CSS animations
- `opacity` less than 1
- `filter`
- `z-index` on Flex or Grid children
- `will-change` with certain values
- And many other properties

### Overflow and Absolute Positioning

Setting `overflow: hidden` on an element **will clip** absolutely positioned children — but only when combined with `position: relative` on that same element. Without the positioned ancestor, the absolute element is positioned relative to a higher ancestor and the overflow does not clip it.

## Position: Fixed and Its Limitations

`position: fixed` positions elements relative to the **viewport** rather than a positioned ancestor. This provides two advantages:
1. Modals/dialogs can be centered on screen regardless of where they appear in the DOM
2. Fixed elements **escape** `overflow: hidden` traps

```css
.modal {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
```

### Fixed Cannot Escape Stacking Context

Even `position: fixed` **cannot escape** the rules of Stacking Context. Nothing can. It is like a black hole — once a Stacking Context forms, everything within its gravitational reach is trapped.

If a parent div has `z-index: 1` and a sibling div has `z-index: 2`, a `position: fixed` modal inside the first div will still appear underneath the second div. No amount of `z-index` on the modal will help.

### Fixed Is Not Always Relative to the Viewport

`position: fixed` is actually positioned relative to the **Containing Block**, which happens to be the viewport most of the time. But if any ancestor has certain properties set (notably `transform`), it forms a new Containing Block, and the fixed element positions relative to **that ancestor** instead — the same problem as with `absolute`.

The `transform` property triggers this, and it is widely used for animation. So any CSS animation on a parent can silently break a `position: fixed` child.

## How Stacking Contexts Accumulate in Real Apps

Would a Stacking Context trap actually occur in a real app? Easily and frequently.

The prime candidates are:
- **Sticky headers**: `position: sticky` with `z-index` to stay on top during scroll
- **Animated sidebars/navigation**: `transform` for smooth expand/collapse transitions
- **CSS transitions**: The `transition` property combined with `transform`
- **Any third-party component** that uses these properties internally

Consider this realistic layout:

```jsx
const App = () => {
  const [isNavExpanded, setIsNavExpanded] = useState(true);
  const [isVisible, setIsVisible] = useState(false);

  return (
    <>
      <div className="header" />  {/* position: sticky; z-index: 2 */}
      <div className="layout">
        <div
          className="sidebar"
          style={{
            transform: isNavExpanded
              ? 'translate(0, 0)'
              : 'translate(-300px, 0)',
          }}
        >
          {/* navigation links */}
        </div>
        <div
          className="main"
          style={{
            transform: isNavExpanded
              ? 'translate(0, 0)'
              : 'translate(-300px, 0)',
          }}
        >
          <button onClick={() => setIsVisible(true)}>
            show more
          </button>
          {isVisible && <ModalDialog />}
        </div>
      </div>
    </>
  );
};
```

```css
.header {
  position: sticky;
  z-index: 2;
}

.main {
  transition: all 0.3s ease-in;
}

.sidebar {
  transition: all 0.3s ease-in;
}
```

The header stays on top during scroll (good), the sidebar animates smoothly (good), but the modal dialog in the main area is **completely busted**:
- It is no longer centered on screen (the `transform` on `.main` creates a new Containing Block)
- It appears under the sticky header when scrolling (the header's `z-index` creates a Stacking Context that sits above the main area)

This happens even though there are no explicit `position: relative` or `overflow: hidden` properties — just `sticky`, `z-index`, `transform`, and `transition`, all of which are common in modern CSS.

A quick test: open popular websites with sticky elements or animations, open Chrome DevTools, find a block deep in the DOM tree, set `position: fixed` with a high `z-index`, and move it around. Many major sites (Facebook, Airbnb, LinkedIn) have main areas that are Stacking Context traps.

## How React Portal Solves the Problem

The only way to escape a Stacking Context is to render the element **outside** the DOM elements that form the context. In vanilla JavaScript, this would be:

```js
document.getElementById('root').appendChild(modalDialog);
```

In React, we use `createPortal` from `react-dom`. It accepts two arguments:
1. **What** to teleport — a React Element (`<ModalDialog />`)
2. **Where** to teleport it — a DOM element (not an id, but the actual element)

```jsx
import { createPortal } from 'react-dom';

const App = () => {
  const [isVisible, setIsVisible] = useState(false);

  return (
    <>
      {/* ... the rest of the layout with the button */}
      <button onClick={() => setIsVisible(true)}>show more</button>
      {isVisible &&
        createPortal(
          <ModalDialog />,
          document.getElementById('root'),
        )}
    </>
  );
};
```

The modal is still "rendered" together with the button from the developer experience perspective. But in the DOM, it ends up inside the `#root` element, at the bottom — outside all the Stacking Context traps. Opening Chrome DevTools confirms the modal is not inside the `.main` div.

The dialog is now centered as it should be and appears on top of the header.

## Portals and the React Lifecycle

The fundamental rule of Portals:

> **What happens in React stays in React. Where React has no power, the behavior is controlled by the DOM rules.**

### Event Bubbling (React Synthetic Events)

From React's perspective, the portalled modal is part of the render tree of the component that created the `<ModalDialog />` element. React synthetic events (like `onClick`) bubble through the **React tree**, not the DOM tree.

This means an `onClick` handler on a parent `div` in the React tree **will** catch click events from inside the portal, even though the portal is in a completely different part of the DOM:

```jsx
const App = () => {
  return (
    <div onClick={() => console.log('caught click from portal!')}>
      <div className="main">
        {createPortal(
          <button>Click me</button>,
          document.getElementById('root'),
        )}
      </div>
    </div>
  );
};
// Clicking the button logs "caught click from portal!"
```

### Re-renders

If the parent component re-renders, all components inside the portal re-render too — exactly as they would without a portal. The portal does not change the React component hierarchy at all.

### Context

If the parent part of the app has access to a React Context, the portalled component has access to the **exact same Context**. Context follows the React tree, not the DOM tree.

### Unmounting

If the part of the app where the portal is created unmounts, the portalled component also disappears.

## CSS Inheritance Differences with Portals

Since the DOM parent changes, **CSS properties that cascade or inherit** behave differently. The portalled element inherits styles from its new DOM parent, not from the React parent.

```css
/* This will NOT work with portalled modal */
.main .dialog {
  background: red;
}
```

The `.dialog` is no longer a DOM descendant of `.main`, so the CSS selector does not match. Properties that inherit through the DOM tree (font-family, color, font-size, etc.) will come from wherever the portal is mounted (`#root`), not from `.main`.

## Native JavaScript Events vs React Events with Portals

Native DOM events dispatched via `addEventListener` bubble through the **DOM tree**, not the React tree. If you try to catch events from a portalled component using native event listeners on the React parent, it will not work:

```jsx
const App = () => {
  const ref = useRef(null);

  useEffect(() => {
    const el = ref.current;

    el.addEventListener('click', () => {
      // trying to catch events from the portalled modal
      // NOT GOING TO WORK — the modal is not a DOM descendant of this div
    });
  }, []);

  return (
    <div ref={ref}>
      {createPortal(<ModalDialog />, document.getElementById('root'))}
    </div>
  );
};
```

Similarly, `parentElement` on the modal element returns the portal container (`#root`), not the React parent element.

This distinction matters:
- **React `onClick`**: Follows React tree (works through portals)
- **`addEventListener('click', ...)`**: Follows DOM tree (does not work through portals)

## Form Submission and Portals

The `onSubmit` event on `<form>` elements is the **least obvious** caveat. Although `onSubmit` looks similar to `onClick` in JSX, the submit event is **not managed by React** — it is a native API and DOM event.

If you wrap the main part of your app in a `<form>`, clicking buttons inside a portalled dialog will **not** trigger the form's `onSubmit` event. From the DOM perspective, those buttons are outside the form:

```jsx
const App = () => {
  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      console.log('submitted'); // never fires from portal button clicks
    }}>
      <div className="main">
        {createPortal(
          <button type="submit">Submit</button>,
          document.getElementById('root'),
        )}
      </div>
    </form>
  );
};
```

**Solution**: If you need form submission inside a portal, put the `<form>` tag **inside** the portalled component:

```jsx
const ModalDialog = () => {
  return (
    <form onSubmit={(e) => {
      e.preventDefault();
      console.log('submitted'); // this works
    }}>
      <input type="text" />
      <button type="submit">Submit</button>
    </form>
  );
};
```

## Key Takeaways

- **`position: absolute`** positions an element relative to a positioned ancestor, not the viewport. It is "absolute" in name only.
- **`position: fixed`** positions relative to the viewport (usually), and can escape `overflow: hidden`. But it **cannot** escape Stacking Context.
- **Nothing can escape the Stacking Context.** Once you are trapped, it is game over. No `z-index` value, no matter how large, can break out.
- **Stacking Context is formed by** `position` + `z-index`, `transform`, `opacity < 1`, `filter`, and many other CSS properties. In real apps, these accumulate silently through sticky headers, animated sidebars, and third-party components.
- **React Portals** render elements outside their current DOM position, escaping the Stacking Context trap while preserving the React component hierarchy.
- **React behavior is preserved** through portals: synthetic event bubbling, re-renders, Context access, and unmounting all follow the React tree.
- **DOM behavior changes** with portals: CSS inheritance, native JavaScript events (`addEventListener`), `parentElement`, and form submission all follow the DOM tree (where the portal is mounted).
- **Form `onSubmit` is a native event**, not managed by React. Put the `<form>` inside the portalled component if you need form submission to work.

## See Also

- [React Re-renders](react-re-renders.md)
- [Reconciliation and Diffing](reconciliation-and-diffing.md)
- [React Context Performance](react-context-performance.md)
- [Closures in React](closures-in-react.md)
- [Refs and Imperative API](refs-and-imperative-api.md)
