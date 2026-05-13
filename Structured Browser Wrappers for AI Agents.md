# Structured Browser Wrappers for AI Agents

Integrating browser automation (Playwright/Selenium) into an AI agent is strongest when the browser tool behaves like a deterministic sensor—not like a “blob of text” producer. The core engineering move is to **prevent raw terminal output, free-form logs, or unstructured HTML from becoming the model’s primary input**, and instead interpose **structured wrappers** that query the *live rendered page* and return **bounded, typed JSON**.

This is not just about cleanliness. Modern LLMs can produce convincing but incorrect inferences (“hallucinations”), especially when forced to interpret noisy or ambiguous inputs. Empirically, LLMs’ unreliability around ungrounded outputs is a well-studied concern in the research literature, and is significant enough that multiple detection and mitigation approaches have been proposed. citeturn9search1turn10view2 When agents are exposed to raw HTML or CLI noise, they are pushed into exactly the failure mode you want to avoid: *guessing what the UI “probably” looks like* rather than verifying what the UI *actually* is.

Your repository already implements the end-state architecture: a suite runner that captures pages, performs deterministic DOM inspections, normalizes findings into a typed issue schema, and emits a stable JSON manifest designed for machine consumption (with guardrails like bounded sampling and atomic file writes). Separately, your design-audit pipeline builds intermediate JSON manifests from screenshot sets, enabling multimodal inspection without “free-styling” from pixels.

## Why raw HTML and terminal output break agent reliability

A browser automation tool can easily dump:

- the page’s HTML (or a fragment),
- console logs,
- grep-like outputs from failed locators,
- screenshots, or
- stacks of network events.

But **raw HTML or noisy logs are not runtime truth**. Modern web UIs are shaped by asynchronous data, layout engines, computed styles, responsive breakpoints, and often hydration/runtime behavior. As a result, the **same HTML** can yield different **rendered** UI states across viewport sizes, theme modes, or timing windows.

Playwright’s own documentation stresses that evaluation occurs in a separate environment: your automation code runs outside the browser, while `page.evaluate()` executes inside the page environment, specifically to bridge that gap and “bring results back” from the browser context. citeturn0search2 This separation is exactly what you exploit with structured wrappers: you query the browser for computed reality and return it as data, not prose.

Selenium supports the same pattern through its JavaScript execution interface: scripts run in the context of the selected frame/window, and return values are mapped into typed host-language objects (e.g., primitives, `WebElement`, lists). citeturn1search3

There is also a **security** angle. When you pass arbitrary webpage text or HTML to a model, you are implicitly letting untrusted content act like instructions. Prompt injection is widely recognized as a top risk in LLM applications; the entity["organization","OWASP Foundation","appsec nonprofit"] Top 10 for LLM Apps lists “Prompt Injection” as LLM01 and highlights how crafted inputs can manipulate model behavior. citeturn11search2 Structured wrappers reduce this attack surface by (a) minimizing raw text ingestion, and (b) enforcing a schema where fields are treated as data (selectors, numbers, booleans), not imperative instructions.

Tool-use research also converges on the same intuition: LLM capabilities improve when they can *act* and then incorporate structured evidence from external systems rather than relying purely on internal inference. citeturn10view0turn10view1

## Deterministic DOM inspection in a live browser

Deterministic DOM inspection means you stop asking the model to infer UI state from source or raw markup, and you instead ask the browser:

- **Where is the element (bounding box)?**
- **Is it visible (computed style, geometry)?**
- **Is content overflowing/clipped?**
- **Which style tokens are active?**
- **Does contrast meet accessibility thresholds?**

Two DOM APIs matter heavily here:

- `getComputedStyle()` returns resolved CSS property values after cascading and computation. citeturn0search0  
- `getBoundingClientRect()` returns a `DOMRect` describing an element’s size and position relative to the viewport. citeturn0search3

Your flagship implementation is the pattern below (excerpted from the uploaded `playwright_screenshot.py`), which runs a large `page.evaluate()` bundle to collect a structured “layout health” snapshot:

```js
// (excerpt) page.evaluate(() => { ... })
const nodes = Array.from(document.querySelectorAll("*")).slice(0, 2000);

// overflow sampling
for (const node of nodes) {
  const rect = node.getBoundingClientRect();
  if (rect.width > viewportWidth + 24) {
    overflowSamples.push({
      selector: toSelector(node),
      width: Math.round(rect.width),
      viewport_width: viewportWidth,
      overflow: Math.round(rect.width - viewportWidth),
    });
    if (overflowSamples.length >= 8) break;
  }
}

// clipping sampling based on overflow:hidden/auto/scroll parent
const parentStyle = window.getComputedStyle(parent);
if (/^(hidden|auto|scroll)$/.test(parentStyle.overflow)) {
  // compute rect & parent_rect and boolean clip flags
}
```

This has three important determinism properties:

1. **Empirical geometry**: it uses runtime rectangles (`getBoundingClientRect`) rather than guessing. citeturn0search3  
2. **Computed styles**: it checks visibility and overflow based on computed CSS. citeturn0search0turn2search0  
3. **Bounded output**: it caps traversal and samples (2000 nodes, up to 8 samples), which is essential for keeping tool outputs within an agent’s context budget (and for preventing “model DoS by DOM explosion”).

Your clipping logic correctly focuses on `overflow` states that can clip content. MDN notes that `overflow: hidden` clips content at the padding box (content still exists but is not visible), and scroll/auto creates a scroll container. citeturn2search0

### Contrast checks with explicit WCAG thresholds

Your layout collector also computes contrast ratios and emits `contrast_issues` as structured findings, including `ratio`, `font_size`, and `threshold`. The thresholds you encode reflect the WCAG minimum guidance: 4.5:1 for normal text and 3:1 for large text. citeturn0search6

That matters because contrast failures are a classic “looks fine to the model” bug: an LLM reading HTML has no reliable sense of the final rendered colors or opacity blending. Only the browser can supply computed colors and effective background context (your code walks ancestors to resolve a non-transparent background).

### Style token inspection as a “shared shell health probe”

You also implement a lighter-weight wrapper that focuses specifically on whether shared design tokens/stylesheets are applied. In `playwright_screenshot.py` proving style activation:

```js
return {
  has_background_token: rootStyle.getPropertyValue("--background").trim().length > 0,
  has_foreground_token: rootStyle.getPropertyValue("--foreground").trim().length > 0,
  next_stylesheet_count: sheetCount,
  sidebar_width: sidebarWidth,
  viewport_width: window.innerWidth,
};
```

This leverages the principle that computed styles (including custom properties) are the canonical ground truth for “has the app shell styling loaded.” citeturn0search0

Your TypeScript E2E test version (`shared-shell-styling.spec.ts`) mirrors the same idea:

```ts
const styleState = await getShellStyleState(page);
expect(styleState.backgroundToken).not.toBe("");
expect(styleState.foregroundToken).not.toBe("");
expect(styleState.nextSheetCount).toBeGreaterThan(0);
```

This is a strong demo for a presentation because it shows the same wrapper concept in two contexts: a suite capture tool and a CI-style Playwright test.

## Turning noisy signals into typed JSON issues

Once you can deterministically inspect the DOM, the next “agentic engineering” step is to **normalize everything into a small set of typed issue objects**, so the model never has to parse arbitrary output formats.

Your `_new_issue()` function acts as a schema constructor: it takes `code`, `severity`, `message`, plus optional `details` and a context object, and returns a normalized issue payload with stable keys (and prunes null fields):

```py
issue: dict[str, Any] = {
    "code": code,
    "severity": severity,
    "message": message,
    "route": route,
    "resolution": ctx.get("resolution"),
    "label": ctx.get("label"),
    "theme": ctx.get("theme"),
    "sidebar": ctx.get("sidebar"),
    ...
    "details": issue_details,
    "recommendation": CONSOLE_REMEDIATION.get(code, "..."),
}
```

This is exactly the move that converts “LLM as probabilistic parser” into “LLM as consumer of a typed RPC response.”

A subtle but important design detail: **lists are where you enforce ordering.** JSON objects are defined as *unordered* collections in the JSON standard. citeturn1search2 Even though most modern runtimes preserve insertion order when serializing, the standard definition is “unordered,” which means you should treat **your list ordering** (`issues[]`, `screenshots[]`, regressions) as your canonical deterministic sequence.

### Treat the wrapper output like an RPC contract

Playwright’s evaluation primitives are built to return serializable values from the page environment back to the automation environment. citeturn0search2 Your wrapper essentially turns the page into an RPC server:

- method: “collect_layout_health”
- response: JSON with fixed keys, bounded arrays, numeric fields
- errors: structured issues (`navigation-failure`, `screenshot-failed`, etc.)

Selenium’s `executeScript` interface has the same “RPC over JS” flavor; scripts run in the page and return typed values (elements/primitives/arrays) back to the driver. citeturn1search3

### Recommended schema enforcement (presentation-ready)

Your repo already embodies schema discipline through code structure. For a presentation, it’s often effective to show a formal schema to make the “typed machine interface” point explicit.

Here is a compact JSON Schema-style sketch you can include on a slide (illustrative, not exhaustive):

```json
{
  "type": "object",
  "required": ["base_url", "exit_code", "issues", "screenshots"],
  "properties": {
    "exit_code": { "type": "integer" },
    "severity_summary": {
      "type": "object",
      "properties": {
        "low": { "type": "integer" },
        "medium": { "type": "integer" },
        "high": { "type": "integer" },
        "critical": { "type": "integer" }
      }
    },
    "issues": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["code", "severity", "message", "details", "recommendation"],
        "properties": {
          "code": { "type": "string" },
          "severity": { "enum": ["low", "medium", "high", "critical"] },
          "message": { "type": "string" },
          "route": { "type": "string" },
          "details": { "type": "object" }
        }
      }
    }
  }
}
```

The didactic point: the model is no longer doing ad hoc parsing; it’s consuming a contract.

## Intermediate design specs for multimodal workflows

Your third topic (“Screenshot → JSON → Code generator”) is fundamentally an *intermediate representation (IR)* story: instead of mapping pixels directly to code, you create a structured spec that constrains what downstream steps are allowed to infer.

In your repo’s uploaded `design-audit.sh`, `build_manifest_from_pngs()` constructs a manifest by parsing screenshot filenames into structured items:

```py
pattern = re.compile(
  r"^(?P<label>.+)-(?P<resolution>\d+x\d+)-(?P<theme>light|dark)-(?P<sidebar>expanded|collapsed|settings)\.png$"
)

screenshots.append({
  "page": f"/{label}",
  "label": label,
  "resolution": match.group("resolution"),
  "theme": match.group("theme"),
  "sidebar": match.group("sidebar"),
  "path": png_path.name,
  "status": "pass",
  "issues": [],
  "layout": {},
})
```

This is a clean example of an intermediate spec because it does three things:

- It **names** each UI state (label + resolution + theme + sidebar).
- It defines **slots** where later stages can attach evidence (`issues[]`, `layout{}`) without inventing new structure mid-flight.
- It provides an anchor for *multimodal prompting* (“look at these images, but interpret them through this manifest”).

### Why “design tokens” fit naturally into the IR

When you talk about typography scales, spacing rhythms, multi-theme tokens, or breakpoint rules, you are describing exactly what design systems encode as “design tokens.” The entity["organization","World Wide Web Consortium","web standards body"] community-hosted Design Tokens Community Group explicitly frames design tokens as standardized pieces of design system data meant to be shared across tools and platforms. citeturn12search0turn12search1

So, for the “image → JSON layout spec” slide, you can position the JSON IR as “a token-and-layout aware spec,” not as raw pixel interpretation.

### Grid constraints as first-class spec fields

If your intermediate JSON encodes “12-column grid” constraints, that should be justified as a widely used layout convention, not an arbitrary model choice. For example, Bootstrap documents its grid as a “twelve column system,” and MDN also demonstrates how to implement a flexible 12-column system with CSS grid. citeturn13search0turn13search2

That gives you a strong citation-backed reason to say: “We encode grid constraints explicitly so the agent can’t hallucinate layout structure.”

### Visual regression as structured numeric evidence

Your `_compute_image_diff()` returns a structured result:

- `size_mismatch`
- `current_size`, `baseline_size`
- `changed_pixels`, `total_pixels`
- plus a numeric diff ratio

This is an archetypal “image → JSON signal” step. You are converting a subjective visual judgment into a numeric, bounded artifact that downstream logic (or an agent) can reason about deterministically.

## AI-native tool contract and guardrails in your suite runner

Your suite runner (`run_suite`) emits a single top-level JSON manifest with:

- `artifact_integrity` (expected vs missing/empty),
- `route_coverage` (expected vs captured, missing routes),
- `severity_summary`,
- deduplicated `issues[]`,
- `screenshots[]` each including `layout{}` and issues.

This is what “AI-native tool design” looks like in practice: one stable JSON document that can drive a loop, a dashboard, CI gates, or an AI triage agent.

Two guardrail patterns in your implementation are particularly presentation-worthy:

### Bounded sampling and bounded traversal

You cap DOM traversal and sample counts in the JS collectors (e.g., `.slice(0, 2000)`, break after 8). This is a deliberate context-budget defense: it prevents the browser output from overwhelming the model and reduces “needle in haystack” errors.

The design aligns with broader agent security guidance that warns against “unbounded consumption” and “excessive agency” patterns; OWASP’s LLM Top 10 includes categories that map closely to these concerns (e.g., prompt injection and unsafe design of tool/plugin surfaces). citeturn11search2

### Atomic manifest writes

Your manifest write strategy:

- creates a temp file (`tempfile.mkstemp`)
- writes JSON
- swaps into place (`os.replace`)

This matters because agents, CI, and dashboards often read manifests concurrently. Partial writes create ambiguous states that an agent will misinterpret.

Python documents `mkstemp()` as a secure temp file primitive (low-level, manual cleanup, designed to avoid race conditions). citeturn3search1  
Python also documents `os.replace()` as replacing the destination if it exists and (on POSIX) performing the rename atomically when successful. citeturn6view0

If you want a crisp “AI-native reliability” message: **an agent cannot reason reliably over artifacts that are themselves non-deterministic or partially written**.

## Walkthrough using your repo’s implementation

This walkthrough is written so you can lift it into speaker notes.

### Capture phase: run the suite and produce a JSON manifest

Your `playwright_screenshot.py` supports “suite mode” (multi-route, multi-variant). A typical invocation pattern is:

```bash
python3 scripts/playwright_screenshot.py \
  --suite \
  --base-url http://localhost:3099 \
  --out-dir /tmp/design-verify \
  --manifest /tmp/design-verify/manifest.json \
  --fail-on-severity medium
```

What happens conceptually:

1. The runner navigates each route.
2. It stabilizes the page (waits, clears overlays, checks styling readiness).
3. It captures screenshots per variant (theme/sidebar/extra).
4. For each screenshot, it calls `_collect_layout_health()` (DOM → JSON).
5. It normalizes findings via `_new_issue()` (typed issue objects).
6. It emits a single manifest JSON document.

If you want to tie this to Playwright’s conceptual model: audit logic runs outside the page, but all DOM inspection logic runs inside the page and returns serializable values back, via `page.evaluate`. citeturn0search2

### Inspect phase: treat the manifest as your “single source of runtime truth”

In a terminal (or a CI step), you can extract summaries without inventing new parsing logic.

Examples:

```bash
jq '.severity_summary' /tmp/design-verify/manifest.json

jq '.issues[] | {code, severity, route, resolution, message}' /tmp/design-verify/manifest.json
```

Because you enforced a strict issue shape, you avoid a common agent trap: “scan raw logs and guess what matters.”

### Agent phase: give the model JSON, not the page

A “safe prompt payload” approach:

- Provide the manifest (or a filtered slice: only issues ≥ medium and only top N samples).
- Provide the relevant screenshots as separate multimodal inputs.
- Provide a tool that can fetch source code context (grep/ripgrep) *only after* the manifest points to a route/selector/code.

This mirrors the ReAct-style pattern: interleave reasoning with environment interaction so the model’s claims stay grounded in tool evidence rather than free-form inference. citeturn10view0

### Iterate phase: rescan only failing routes

Your suite supports a “rescan only failed routes” workflow (based on a prior manifest). That turns a broad crawl into a tight feedback loop:

```bash
python3 scripts/playwright_screenshot.py \
  --suite \
  --base-url http://localhost:3099 \
  --out-dir /tmp/design-verify-rescan \
  --rescan-only /tmp/design-verify/manifest.json
```

This is a strong “agentic engineering” story: the tool itself narrows scope deterministically, rather than relying on the agent to pick routes heuristically.

### Multimodal audit phase: build an intermediate screenshot manifest

Your `design-audit.sh` includes a fallback mode where it can synthesize a manifest directly from PNG filenames (useful when reusing captures). This is the “Screenshot → JSON intermediate spec” step.

The key presentational point: in your pipeline, **images are never “floating context.”** They are indexed by a structured manifest, which defines the allowed interpretation space (route label, theme, breakpoint). That reduces both accidental drift and injection risk, because the agent is constrained to talk about *known artifacts*, not random page text. citeturn11search2

### Optional: show the Selenium analog on one slide

To connect your architecture to Selenium, show the conceptual equivalent (pseudocode):

```python
# Selenium: execute JS in-page and return structured data
layout = driver.execute_script("""
  const el = document.querySelector(arguments[0]);
  if (!el) return { present: false };
  const rect = el.getBoundingClientRect();
  const cs = window.getComputedStyle(el);
  return {
    present: true,
    rect: { left: rect.left, top: rect.top, width: rect.width, height: rect.height },
    display: cs.display,
    visibility: cs.visibility,
    opacity: cs.opacity,
  };
""", "[data-slot='sidebar']")
```

Then cite Selenium’s official contract: scripts execute in the page context, and return values are converted to host objects per documented mapping. citeturn1search3

That makes your thesis general: **Playwright and Selenium are execution engines; the structured wrapper is the agent-facing API.**