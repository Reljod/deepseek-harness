# Agent Note: The composer survives entry crashes and the workspace handoff

Status: implemented

English | [中文](2026-08-15-composer-survives-crashes-and-workspace-handoff.zh.md)

## Problem

Two defects made the Web composer stop accepting input, and both presented to users as "my typing does not appear".

Picking a workspace from the hero chip froze the draft. The pick runs `selectWorkspace`, which connects the workspace, moves the draft to the session it lands in, and opens that session; the picker list unmounts under the pointer while that runs, so focus falls to the document. Every keystroke after the pick reached no editable node, and because the composer paints its glyphs from a backdrop layer over a `color: transparent` textarea, the box kept showing the pre-pick draft. A user who typed one character, picked a workspace, and kept typing saw exactly that first character and nothing else, then sent it — one production session log records a submitted user message of exactly `"C"`.

Separately, any render error inside the composer bar removed the composer permanently. `SlotCore.reportEntryError` retired the crashed entry from its cell and never expired that retirement, so a `single`-kind cell holding one registration — which is what `conversation.composer.bar` is — went dry and rendered a bare `<div data-slot-error>` for the life of the page. The hero headline and workspace row are siblings of the bar and kept rendering, so the window looked normal with the input card missing, and the draft was unrecoverable without a reload.

## Decision

**Retirement requires a successor.** `reportEntryError` retires a crashed entry only when another live entry shares its cell. Retiring is a fall-through, and a cell with nothing to fall to gains nothing from it while permanently losing its surface. A lone crashed entry stays on its cell and its crash stays boundary-local.

**A crashed entry gets another render when its inputs change.** `SlotErrorBoundary` takes a `retryToken` and clears its failed state when that token changes. Session-maybe entries pass the current session id, because they deliberately survive session changes without remounting — the composer keeps its textarea across the hero-to-docked flip — so without this a crash would outlive every input that caused it. Root entries pass nothing and stay latched until they remount.

**The composer bar renders a session with no binding as its no-session state.** The bar's inject resolved the input shell through `InputHub.shell`, which throws when a session holds no binding. That inject runs inside the entry's error boundary, so the throw was a way for a lifecycle race to delete the only composer registration. `InputHub.maybeShell` answers `undefined` instead, and the inject takes the same branch it already had for "no current session". A live session scope that lost the `conversation` service still fails loud: that is a miswired tree, not a torn-down session.

**The workspace handoff claims focus back.** `ui-conversation` owns a `FocusClaim`: `selectWorkspace` raises one naming the session the draft moved to, and the bar of that session takes focus and puts the caret at the end of its draft. The claim names its session because the bar remounts on the switch it follows — a mount-time baseline would make the new incarnation treat an already-raised claim as old. The claim survives the several renders the handoff takes (outgoing draft, incoming empty draft, then the carried value, each of which re-collapses the caret) and closes when the carried draft has landed or the user types, whichever comes first.

The claim lives on the plugin fiber rather than in `ConversationRoot` because the resident skeleton remounts on the session switch it signals; component state would reset before the bar ever saw it.

## Alternatives considered

**Keep the throw and let the boundary catch it.** The throw is how the composer disappeared. A `session-maybe` slot advertises that it renders with or without a session, so its inject must tolerate the absence it advertises; failing loud there converts a transient lifecycle race into a permanently unusable application.

**Expire retirement on a timer, or retry crashed entries on every render.** Both reintroduce the crash loop that permanent retirement exists to prevent. Tying the retry to a change in the entry's inputs keeps recovery bounded by user-driven events.

**Remount the composer bar per session (key the boundary by session id).** That would clear a crash, but it also destroys the textarea across the hero-to-composer flip, which the session-maybe adoption contract exists to prevent. Resetting boundary state without remounting the subtree keeps both properties.

**Hold the focus claim in `ConversationRoot` as component state.** Tried first and measured: the skeleton remounts on the workspace switch, so the raised token was gone before the bar rendered, and the caret stayed at 0 while typing inserted in front of the carried draft.

**Re-read the outgoing draft after the switch to catch late keystrokes.** The handoff window still exists — a keystroke landing in the outgoing shell in the same tick is dropped. Restoring focus removes the case that loses every subsequent keystroke; chasing the residual sub-frame window needs a two-phase draft transfer whose complexity no current report justifies.

## Testing

`packages/client/ui-slots/tests/core.client.spec.ts` covers the retirement policy: retire with a survivor, keep a lone entry, keep the last survivor of a partly-retired cell, judge list cells per id, and never retire a chain entry. `packages/client/web-react/tests/scoped-slots.client.spec.tsx` proves a session-maybe entry that crashes for one session renders again for the next instead of latching its crash face; its fake host mirrors the ledger policy. `packages/client/ui-conversation/tests/apply-inject.client.spec.tsx` pins the bar's inject answering an unbound session with the no-session faces while a service-less live scope still throws. `packages/client/ui-conversation/tests/input-bar.client.spec.tsx` covers the claim: focus and end-of-draft caret, ignoring another session's claim, answering a claim raised before the incarnation mounted, holding across the handoff's renders, releasing on typing, and never focusing a locked composer.

## Consequences

A render error in the composer now costs one crash face that the next session switch clears, instead of an application with no way to type. The cost is that a genuinely broken entry retries rather than staying retired, so a deterministic crash reappears once per session change and logs each time.

`ComposerFocusClaim` adds a member to the composer bar's hooks compartment, so every test double constructing those props supplies it.

The composer's glyphs come from a backdrop layer over a transparent textarea. Any future defect that stalls the published draft while the DOM keeps accepting input will present as invisible text rather than a dead control, which is what made the original reports hard to read.
