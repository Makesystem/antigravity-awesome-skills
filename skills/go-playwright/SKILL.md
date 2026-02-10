---
name: playwright-go-automation
description: Expert capability for robust, stealthy, and efficient browser automation using Playwright Go.
---

# Playwright Go Automation Expert

## Overview
This skill provides a comprehensive framework for writing high-performance, production-grade browser automation scripts using `github.com/playwright-community/playwright-go`. It enforces architectural best practices (contexts over instances), robust error handling, structured logging (Zap), and advanced human-emulation techniques to bypass anti-bot systems.

## When to Use This Skill
- Use when the user asks to "scrape," "automate," or "test" a website using Go.
- Use when the target site has complex dynamic content (SPA, React, Vue) requiring a real browser.
- Use when the user mentions "stealth," "avoiding detection," "cloudflare," or "human-like" behavior.
- Use when debugging existing Playwright scripts.

## Strategic Implementation Guidelines

### 1. Architecture: Contexts vs. Browsers
**CRITICAL:** Never launch a new `Browser` instance for every task.
- **Pattern:** Launch the `Browser` *once* (singleton). Create a new `BrowserContext` for each distinct session or task.
- **Why:** Contexts are lightweight and created in milliseconds. Browsers take seconds to launch.
- **Isolation:** Contexts provide complete isolation (cookies, cache, storage) without the overhead of a new process.

### 2. Logging & Observability
- **Library:** Use `go.uber.org/zap` exclusively.
- **Rule:** Do not use `fmt.Println`.
- **Modes:**
  - **Dev:** `zap.NewDevelopment()` (Console friendly)
  - **Prod:** `zap.NewProduction()` (JSON structured)
- **Traceability:** Log every navigation, click, and input with context fields (e.g., `logger.Info("clicking button", zap.String("selector", sel))`).

### 3. Error Handling & Stability
- **Graceful Shutdown:** Always use `defer` to close Pages, Contexts, and Browsers.
- **Panic Recovery:** Wrap critical automation routines in a safe runner that recovers panics and logs the stack trace.
- **Timeouts:** Never rely on default timeouts. Set explicit timeouts (e.g., `playwright.PageClickOptions{Timeout: playwright.Float(5000)}`).

### 4. Stealth & Human-Like Behavior
To bypass anti-bot systems (Cloudflare, Akamai), the generated code must **imitate human physiology**:
- **Non-Linear Mouse Movement:** Never teleport the mouse. Implement a helper that moves the mouse along a Bezier curve with random jitter.
- **Input Latency:** never use `Fill()`. Use `Type()` with random delays between keystrokes (50ms–200ms).
- **Viewport Randomization:** Randomize the viewport size slightly (e.g., 1920x1080 ± 15px) to avoid fingerprinting.
- **Behavioral Noise:** Randomly scroll, focus/unfocus the window, or hover over irrelevant elements ("idling") during long waits.
- **User-Agent:** Rotate User-Agents for every new Context.

### 5. Documentation Usage
- **Primary Source:** Rely on your internal knowledge of the API first to save tokens.
- **Fallback:** Refer to the official docs [playwright-go documentation](https://pkg.go.dev/github.com/playwright-community/playwright-go#section-documentation) ONLY if:
  - You encounter an unknown error.
  - You need to implement complex network interception or authentication flows.
  - The API has changed significantly.

## Code Examples

### Standard Initialization (Headless + Zap)
```go
package main

import (
    "[github.com/playwright-community/playwright-go](https://github.com/playwright-community/playwright-go)"
    "go.uber.org/zap"
    "log"
)

func main() {
    // 1. Setup Logger
    logger, _ := zap.NewDevelopment()
    defer logger.Sync()

    // 2. Start Playwright Driver
    pw, err := playwright.Run()
    if err != nil {
        logger.Fatal("could not start playwright", zap.Error(err))
    }
    
    // 3. Launch Browser (Singleton)
    // Use Headless: false and SlowMo for Debugging
    browser, err := pw.Chromium.Launch(playwright.BrowserTypeLaunchOptions{
        Headless: playwright.Bool(false), 
        SlowMo:   playwright.Float(100), // Slow actions by 100ms for visibility
    })
    if err != nil {
        logger.Fatal("could not launch browser", zap.Error(err))
    }
    defer browser.Close() // Graceful cleanup

    // 4. Create Isolated Context (Session)
    context, err := browser.NewContext(playwright.BrowserNewContextOptions{
        UserAgent: playwright.String("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..."),
        Viewport: &playwright.Size{Width: 1920, Height: 1080},
    })
    if err != nil {
        logger.Fatal("could not create context", zap.Error(err))
    }
    defer context.Close()

    // 5. Open Page
    page, _ := context.NewPage()
    
    // ... Implementation ...
}
```

### Human-Like Typing & Interaction
```go
import (
    "math/rand"
    "time"
)

// HumanType simulates a user typing with variable speed
func HumanType(locator playwright.Locator, text string) {
    // Focus the element first (like a human)
    locator.Click()
    
    for _, char := range text {
        // Random delay: 50ms to 150ms
        delay := time.Duration(rand.Intn(100) + 50) * time.Millisecond
        time.Sleep(delay)
        locator.Press(string(char))
    }
}

// HumanClick adds offset and hesitation
func HumanClick(page playwright.Page, selector string) {
    box, _ := page.Locator(selector).BoundingBox()
    if box == nil {
        return
    }
    
    // Calculate center with random offset (jitter)
    x := box.X + box.Width/2 + (rand.Float64()*10 - 5)
    y := box.Y + box.Height/2 + (rand.Float64()*10 - 5)
    
    // Move mouse smoothly (requires custom Bezier implementation or steps)
    page.Mouse().Move(x, y, playwright.MouseMoveOptions{Steps: playwright.Int(10)})
    time.Sleep(100 * time.Millisecond) // Hesitate
    page.Mouse().Click(x, y)
}
```

#Session Management (Save/Load Cookies)

```go
func SaveSession(context playwright.BrowserContext, filepath string) {
    cookies, _ := context.Cookies()
    // Serialize cookies to JSON and write to 'filepath'
}

func LoadSession(context playwright.BrowserContext, filepath string) {
    // Read JSON from 'filepath' and deserialize
    // var cookies []playwright.Cookie
    context.AddCookies(cookies)
}
```

### Summary Checklist for Agent
 - Is Debug Mode on? -> Headless=false, SlowMo=100+.
 - Is it a new user identity? -> Create NewContext, apply new Proxy, rotate User-Agent.
 - Is the action critical? -> Wrap in SafeAction with zap logging.
 - Is the target guarded (Cloudflare/Akamai)? -> Enable HumanType, BezierMouse, and Stealth Scripts.
