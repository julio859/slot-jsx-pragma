# Slotting JSX Pragma Specification
A Declarative Nested Slottable Component System Using a Custom JSX Runtime (React-Focused)

========================================================================================

1. Goal
-------

Define a JSX pragma extension ("slotting macro") that allows React components to support
an \`asChild\` prop enabling semantics like:

    <IconButton asChild>
      <a href="/foo">Click me</a>
    </IconButton>

Transformation result:

    <a href="/foo">
      <Icon />
      <span>Click me</span>
    </a>

Requirements:

- Slotting logic runs inside a custom JSX runtime (pragma)
- Async children/components must be supported
- Nested slottables supported
- \`React.Children\` and \`React.isValidElement\` *may be used*
- \`React.cloneElement\` **must NOT be used**
- Must be composable with other custom JSX pragmas via a \`withSlot\` wrapper
- Provide a precomposed JSX runtime for convenience


2. Component Authoring API
--------------------------

Components opt into slotting by using two marker components:

- \`Slot\`: replaces the component's root when \`asChild\` is true
- \`Slottable\`: marks where the host element’s children will be inserted

Example:

    import { Slot, Slottable } from "@your-lib/slot";

    function IconButton({ asChild, children, ...rest }) {
      const Comp = asChild ? Slot : "button";

      return (
        <Comp {...rest}>
          <Icon />
          <span><Slottable>{children}</Slottable></span>
        </Comp>
      );
    }

Call site:

    <IconButton asChild>
      <a href="/foo">Click me</a>
    </IconButton>


3. Marker Components (Runtime No-Ops)
-------------------------------------

These components are identity markers. They do not execute special runtime logic.

    export function Slot(props) {
      return <>{props.children}</>;
    }

    export function Slottable(props) {
      return <>{props.children}</>;
    }


4. JSX Runtime Responsibilities
-------------------------------

The custom \`jsx\`/\`jsxs\` functions must:

### 4.1 Intercept \`<Slot>\` Elements

When \`type === Slot\`, the pragma must:

1. Locate the nested \`<Slottable>\` element inside \`props.children\`.
2. Extract its child element — this is the **host element** (e.g. \`<a>\`).
3. Extract:
   - hostElement.type
   - hostElement.props
   - hostElement.props.children (host children)
4. Replace \`<Slottable>\` with the host's children.
5. Merge props:
   - outer props (from \`IconButton\`)
   - host element props (excluding children)
6. Create a new element using:
   - type = hostElement.type
   - props = mergedProps
   - children = rewritten template

### 4.2 Pass Through Non-Slot Elements

If \`type !== Slot\`, return \`baseJsx(type, props, key)\` unchanged.

### 4.3 Slottable Behavior

\`<Slottable>\` is removed during slotting and replaced with the host's children.

When not under a \`<Slot>\`, it renders its children unchanged at runtime.


5. Helper Functions (React APIs Allowed)
----------------------------------------

The implementation should use:

- \`React.Children.forEach\`
- \`React.Children.toArray\`
- \`React.isValidElement\`

These help detect elements and iterate children safely.

Helpers required:

### 5.1 \`findHostFromSlottable(children)\`

Walks a child tree and finds:

    <Slottable>
      {children}
    </Slottable>

Returns the \`children\`. Throws if:

- no slottable is found
- slottable contains more than one child
- child is not a valid React element

### 5.2 \`replaceSlottableWithHostChildren(tree, hostChildren)\`

Walks the tree and replaces:

    <Slottable>{children}</Slottable>

with:

    children.props.children

### 5.3 Rebuilding Elements

Do *not* use \`React.cloneElement\`.

Instead rebuild elements via:

    baseJsx(element.type, { ...newProps, children: newChildren })


6. The \`withSlot\` Pragma Wrapper API
--------------------------------------

Signature:

    export function withSlot(baseJsx: JsxFactory): JsxFactory;

Also required:

    export function withSlotJsxs(baseJsxs: JsxFactory): JsxFactory;

Behavior:

1. If \`type !== Slot\` → delegate to \`baseJsx\`.
2. If \`type === Slot\` → perform the full slotting macro rewrite (as described above).
3. Must not mutate \`baseJsx\`.
4. Must be composable with other wrappers:

    const jsx = withCssProp(withSlot(baseJsx));
    const jsxs = withCssProp(withSlot(baseJsxs));


7. Pre-Composed JSX Runtime for Users
-------------------------------------

Provide:

    import {
      jsx as baseJsx,
      jsxs as baseJsxs,
      Fragment,
    } from "react/jsx-runtime";

    import { withSlot } from "./withSlot";

    export const jsx = withSlot(baseJsx);
    export const jsxs = withSlot(baseJsxs);
    export { Fragment };

Consumers who don’t need to compose pragmas can set:

    {
      "compilerOptions": {
        "jsx": "react-jsx",
        "jsxImportSource": "slot-jsx-pragma/react"
      }
    }


8. Expected Macro Output Example
--------------------------------

Given:

    <IconButton asChild>
      <a href="/foo">Click me</a>
    </IconButton>

\`IconButton\` renders:

    <Slot {...rest}>
      <Icon />
      <span>
        <Slottable>
          <a href="/foo">Click me</a>
        </Slottable>
      </span>
    </Slot>

The pragma must output:

    jsx("a", {
      href: "/foo",
      ...rest,
      children: [
        <Icon />,
        <span>Click me</span>
      ]
    });


9. Edge Cases
-------------

- Only one \`Slottable\` is supported per \`Slot\`.
- Slottable must contain exactly one React element.
- Host children replace the entire \`Slottable\` wrapper.
- Async host components are allowed; pragma does not execute host types.
- No \`React.cloneElement\` is allowed.
- If no \`Slottable\` exists under \`Slot\`, assume the Slot's children is the host element.


10. Deliverables for Implementation
-----------------------------------

The engineer/AI must deliver:

- \`Slot\` runtime marker
- \`Slottable\` runtime marker
- \`withSlot\` and \`withSlotJsxs\`
- Pre-composed runtime module
- Helper functions:
  - \`findHostFromSlottable\`
  - \`replaceSlottableWithHostChildren\`
- React-friendly element walkers using:
  - \`React.Children\`
  - \`React.isValidElement\`
- TypeScript typings
- Usage examples


11. Integration Examples
------------------------

### Simple use:

    {
      "compilerOptions": {
        "jsx": "react-jsx",
        "jsxImportSource": "slot-jsx-pragma/react"
      }
    }

### Composed use:

    import { withCss } from "@tokenami/css/jsx-runtime";
    import { withSlot } from "slot-jsx-pragma/react";

    import { jsx as baseJsx, jsxs as baseJsxs, Fragment } from "react/jsx-runtime";

    export const jsx = withCss(withSlot(baseJsx));
    export const jsxs = withCss(withSlot(baseJsxs));
    export { Fragment };

