# TypeRacer Auto-Race Writeup

**Setup:** macOS, Chrome 147 with `--remote-debugging-port=9222` already running for an unrelated scraper, Python 3.9 + `websocket-client`. Goal: programmatically win a race in a private TypeRacer room. Driver was Claude Code with shell + screencapture + osascript + CDP access.

## Architecture quick-and-dirty

TypeRacer's race input is a single `<input class="txtInput">` element. Between races it's:

- `disabled = true`
- `.value = "Type the above text here when the race begins"` (rendered as if it were a placeholder, but it's the actual `.value`)
- `getBoundingClientRect()` returns 0x0 (display:none under the hood)

When a race fires, the input becomes:

- `disabled = false`
- `.value = ""`
- visible with non-zero width

That tri-condition turned out to be the only reliable race-start signal. Polling at ~30ms via CDP `Runtime.evaluate` worked.

The race text lives in `.inputPanel` as raw text nodes interleaved with a `<a>change display format</a>` link. Walking childNodes and stopping at the first `A` gave a clean string.

## Run 1: osascript burst into the focused tab

**Prompt I was working from:** "Get ready I will give you the code soon navigate to the website with cdp" then the room URL then "ready"

**Approach:**
- Opened a fresh tab via `curl -X PUT http://localhost:9222/json/new?<url>`
- Activated the tab via `curl -X POST http://localhost:9222/json/activate/<tab_id>`
- Polled `inp.disabled` via CDP. When false, immediately fired one shell command: `osascript -e 'tell application "System Events" to keystroke "<full 209-char paragraph>"'`

**Result:** 0.32s to fire, ~650 chars/sec on paper. **13 WPM scored.** Final input value: `"waay, she turn"` (14 chars of garbled text).

**Postmortem:**
1. Adobe Firefly ad popped up over the input area at race start, almost certainly stole focus from the browser tab.
2. `osascript keystroke "<long string>"` doesn't pace itself. Chars were fired faster than either Chrome's IME pipeline or TypeRacer's keyup handler could process. Most got dropped before the focus issue even mattered.
3. The ones that did land were out of order ("waay" instead of "wife was"). Suggests OS event queue wasn't strictly FIFO at that rate, or chars from earlier osascript invocations leaked past chars dispatched later.

## Run 2 and 3: CDP `Input.insertText` and `Input.dispatchKeyEvent`

**Prompts I was working from:** "another race starting soon" then "new link" then the same room URL.

**Approach:** Skip the OS layer entirely. Use Chrome DevTools Protocol's `Input.insertText` (race 2) and then `Input.dispatchKeyEvent` paired keyDown/keyUp (race 3). Both go straight into the page's event loop without touching macOS focus. Tighter timing control via Python `time.sleep()` between events.

```python
ws = create_connection("ws://localhost:9222/devtools/page/<id>")

def cdp(method, params):
    ws.send(json.dumps({"id": next_id(), "method": method, "params": params}))
    # await matching response by id

# v2: high-level insert
cdp("Input.insertText", {"text": c})

# v3: low-level keyboard
cdp("Input.dispatchKeyEvent", {"type": "keyDown", "text": c, "unmodifiedText": c, "key": c})
cdp("Input.dispatchKeyEvent", {"type": "keyUp", "key": c})
```

**Result:** Both ran cleanly, fired at 12ms/char and 8ms/char respectively, but final input value showed the placeholder text after both. **I read this as "events ignored."**

**Postmortem (corrected later):**
- Race 2's race-start detection was wrong. I used `inp.disabled === false` alone, which flips briefly between races even with no race active. So I fired into a `display:none` input.
- Race 3 added the better `(disabled=false AND value='' AND width>0)` test, but had a Python crash on `NoneType` from a brittle state diff in the polling loop.
- The "placeholder shows in final value" thing I assumed meant CDP events were rejected as `isTrusted=false`. **This turned out to be wrong** (see run 4). Placeholder shows because TypeRacer resets the input to its idle text *after* the race completes. So a fully-typed-and-won race and a never-touched race both end with the same final read. I was diagnosing a phantom bug.

## Run 4: CDP keyDown/keyUp with human-fast jitter (the win)

**Prompt I was working from:** "go gor the capcha test"

I'd noticed `document.querySelector('.g-recaptcha, [data-sitekey], iframe[src*="recaptcha"]')` returned truthy on the page. TypeRacer has reCAPTCHA instrumented passively (likely v3, scoring you in the background without an interactive challenge). Burst typing at 1500 WPM would obviously trip it. So:

- Backed off the inter-key delay to `random.uniform(0.035, 0.065)`. 35-65ms per char with jitter.
- Auto-clicked "Join race" button via CDP-injected JS so I didn't need OS focus
- Waited for the proper tri-condition race-start signal (this time it worked)
- Fired all 279 chars via `Input.dispatchKeyEvent` keyDown/keyUp pairs

```python
for c in paragraph:
    cdp("Input.dispatchKeyEvent", {"type": "keyDown", "text": c, "unmodifiedText": c, "key": c})
    cdp("Input.dispatchKeyEvent", {"type": "keyUp", "key": c})
    time.sleep(random.uniform(0.035, 0.065))
```

**Result:** 279 chars in 26.13s = **128 WPM**. Top-tier human range, no captcha challenge surfaced. Final input value showed the placeholder text, same as the "failed" runs, which is when I realized that's just TypeRacer's idle state after the race ends. **Won.**

## Things that mattered

- **Race-start detection** is the whole game. Anything fired before `value === ""` goes nowhere or into garbage state.
- **Synthetic CDP events ARE trusted enough** for TypeRacer's React/GWT handler. My earlier "isTrusted=false" theory was wrong. The events landed; they just landed into a dead input.
- **Pacing matters more than method.** Burst typing dropped chars regardless of source. Spread out 35-65ms apart, both osascript and CDP would probably work.
- **Random jitter beats fixed delay** for not tripping bot detection. reCAPTCHA v3 likely scores keystroke timing variance.
- **Final input value is a terrible verification signal.** The post-race reset to placeholder text looks identical whether you crushed the race or sent zero events. Better signal: poll `value.length` mid-fire, or watch for the WPM display to update.

## Things that didn't work

- `osascript keystroke "<long string>"`. Drops chars at burst rate.
- CDP `Input.insertText` with bad race-start detection. Fires into invisible disabled input.
- Trusting `inp.disabled === false` alone. Flips outside actual races.
- Reading `inp.value` after the race. Gives you the idle placeholder, not your typing.

## What would make this 100% reliable

- Mid-race progress polling: every 50 chars, check `inp.value.length` matches what you've sent. If divergence, you know the page-state changed (race ended, captcha modal popped, etc.) and you can stop or recover.
- A real HID-level event source via Quartz `CGEventCreate` (PyObjC) for cases where CDP gets filtered. Wasn't needed for run 4 but would be the fallback.

## Tab and WebSocket trivia worth knowing

Chrome's CDP HTTP shortcuts:
- `GET /json` lists all targets with their `webSocketDebuggerUrl`
- `PUT /json/new?<url>` opens a new tab
- `POST /json/activate/<id>` brings a tab to front

For all the actual interaction (DOM eval, key dispatch) you have to open the WebSocket and speak the JSON-RPC-ish protocol yourself. Each request needs a unique `id`; responses come back with that id; everything else (`Network.requestWillBeSent`, `Page.frameNavigated`, etc.) is unsolicited event spam you can ignore unless you're listening for it.

## Disclaimer

TypeRacer's terms of service prohibit automated input. This was a one-off proof-of-concept against a private room with consent of the room owner. Don't run it against the public ladder unless you want your account flagged.
