# NOTES.md — Custom Components vs. shadcn/ui

## Scope

This compares the from-scratch `Modal`, `Tabs`, and `Disclosure` in `src/playground/`
against shadcn/ui's `Dialog` and `Tabs` (which wrap Radix UI primitives). Radix is the
actual engine under shadcn's styling layer — shadcn/ui is Radix + Tailwind classes you
own the source of. So "what shadcn handles" really means "what Radix's primitive
handles," which is the honest comparison to make.

---

## Gap 1: Portal mounting and stacking context

**What my Modal does:** Renders inline in the React tree, positioned with
`position: fixed` on the overlay. This works for a single modal at a top level, but it
inherits whatever `z-index`, `transform`, or `overflow: hidden` ancestors exist in the
DOM. A parent with `overflow: hidden` or a CSS `transform` (which creates a new
containing block) can silently clip or mis-position the overlay.

**What shadcn/Radix does:** `Dialog.Portal` mounts the overlay and content into a
`document.body`-level DOM node via `ReactDOM.createPortal`, sidestepping ancestor
stacking-context and overflow issues entirely. It also composes multiple open
dialogs/popovers correctly (nested Dialog + DropdownMenu + Tooltip don't fight over
z-index) because each primitive manages its own portal root and Radix tracks a global
layer stack.

**Concrete gap:** my version would break if used inside a component with
`overflow: hidden` or a CSS `transform` ancestor (e.g., a card with hover scale). A
production fix means adding `createPortal(overlay, document.body)` myself and building
layer-stack awareness for nested dialogs — non-trivial, and exactly what Radix
abstracts away.

---

## Gap 2: `aria-hidden` / `inert` on background content

**What my Modal does:** Traps *focus* inside the panel via a Tab/Shift+Tab handler, and
traps *keyboard* navigation. It does **not** mark the rest of the page as hidden from
assistive tech. A screen reader user navigating by heading or landmark shortcuts (not
Tab) can still "virtually" browse into content behind the modal, because that content
is still exposed in the accessibility tree.

**What shadcn/Radix does:** Applies `aria-hidden="true"` (or the newer `inert`
attribute where supported) to all sibling DOM subtrees outside the portal root while
the dialog is open, then removes it on close. This is the difference between focus
being trapped for *keyboard* users versus content being fully hidden for *all*
assistive-tech navigation modes (virtual cursor, browse mode, rotor).

**Concrete gap:** a screen reader user in NVDA/JAWS browse mode could read page content
behind my modal that they logically shouldn't be able to reach. Fixing this means
walking `document.body`'s direct children, toggling `aria-hidden`/`inert` on every
sibling of the portal root, and carefully restoring original values on unmount —
straightforward in isolation, easy to get subtly wrong with nested overlays.

---

## Gap 3 (bonus): Focus restoration edge cases

**What my Modal does:** Captures `document.activeElement` on open, restores it on
close via a simple ref. This works for the common case (button opens modal, focus
returns to that button).

**What shadcn/Radix handles that mine doesn't:**
- If the previously-focused element is **removed from the DOM** while the modal is
  open (e.g., the trigger was inside a list item that got deleted by an async action),
  naive restoration throws or silently no-ops. Radix falls back to a sane default
  (e.g., the nearest surviving ancestor, or document body) instead of failing quietly.
- Radix also guards against restoring focus into an element that has since become
  `disabled` or `hidden`, which my implementation does not check.

**Concrete gap:** my restoration is a single `.focus()` call with no existence/validity
check. A defensive version needs to verify the element is still `isConnected` and not
`disabled`/`aria-hidden` before calling `.focus()`, with a documented fallback target.

---

## Gap 4 (bonus): Tabs — orientation, RTL, and manual vs. automatic activation

**What my Tabs does:** Hardcodes horizontal navigation (`ArrowLeft`/`ArrowRight`),
manual activation (arrow moves selection immediately — matches the assignment brief),
and assumes LTR reading order.

**What shadcn/Radix Tabs handles that mine doesn't:**
- `orientation="vertical"` support, which remaps the active keys to
  `ArrowUp`/`ArrowDown` and removes `ArrowLeft`/`ArrowRight` from the tab's own key
  handling (so they don't accidentally fire in a vertical layout).
- RTL awareness — in an RTL document, Radix swaps the semantic meaning of "next/previous"
  so `ArrowLeft` moves to the next tab, matching reading direction, instead of always
  mapping literally to "previous DOM index."
- An `activationMode="automatic"` option (arrow key immediately activates and fires the
  panel's content, vs. `"manual"` where Enter/Space is required after arrowing) — both
  are valid per the APG pattern depending on cost of activation; mine only implements
  manual-select-on-arrow.

**Concrete gap:** my Tabs would need direction detection (`getComputedStyle(el).direction`)
and an orientation prop threaded through the key-handling switch to reach parity.

---

## What I kept correct in my version (no gap)

- `role`/`aria-*` wiring (`tablist`/`tab`/`tabpanel`, `dialog`/`aria-modal`,
  `aria-expanded`/`aria-controls`) matches the APG patterns exactly.
- Roving `tabindex` on Tabs (only the selected tab is in the natural Tab order) is
  implemented correctly — this is the detail most hand-rolled tab implementations get
  wrong (leaving every tab focusable via Tab, which is non-conformant).
- Disclosure correctly relies on native `<button>` semantics for Space/Enter instead of
  reimplementing key handling — reimplementing this is a common source of bugs (e.g.,
  handling `keyup` vs `keydown` incorrectly, or breaking screen-reader "activate"
  gestures that synthesize a click rather than a keydown).
- Escape-to-close and overlay-click-to-close in the Modal are both implemented and
  match APG guidance.

---

## Bottom line

Structurally my components are APG-conformant for the keyboard/ARIA contract the
assignment tests directly. The gaps are all in the *unglamorous, hard-to-test-by-hand*
layer: portal/stacking-context isolation, `inert`/`aria-hidden` sibling suppression,
defensive focus restoration, and internationalization/orientation flexibility. That's
exactly the value proposition of building on Radix rather than hand-rolling — you trade
some bundle size and abstraction for edge cases that are expensive to discover through
manual QA and only surface with screen readers, RTL locales, or unusual DOM nesting.
