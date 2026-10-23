# MTG Whiteboard

Single static file: `index.html` (HTML + minimal CSS + vanilla JS). Open it directly in a browser; no build, no dependencies, no server.
Purpose: a white, fullscreen-friendly whiteboard for presenting Magic: The Gathering card interactions.

## Constraints

- No frameworks or extra libraries; keep CSS minimal; white theme.
- Controls must be unobtrusive (toolbar is 25% opacity until hover/focus).
- Card data comes from the Scryfall API (`/cards/named?fuzzy=`, `/cards/autocomplete`).
- Every button shows its shortcut in its label.

## Features and shortcuts

| Key | Action |
| --- | --- |
| `/` | Focus card-name input (Enter adds the card) |
| `A` | Arrow mode: drag from a spot on a card to a spot on a card |
| `B` | Box mode: drag a highlight rectangle on a card |
| `T` | Text mode: click to create a note (anchored if clicked on a card) |
| `R` | Turn over the selected two-faced card (updates the URL) |
| `Z` | Undo (plain `Z`, not Ctrl/Cmd) |
| `Del`/`Backspace` | Delete selection |
| `F` | Fullscreen |
| `Esc` | Leave current mode (or finish editing a note) |

Move mode (default): drag cards/notes; click an arrow to select it, then drag its endpoint handles to re-anchor; double-click a card to flip image/oracle text; double-click a note to edit.

## Architecture

- Global `state = { cards, arrows, notes }`, fully JSON-serializable. Undo = stack of `JSON.stringify(state)` snapshots (`history`). Call `commit()` BEFORE mutating, or `history.push(before)` after a drag finishes. Anything new that is undoable must live in `state`.
- `render()` rebuilds card and note DOM from state; `drawArrows()` redraws SVG lines (also called during drags for performance).
- Anchoring: arrows use `{c: cardId, x, y}` (or `{n: noteId, x, y}` for notes) endpoints relative to the card/note, resolved by `anchorAt()`; `pruneArrows()` drops arrows whose target was deleted (`pt()` converts to board coordinates). Boxes live in `card.boxes` (card-relative). Notes have `c` (card id or null for free notes) and card-relative or absolute `x,y`.
- Selection `sel` is a string: `c<id>`, `a<index>`, `b<cardId>:<index>`, `n<id>`.
- Pointer handling is in one `pointerdown`/`pointermove`/`pointerup` set on `#board`, dispatching on `drag.type` (`move`, `arrow`, `box`, `handle`, `note`, `text`). The SVG overlay has `pointer-events: none`; arrows are hit-tested geometrically (`arrowAt`).
- Note editing uses `contenteditable`; the `blur` handler commits the text and pushes undo history. Note keydown stops propagation so global shortcuts don't fire while typing.

## URL sharing and two-faced cards

- The URL is the only persistence: `?c=<name>&c=<name>` lists the cards (names only; no positions, arrows, notes, or boxes). `syncUrl()` runs from `render()` via `history.replaceState` (note `history` is the undo stack, so use `window.history`); it is paused while the URL's cards are loading.
- Cards store `faces: [{name, img, oracle}]` and the current face index `f`. Scryfall returns `card_faces` for double-faced/split cards; each face may have its own `image_uris`. The URL stores the CURRENT face name, and `addCard` picks the face whose name matches the requested name (so `c=Soporific Springs` shows the back of `Sink into Stupor // Soporific Springs`). `R` cycles `f` (undoable). Boxes/arrows stay at their position when turning.

## Known limitations

- No persistence: reloading clears the board.
- Notes anchor to the topmost card under the pointer when dropped (free if none).
- Not tested in an automated way; verify manually in a browser (syntax check: extract the script and run `node --check`).

## Caution: interaction edge cases
Features overlap (cards, notes, boxes, arrows, handles all share one pointer pipeline), so check combinations whenever you change something. Past bugs:
- Arrows attached to a note did not follow it while dragging: every drag branch that moves an anchor target must call `drawArrows()` in `pointermove`, not only on `pointerup`.
- Arrow handles under a note could not be grabbed: hit-test order in `pointerdown` matters (selected-arrow handles > notes > arrows > cards). Keep new hit-tests in the right priority.
- Deleting or re-anchoring a card/note must keep dependent data valid (`pruneArrows()`, note re-anchoring keeps the same screen position).

Before finishing a change, mentally (or manually) test: dragging, undo, delete, and re-anchoring for each object type against every other type (e.g. arrow to note on a card, note moved onto/off a card, card moved with notes and arrows attached).
