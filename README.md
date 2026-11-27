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
| `agentbar:dock` | `{ id, width }` | the bar's panel docks (`width` in CSS px) or undocks (`width: 0`) |
| `agentbar:pill` | `{ id, height }` | the bar's pill appears, changes height, or hides (`height: 0`); also in reply to `agentbar:hello` |

Add your listeners **synchronously, before** firing `agentbar:hello`, so bars that load right after yours are heard.

## Rules

1. **Take turns.** Fire `agentbar:open` *before* opening your panel. When another bar fires it, minimize your panel to its pill.
2. **Move aside for a docked panel.** While any other bar's last `agentbar:dock` has `width > 0`, shift your pill left by that width, so it sits at the corner of the page area beside the panel. Fire `agentbar:dock` yourself whenever your panel docks or undocks. A full-screen sheet isn't docked.
3. **Only the docked bar touches the page.** While docked, a bar may push the page aside by setting `margin-right` on `document.documentElement`. On undock, it clears the value only if it is still the one it set. A bar that isn't docked never writes the page's styles.
4. **Stack pills in load order.** Fire `agentbar:hello` on load, and answer every other bar's `hello` with your current `agentbar:pill`. Bars you heard from before your own `hello` (through their replies) are *earlier*; bars whose `hello` arrives later are *later*. Raise your pill's bottom offset by the sum of `height + 12px` over every earlier bar whose pill is showing (`height > 0`). Later bars stack above you; you don't move for them.

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
addEventListener("agentbar:open", (e) => e.detail.id !== ID && minimize());

function layout() {
  const below = [...others.values()].filter((o) => o.order !== Infinity && o.height > 0);
  pill.style.bottom = `${BASE_BOTTOM + below.reduce((s, o) => s + o.height + 12, 0)}px`;
  pill.style.right = `${BASE_RIGHT + Math.max(0, ...[...others.values()].map((o) => o.dock))}px`;
}

fire("hello");
```

## Adopters

- [Agent Keyboard](https://github.com/vibhasjain/agent-keyboard): rules 1 and 2.
- [anymcp](https://anymcp.chat): all four (in progress).

Open an issue or a pull request to propose a change or to add your bar.

## License

[CC0 1.0](LICENSE): public domain. Use it, copy it, change it.
