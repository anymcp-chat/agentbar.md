# agentbar.md

**v0.1 (draft)**. A small convention that lets several chat bars share one web page without clashing.

A *chat bar* here is an embeddable widget with two surfaces:
- **Pill**: a small resting box in the bottom-right corner.
- **Panel**: the chat, which opens from the pill. On wide screens it docks as a right-hand column with the page pushed aside. On narrow screens it covers the screen as a full-screen sheet.

A page can carry more than one: say, an owner's site-editing agent and the product's agent for its users. If each bar follows the four rules below, only one panel is open at a time, a docked panel never covers another bar's pill, and pills stack instead of overlapping.

## Events

All coordination happens through `CustomEvent`s dispatched on `window`. Every `detail` carries `id`, a stable string unique to the bar (e.g. `"agentkeyboard"`, `"anymcp"`). A bar ignores events carrying its own `id`.

| Event | `detail` | Fired when |
|---|---|---|
| `agentbar:hello` | `{ id }` | the bar loads |
| `agentbar:open` | `{ id }` | the bar's panel opens |
| `agentbar:close` | `{ id }` | the bar's panel closes, even if the bar then hides entirely |
| `agentbar:dock` | `{ id, width }` | the bar's panel docks (`width` in CSS px) or undocks (`width: 0`) |
| `agentbar:pill` | `{ id, height }` | the bar's pill appears, changes height, or hides (`height: 0`); also in reply to `agentbar:hello` |

Add your listeners **synchronously, before** firing `agentbar:hello`, so bars that load right after yours are heard.

## Rules

1. **Take turns.** Fire `agentbar:open` *before* opening your panel. When another bar fires it, minimize your panel to its pill. If that bar isn't docked (its last `agentbar:dock` has no `width > 0`), its panel is a full-screen sheet: hide your pill too (fire `agentbar:pill` with `height: 0`) until that bar fires `agentbar:close`, `agentbar:pill` with `height > 0` or `agentbar:dock` with `width > 0`. Fire `agentbar:close` whenever your panel closes, including when your whole bar goes away with it (its pill never comes back, so that is the only signal). Don't treat `height: 0` as closed: an open sheet usually hides its own pill. A docked bar fires `agentbar:dock` before `agentbar:open`.
2. **Move aside for a docked panel.** While any other bar's last `agentbar:dock` has `width > 0`, shift your pill left by that width, so it sits at the corner of the page area beside the panel. Fire `agentbar:dock` yourself whenever your panel docks or undocks. A full-screen sheet isn't docked.
3. **Only the docked bar touches the page.** While docked, a bar may push the page aside by setting `margin-right` on `document.documentElement`. On undock, it clears the value only if no other bar has fired `agentbar:dock` with `width > 0` since it docked. Don't compare the value: two bars often dock at the same width, so a matching margin may be the other bar's. On a page that doesn't scroll (an app shell), `position: fixed` elements ignore that margin, so while docked there a bar also sets `width: auto; height: 100%; transform: translateX(0)` on `document.documentElement` to keep them in the page area. On undock it clears those only if no other bar's margin is set. Keep your own bar out of that transform by putting it in the top layer (`popover="manual"` + `showPopover()`). A bar that isn't docked never writes the page's styles, except for that clearing.
4. **Stack pills in load order.** Fire `agentbar:hello` on load, and answer every other bar's `hello` with your current `agentbar:pill` (and, while docked, your `agentbar:dock`, so a bar that loads after yours, e.g. on reload, still moves aside; while your panel is open as a full-screen sheet, `agentbar:open` instead, so it hides its pill per rule 1). Bars you heard from before your own `hello` (through their replies) are *earlier*; bars whose `hello` arrives later are *later*. Raise your pill's bottom offset by the sum of `height + 12px` over every earlier bar whose pill is showing (`height > 0`). Later bars stack above you; you don't move for them.

Each bar keeps its own base offset from the corner; the gap between stacked pills is 12px.

## Example

```js
const ID = "mybar";
const others = new Map(); // id → { order, height, dock }
let seq = 0;
const fire = (type, extra = {}) => dispatchEvent(new CustomEvent(`agentbar:${type}`, { detail: { id: ID, ...extra } }));

addEventListener("agentbar:hello", (e) => {
  if (e.detail.id === ID) return;
  if (!others.has(e.detail.id)) others.set(e.detail.id, { order: Infinity, height: 0, dock: 0 }); // later than us
  fire("pill", { height: pillHeight() });
});
addEventListener("agentbar:pill", (e) => {
  if (e.detail.id === ID) return;
  const o = others.get(e.detail.id) ?? { order: seq++, height: 0, dock: 0 }; // first heard as a reply: earlier
  others.set(e.detail.id, { ...o, height: e.detail.height });
  layout();
});
addEventListener("agentbar:dock", (e) => {
  if (e.detail.id === ID) return;
  const o = others.get(e.detail.id) ?? { order: Infinity, height: 0, dock: 0 };
  others.set(e.detail.id, { ...o, dock: e.detail.width });
  layout();
});
addEventListener("agentbar:open", (e) => {
  if (e.detail.id === ID) return;
  minimize();
  if (!(others.get(e.detail.id)?.dock > 0)) hidePillUntilItsPillOrDock(e.detail.id); // a full-screen sheet
});
addEventListener("agentbar:close", (e) => e.detail.id !== ID && showPillIfHiddenFor(e.detail.id));

function layout() {
  const below = [...others.values()].filter((o) => o.order !== Infinity && o.height > 0);
  pill.style.bottom = `${BASE_BOTTOM + below.reduce((s, o) => s + o.height + 12, 0)}px`;
  pill.style.right = `${BASE_RIGHT + Math.max(0, ...[...others.values()].map((o) => o.dock))}px`;
}

fire("hello");
```

## Adopters

- [Agent Keyboard](https://github.com/vibhasjain/agent-keyboard): all four.
- [anymcp](https://anymcp.chat): all four.

Open an issue or a pull request to propose a change or to add your bar.

## License

[CC0 1.0](LICENSE): public domain. Use it, copy it, change it.
