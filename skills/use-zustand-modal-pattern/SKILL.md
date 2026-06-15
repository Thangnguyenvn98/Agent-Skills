---
name: use-zustand-modal-pattern
description: Standardize TypeScript and TSX modal work around a centralized Zustand modal store, a trigger component, and a dedicated modal component, including validated form-input modals and multi-step wizard modals. Use whenever adding, changing, refactoring, or reviewing a React modal or dialog in a TypeScript codebase that uses Zustand, especially with shadcn/ui or Radix Dialog.
---

# Use Zustand Modal Pattern

Apply the repository's existing naming, import, formatting, and component conventions while preserving the architecture below. Do not migrate unrelated modals or introduce a second modal state system.

## Required Structure

For each new or materially changed modal:

1. Add a unique `ModalType` value to the central modal store.
2. Add the modal payload fields to `ModalData`, or use the repository's existing typed payload map.
3. Open the modal from the trigger with `onOpen(type, data)`.
4. Put dialog markup and modal-specific behavior in a dedicated component.
5. Derive visibility with `isOpen && type === '<modalType>'`.
6. Mount one modal instance at a stable parent or shared modal-provider level.

Read [references/canonical-example.md](references/canonical-example.md) when implementing or reviewing code.
For a modal containing inputs, validation, or an API submission, also read [references/form-modal-example.md](references/form-modal-example.md).
For a modal that coordinates several steps, card selection, or wizard-style form flows, also read [references/multi-step-form-modal-example.md](references/multi-step-form-modal-example.md).

## Store Rules

- Keep `type`, `isOpen`, `data`, `onOpen`, and `onClose` in the central store.
- Type `onOpen` to accept its payload if the implementation accepts a payload.
- Default omitted payloads to an empty object.
- Reset `type`, `isOpen`, and `data` in `onClose` so stale payloads cannot leak into the next modal.
- Prefer selector calls such as `useModal(state => state.onOpen)` when only one field is needed.
- Reuse the existing store rather than creating feature-local `useState` for open state.

Use a discriminated payload map instead of a broad `ModalData` interface when the repository already uses one or when payload mistakes have become common.

## Component Rules

- Stop propagation in the trigger only when its parent has a conflicting click action.
- Pass runtime content through the store payload instead of importing placeholder data in the modal.
- Keep the modal controlled through Zustand.
- Handle Radix/shadcn close events with:

```tsx
onOpenChange={open => {
    if (!open) onClose()
}}
```

- Do not pass `onClose` directly to `onOpenChange`; that callback receives a boolean.
- Use `DialogClose asChild` with an actual `button` for icon-only close controls.
- Provide an accessible title, useful image alt text, and a description or explicitly set `aria-describedby={undefined}`.
- Prefer responsive `width`/`max-width` classes over fixed viewport-breaking minimum widths.
- If payloads contain `File` objects, create object URLs in a component or hook and revoke them during cleanup.

## Form Modal Rules

- Keep transient input values in React Hook Form, not in the Zustand modal store. Put only context needed to perform the action, such as `reviewId`, in the modal payload.
- Define a Zod schema and derive the form value type with `z.infer`.
- Supply complete `defaultValues` so inputs remain controlled.
- Reset the form after a successful submission and whenever a closed modal must reopen cleanly.
- Read payload identifiers before closing because `onClose` clears `data`.
- Close or open a success modal only after the mutation succeeds. Keep the form open and display an error when it fails.
- Disable submission while pending and prevent duplicate requests.
- Use `type="button"` on every non-submit button, including modal triggers rendered inside another form.
- Normalize Radix Checkbox values with `checked === true` before passing them to boolean schema fields.
- Match optional schemas to UI defaults. If an optional email field defaults to `''`, transform the empty string to `undefined` or explicitly allow it.
- Render every validation error. Use `FormMessage` inside a `FormField`; use an explicit `role="alert"` message for group/root errors.
- Prefer mutation callbacks or awaited promises over arbitrary `setTimeout` calls for modal transitions.
- Remove debug logging from completed code and preserve server error details through the repository's established error UI.

## Multi-Step Form Modal Rules

- Keep modal visibility in Zustand and keep the active step in the modal manager component.
- Type steps as a narrow union, such as `type FlowStep = 'choose-service' | 'details'`, or as numbers only when labels add no clarity.
- Reset the active step and any shared draft state in the same close handler that calls `onClose`.
- Guard `Dialog` close handling with `if (!open) handleClose()` so open-state callbacks do not accidentally reset the wizard.
- Compute open state with `isOpen && type === '<modalType>'`; use parentheses when a manager intentionally supports several related modal types.
- Let each step own its React Hook Form instance when schemas differ by step, but persist cross-step draft data in a typed draft store or parent state.
- Pass intent callbacks to child steps, such as `onNext`, `onBack`, `onSelect`, or `onComplete`, instead of passing raw `setState` when possible.
- Compose and submit the final payload inside the final step submit handler. Avoid a "set draft, set flag, submit in useEffect" sequence because it can submit stale draft data.
- Render titles and descriptions inside `DialogContent`; use `sr-only` for visually hidden accessible text.
- For card-selection steps, use real `button` elements, `aria-pressed`, clear selected state, and require a selection before advancing.
- Disable next/submit buttons while mutations are pending and keep the current step stable on server errors.

## Verification

After editing:

1. Run the repository's formatter, TypeScript check, and relevant tests.
2. Verify the trigger opens only the requested modal.
3. Verify overlay, Escape, and close-button actions call `onClose`.
4. Verify closing clears modal payload data.
5. Verify the modal remains usable at narrow viewport widths.
6. For form modals, verify validation, pending, success, failure, close/reopen reset, and duplicate-submit behavior.
7. For multi-step modals, verify step order, back navigation, close reset, reopen reset, card/branch changes, final submission payload, and recovery from failed submission.

Keep tests focused on changed behavior. Add store tests when changing shared modal state and interaction tests when changing user-visible dialog behavior.
