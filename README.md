

# Project 4 Reverse Engineering Report: usecomputer

*   **Project Name:** usecomputer
*   **Repository:** [https://github.com/remorses/usecomputer](https://github.com/remorses/usecomputer)
*   **Project Category:** Desktop Automation CLI / Native Systems Tooling
*   **Date of Submission:** April 3rd, 2026

## 1. Project Overview and Key Components

### Repository Analysis Summary

`usecomputer` is a desktop automation tool built for agent-style workflows. At a high level, it gives an LLM or a script the same small set of actions a human would need to operate a machine: take a screenshot, move the mouse, click, drag, scroll, type text, press shortcuts, and inspect the current display or window state.

What makes the repository interesting is the way it splits responsibilities. The public TypeScript layer is intentionally thin. Files like `src/lib.ts`, `src/bridge.ts`, and `src/coord-map.ts` provide a typed, stable interface for JavaScript consumers, but the real work happens in Zig. The native layer in `zig/src/lib.zig` talks directly to platform APIs on macOS, Linux, and Windows, and the CLI in `zig/src/main.zig` uses that same native core instead of re-implementing the commands separately.

There are a few design choices that show up across the whole repository:

*   The package prefers explicit operations over generic command execution.
*   The JavaScript side normalizes inputs before they ever cross into native code.
*   Screenshots are treated as part of an agent loop, not just as raw image files.
*   Coordinate mapping is treated as a safety problem, not just a geometry problem.

In practice, that gives the project a fairly disciplined architecture:

*   `src/native-lib.ts` loads the platform-specific `.node` addon.
*   `src/bridge.ts` converts TypeScript calls into native-friendly payloads and unwraps native results into real exceptions.
*   `src/lib.ts` exposes the library API with safe defaults.
*   `src/coord-map.ts` handles the conversion between screenshot-space and desktop-space coordinates.
*   `zig/src/lib.zig` implements the native automation logic and returns structured success/error objects.
*   `zig/src/main.zig` provides the standalone CLI using the same core functions.
*   `zig/src/kitty-graphics.zig` adds inline screenshot delivery for agent environments that support Kitty Graphics.

The overall design is practical rather than abstract. The repository is not trying to be a generic automation framework with plugins and dynamic dispatch. It is trying to be a reliable execution layer for computer-use agents, and most of the safety decisions in the code follow directly from that goal.

## 2. Deep Reasoning Questions & Analysis

### Q1. `usecomputer`'s `bridge.ts` converts undefined values to null before sending to native code. Why is this null normalization necessary for JavaScript-to-Zig communication?

This normalization is there to make the boundary between JavaScript and Zig predictable.

On the TypeScript side, callers naturally omit optional values or leave them as `undefined`. On the Zig side, the input structs are built around explicit optionals with `null` defaults. Those are not quite the same thing. Inside regular JavaScript code, `undefined` and `null` are often treated interchangeably, but once values cross a native boundary that assumption becomes dangerous. A missing property, an `undefined` value, and an explicit `null` do not necessarily marshal the same way through N-API.

`bridge.ts` solves that by converting every absent optional field into a single concrete representation before it reaches native code. In other words, it turns "maybe omitted" into "explicitly empty." That matters because the Zig layer is written to interpret absence through `null`, not through JavaScript object shape quirks.

The important point is that this is not cosmetic cleanup. It is contract stabilization. By the time the request reaches the Zig addon, there is only one meaning for "no value provided," so the native code can keep its logic simple and deterministic.
### Related Code Pattern
```javascript
// From README examples - all optional fields normalized to null:
const screenshot = await usecomputer.screenshot({
  path: './tmp/shot.png',
  display: null,      // <- Normalized (not undefined)
  window: null,       // <- Normalized
  region: null,       // <- Normalized
  annotate: null,     // <- Normalized
})
```

References: [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [src/bridge.ts](https://github.com/remorses/usecomputer/blob/main/src/bridge.ts), [src/native-lib.ts](https://github.com/remorses/usecomputer/blob/main/src/native-lib.ts)

### Q2. `usecomputer` normalizes all mouse button operations through `unwrap*` helper functions instead of having each method handle errors directly. Why does this centralized error pattern prevent user code from mishandling failures?

The big advantage is that it prevents partial handling and silent failure.

The Zig layer returns structured result objects like `{ ok, error }` or `{ ok, data, error }`. If those raw objects were exposed directly to every JavaScript caller, then every caller would have to remember to check `ok`, inspect `error`, and convert failures into real control flow. In practice, some code would do that carefully and some code would not. That is exactly how automation code ends up continuing after a failed click or drag.

The bridge does not allow that. `unwrapCommand` and `unwrapData` convert the native result into one of two outcomes only:

*   success returns the expected value, or
*   failure throws a `NativeBridgeError`.

That design matters because exceptions are hard to ignore accidentally. A caller cannot casually keep going with a failed native operation unless it deliberately catches and suppresses the exception. It also means every error leaves the bridge with the same structure and with the native command name attached, which makes debugging a lot easier.

So the central helper functions are really enforcing one failure policy for the whole library. That is safer than letting each operation or each caller invent its own.

### Related Code Pattern
```typescript
// From library examples:
await usecomputer.click({
  point: usecomputer.mapPointFromCoordMap({
    point: { x: action.x, y: action.y },
    coordMap: usecomputer.parseCoordMapOrThrow(coordMap),
  }),
  button: action.button ?? 'left',
  count: 1,
})
// parseCoordMapOrThrow unwraps immediately; if coordMap is malformed, throws
// click() wraps Zig's error union; if native click fails, throws
```

References: [src/bridge.ts](https://github.com/remorses/usecomputer/blob/main/src/bridge.ts), [src/lib.ts](https://github.com/remorses/usecomputer/blob/main/src/lib.ts), [zig/src/lib.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/lib.zig)

### Q3. The `CoordMap` coordinate transformation uses proportional scaling with clamping rather than linear interpolation. Why does proportional scaling prevent AI agents from clicking outside the capture region?

Because the screenshot the agent sees is not always the same size as the real captured desktop region.

`usecomputer` routinely scales screenshots down so the image stays within a model-friendly size. Once that happens, the coordinates inside the image are no longer one-to-one desktop coordinates. A point at `(400, 220)` inside the screenshot only makes sense relative to the screenshot dimensions. To recover the real desktop point, the code has to scale proportionally from image space back into capture space.

That is why the mapping uses the ratio of the point inside the screenshot span and then applies that ratio to the captured desktop span. The code also clamps the result to the capture bounds. That second step is crucial: if the model produces `-5`, or overshoots the right edge by a few pixels, or lands on a rounding boundary, the result is still forced back inside the area that was actually captured.

This is what keeps the action aligned with what the model could see. Without the proportional transform, scaled screenshots would map badly. Without the clamp, small coordinate mistakes would turn into clicks outside the intended region.

### Related Code Pattern
```javascript
// From README: coordinate mapping with clamping
const coordMap = usecomputer.parseCoordMapOrThrow(screenshot.coordMap)
// coordMap = { captureX: 0, captureY: 0, captureWidth: 1600, captureHeight: 900, imageWidth: 1568, imageHeight: 882 }

const point = usecomputer.mapPointFromCoordMap({
  point: { x: 400, y: 220 },  // Agent's click on 1568px image
  coordMap,
})
// mapPointFromCoordMap internally clamps to [0, 1600] × [0, 900] before returning
// Prevents any click outside the original capture region

// Then safe to use with click:
await usecomputer.click({ point, button: 'left', count: 1 })
```

References: [src/coord-map.ts](https://github.com/remorses/usecomputer/blob/main/src/coord-map.ts), [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [src/coord-map.test.ts](https://github.com/remorses/usecomputer/blob/main/src/coord-map.test.ts)

### Q4. `usecomputer` implements 18 distinct methods (`click`, `drag`, `type`, `scroll`, etc.) as separate exported functions instead of a single `execute` method with a command string. Why does method-per-operation improve safety over dynamic dispatch?

Method-per-operation is safer because it makes the interface explicit, typed, and narrow.

Each exported function has a known payload shape, known defaults, and known validation rules. `click` accepts a point, button, and count. `scroll` accepts a direction and amount. `drag` accepts `from`, `to`, and an optional control point. That sounds simple, but it has a real effect on safety: invalid combinations are harder to express in the first place.

If the whole system were reduced to something like `execute("click", payload)` or worse, `execute(commandString)`, then correctness would depend on runtime parsing and convention. That would make it easier for generated code to send fields that do not belong together, miss required fields, invent unsupported commands, or bypass command-specific checks.

The current design removes that ambiguity. On the Zig side, the N-API module exports named functions one by one. On the TypeScript side, the public library mirrors those functions one by one. So the surface area is deliberately fixed. A caller can only do what the library explicitly exposes.

For a desktop automation tool, that is a safety feature. Fewer degrees of freedom at the interface means fewer chances to do the wrong thing with a malformed dynamic command.

### Related Code Pattern
```javascript
// Safe, type-checked, injection-proof:
await usecomputer.click({ point: {x: 400, y: 220}, button: 'left', count: 1 })
await usecomputer.drag(100, 200, 500, 600)  // Straight drag
await usecomputer.typeText({ text: "hello", delayMs: null })
await usecomputer.press({ key: "cmd+s", count: 1, delayMs: null })
await usecomputer.scroll({ direction: 'up', amount: 5, at: null })

// Each is a distinct function with its own type contract
// All errors are thrown consistently (via unwrap pattern from Q2)
```

References: [src/lib.ts](https://github.com/remorses/usecomputer/blob/main/src/lib.ts), [src/native-lib.ts](https://github.com/remorses/usecomputer/blob/main/src/native-lib.ts), [zig/src/lib.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/lib.zig)

### Q5. Why does `usecomputer` require a native Zig binary (built via `pnpm build:native` or `zig build`) instead of implementing desktop automation in pure JavaScript?

Because the core problem is not JavaScript-friendly. It is operating-system automation.

The repository needs to do things like:

*   synthesize real mouse and keyboard events,
*   capture screenshots from the desktop,
*   enumerate displays and windows,
*   talk to CoreGraphics on macOS,
*   talk to X11/XTest on Linux,
*   talk to `user32` and `gdi32` on Windows.

Pure JavaScript cannot do those jobs by itself in a portable and reliable way. At some point it must call native APIs, shell out to platform utilities, or depend on another native binding. `usecomputer` chooses to own that layer directly in Zig rather than hide it behind a heavier JavaScript abstraction.

That decision also matches the repo's deployment goals. The same build system produces:

*   a `.node` addon for Node-based consumers,
*   a standalone executable CLI,
*   and a C API shared library.

That would be much harder to do cleanly if the real implementation lived in JavaScript and depended on a runtime everywhere.

### System Diagram (from Repository Architecture)
```
┌─────────────────────────────────────────┐
│  JavaScript/TypeScript (src/)           │
│  - CLI parsing                          │
│  - Coordinate mapping (TypeScript)      │
│  - Agent library interface              │
└──────────────────┬──────────────────────┘
                   │ (N-API calls)
                   ▼
┌─────────────────────────────────────────┐
│  Zig Native Binary (zig/src/)           │
│  - XShm/CoreGraphics/GDI+ calls        │
│  - PNG encoding via libpng              │
│  - Mouse/keyboard event injection       │
│  - Compiled to native .so/.dll/.dylib   │
└─────────────────────────────────────────┘
```

References: [build.zig](https://github.com/remorses/usecomputer/blob/main/build.zig), [zig/src/lib.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/lib.zig), [src/bin.ts](https://github.com/remorses/usecomputer/blob/main/src/bin.ts)

### Q6. Why does `usecomputer` include both CLI and JavaScript library interfaces instead of just exposing a single API?

Because the repository is serving two different usage patterns that both matter.

The CLI is for direct automation in shells, scripts, and agent tool calls. It is especially useful in environments where the easiest thing is to run a command and read stdout. It also has one major practical advantage: the standalone binary can run without depending on a Node runtime.

The JavaScript library solves a different problem. It is for code that wants to call the tool in-process and compose it with its own control loop, which is exactly what the OpenAI and Anthropic examples in the README do. In those cases, spawning a subprocess for every mouse move, click, or screenshot would be clumsy and slower.

What is important is that this is not two separate implementations. The CLI and library are thin entry points over the same native core. The project is not duplicating logic; it is exposing the same capabilities in the two forms users are most likely to need.

### Related Code Pattern
```bash
# CLI usage (from README):
usecomputer screenshot ./shot.png --json
usecomputer click -x 400 -y 220 --coord-map "0,0,1600,900,1568,882"
usecomputer type "hello from usecomputer"
```

```javascript
// Library usage (from README examples):
import * as usecomputer from 'usecomputer'

const screenshot = await usecomputer.screenshot({
  path: './tmp/shot.png',
  display: null,
  window: null,
  region: null,
  annotate: null,
})

await usecomputer.click({
  point: usecomputer.mapPointFromCoordMap({
    point: { x: action.x, y: action.y },
    coordMap: usecomputer.parseCoordMapOrThrow(coordMap),
  }),
  button: action.button ?? 'left',
  count: 1,
})
```

References: [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [src/bin.ts](https://github.com/remorses/usecomputer/blob/main/src/bin.ts), [src/lib.ts](https://github.com/remorses/usecomputer/blob/main/src/lib.ts)

### Q7. Why does `usecomputer` include both screenshot scaling and Kitty Graphics Protocol support instead of just returning raw screenshots?

Because the repository is optimized for agent loops, and raw screenshots are not enough for that workflow.

Screenshot scaling solves the size problem. The code caps the longest edge at 1568 pixels so screenshots stay small enough to be practical for model input. That keeps the image detailed enough to be useful while avoiding the waste of sending full native-resolution captures when the model does not need them.

Kitty Graphics solves a different problem entirely: transport. In agent environments that support it, the screenshot command can emit the PNG inline to stdout, and a plugin can inject that image directly into the model context. That removes an extra read step and makes the screenshot available immediately in the same tool call.

So one feature controls image size and the other controls delivery mechanism. They are solving different bottlenecks in the same loop. Returning only raw screenshots would leave both problems unsolved.

### Related Code Pattern
```bash
# Without Kitty: agent must read file and base64 encode
usecomputer screenshot ./shot.png --json
# Output: { path: "./shot.png", coordMap: "...", ... }
# Agent then: fs.readFileSync('./shot.png', 'base64')

# With Kitty: image inlined in output
AGENT_GRAPHICS=kitty usecomputer screenshot --json
# Output includes Kitty Graphics Protocol escape sequences
# { path: "...", coordMap: "...", agentGraphics: true }
# Model sees rendered image directly
```

References: [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [zig/src/main.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/main.zig), [zig/src/kitty-graphics.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/kitty-graphics.zig)


### Q8. Why does `usecomputer` provide a "debug visualization" mode instead of trusting agents to validate clicks correctly?

Because coordinate mistakes are cheap to make and expensive to ignore.

An agent may believe it is clicking the right point, but the real target depends on several layers of transformation: display origin, screenshot scaling, optional region capture, rounding, and coordinate remapping. A human reviewer or a calling program may want proof of where the action will actually land before issuing the real click.

That is what `debug-point` provides. It takes the same coordinate input and the same coord-map, resolves the desktop target the same way a real pointer command would, takes a fresh screenshot, and draws a marker where the click would land. In other words, it validates the actual execution path instead of validating the agent's own reasoning about that path.

That is a sensible safeguard in an automation tool. If the system already knows how to reconstruct the real click target, it is better to visualize it than to trust that the caller got the geometry right.

### Related Code Pattern
```javascript
// Agent workflow with debug validation:
const screenshot = await usecomputer.screenshot({ ... })
const coordMap = usecomputer.parseCoordMapOrThrow(screenshot.coordMap)
const buttonPoint = analyzeAndFindButton(screenshot.imageBase64)

// Before clicking, validate:
const debugScreenshot = await usecomputer.debugPoint({
  point: buttonPoint,
  coordMap,
  // (CLI equivalent: usecomputer debug-point -x 400 -y 220 --coord-map "...")
})
// Agent inspects debugScreenshot to confirm red marker matches button

if (isMarkerOnTarget(debugScreenshot, buttonPoint)) {
  await usecomputer.click({ point: buttonPoint, button: 'left', count: 1 })
} else {
  console.error('Marker not on target, skipping click')
}
```

References: [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [zig/src/main.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/main.zig), [zig/src/lib.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/lib.zig)

### Q9. Why does `usecomputer` clamp coordinates at capture boundaries instead of allowing out-of-bounds clicks that might hit nearby UI?

Because the project treats the screenshot as the safe action boundary.

An out-of-bounds click is not a harmless extension of the current screenshot. It is an action in a part of the UI the agent did not actually observe. That creates exactly the kind of mismatch that desktop automation should avoid. If the model points slightly above a button, or slightly past the right edge of a panel, allowing the click to spill into nearby UI could hit an unrelated control, including something destructive.

Clamping is a conservative answer to that problem. It says: if the model overshoots, keep the action inside the captured rectangle. That keeps the click tied to the visible evidence the model was operating from.

The tests make that intent clear as well. The coordinate mapping code is expected to snap overshoots back to the last valid pixel in the capture area, not to extrapolate past it.

### Mathematical Formula
```
Given:
- capture bounds: [captureX, captureY] to [captureX + captureWidth, captureY + captureHeight]
- screenshot coordinate: (screenshot_x, screenshot_y)
- image dimensions: (imageWidth, imageHeight)

Transformation with clamping:
real_x = ((screenshot_x / imageWidth) * captureWidth) + captureX
clamped_x = max(captureX, min(real_x, captureX + captureWidth))

# Ensures: captureX ≤ clamped_x ≤ captureX + captureWidth
```

References: [src/coord-map.ts](https://github.com/remorses/usecomputer/blob/main/src/coord-map.ts), [src/coord-map.test.ts](https://github.com/remorses/usecomputer/blob/main/src/coord-map.test.ts)


### Q10. Why does `usecomputer` support multiple display indices (for multi-monitor setups) instead of assuming a single display?

Because the code is operating in real desktop space, not in a simplified single-canvas world.

The repository explicitly enumerates active displays and keeps track of their bounds and indices. Screenshots report which desktop index they came from, and commands can target a specific display. On macOS, the code also resolves windows to the display with the greatest overlap. That matters because a modern desktop often has:

*   more than one monitor,
*   monitors with different origins,
*   monitors arranged to the left or above the primary display,
*   windows that live on or span different screens.

If the tool simply assumed a single display starting at `(0, 0)`, then both screenshots and click mapping would break as soon as a user worked on a second monitor. The coordinate mapping tests already reflect this by covering non-zero and negative display origins.

In short, multi-display support is not an optional convenience here. It is part of making the automation coordinates mean the same thing on real machines that they mean in the code.

### Related Code Pattern
```bash
# Capture specific monitor
usecomputer screenshot ./display0.png --display 0
usecomputer screenshot ./display1.png --display 1

# Output includes desktopIndex for agent to track which monitor was captured:
# { desktopIndex: 0, coordMap: "0,0,1600,900,1568,882", ... }
# { desktopIndex: 1, coordMap: "2000,-400,3840,2160,1568,1288", ... }
```

```javascript
// Library usage:
const screenshot = await usecomputer.screenshot({
  display: 1,  // Capture second monitor only
  path: './external.png',
  window: null,
  region: null,
  annotate: null,
})
// screenshot.desktopIndex === 1 (confirms display 1 was captured)
// Agent uses screenshot.coordMap to translate clicks to display 1 coordinates
```

References: [README.md](https://github.com/remorses/usecomputer/blob/main/README.md), [zig/src/lib.zig](https://github.com/remorses/usecomputer/blob/main/zig/src/lib.zig), [src/coord-map.test.ts](https://github.com/remorses/usecomputer/blob/main/src/coord-map.test.ts)

## 3. Findings and Conclusion

The main finding from the repository is that `usecomputer` is built around controlled explicitness. The project avoids "smart" generic interfaces in favor of fixed native operations, fixed result shapes, and fixed coordinate handling rules. That makes the system less flexible in a superficial sense, but much safer for the kind of agent-driven automation it is designed for.

The most important theme across the codebase is that screenshots and actions are treated as one closed loop. Screenshots are scaled to practical sizes, their capture bounds are returned as a coord-map, pointer commands can map back into real desktop space, and `debug-point` exists to verify that mapping visually. Once that design is understood, many of the individual implementation choices make sense immediately: null normalization, clamped coordinate transforms, method-per-operation exports, and support for inline screenshot transport all serve that same loop.

So the repository is best understood not just as a desktop automation library, but as an execution layer for computer-use agents. Its architecture is shaped less by abstract API design and more by the practical realities of cross-platform input synthesis, screenshot alignment, and failure containment. That is why the code consistently favors explicit contracts over dynamic behavior, and it is also why the project feels more reliable than a looser automation wrapper would.

---
