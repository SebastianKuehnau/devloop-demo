# Vaadin Dev Loop – Spring Initializr based Example

This project has been created using [Spring Initializr](https://start.spring.io/).

The Vaadin **Dev Loop** shortens the cycle needed to make code changes live, visible in the running app, or verified by tests. A daemon owns the application process and hot-swaps, pushes CSS, or restarts only when necessary — no full rebuild or app restart per edit.

This matters most for **agentic development**: when an agent works through a large task, constant rebuilds and restarts add up and slow the whole task down. The Dev Loop can of course also be used when working with an AI assistant directly in the CLI.

Official setup guide: <https://vaadin.com/docs/latest/building-apps/agentic-development/dev-loop-tutorial#what-you-need>

## Prerequisites

- Java 21+ (JetBrains Runtime recommended)
- Vaadin **25.3** or newer
- Claude Code or Codex installed

## Setup

1. Add the `vaadin-dev` dependency to `pom.xml`:

   ```xml
   <dependency>
       <groupId>com.vaadin</groupId>
       <artifactId>vaadin-dev</artifactId>
       <optional>true</optional>
   </dependency>
   ```

2. Install the Dev Loop CLI and skills:

   ```bash
   ./mvnw vaadin:install-dev-cli
   ```

   This adds:

   | Path | Content |
   |---|---|
   | `.vaadin/` | The CLI: `vaadin-dev` (macOS/Linux), `vaadin-dev.cmd` / `vaadin-dev.ps1` (Windows), plus its own `.gitignore` |
   | `.agents/skills/vaadin-devloop/` | Agent-agnostic skill: `SKILL.md` (workflow instructions) and `reference.md` (output vocabulary, troubleshooting) |
   | `.claude/skills/vaadin-devloop/` | `SKILL.md` for Claude Code (no own `reference.md`) |
   | `~/.vaadin/devloop/hotswap-agent-<version>.jar` | Hotswap agent used by the daemon |

3. Add the Vaadin skills and MCP server for your agent. Here for Claude Code:

   ```
   /plugin marketplace add vaadin/agent-marketplace
   /plugin install vaadin-skills@vaadin-marketplace
   ```

   In this repo this is preconfigured in `.claude/settings.json` (Vaadin skills + Vaadin MCP via `vaadin-skills`, Playwright MCP via `playwright@claude-plugins-official`); Claude Code offers to install them when you trust the folder.

4. Start the app through the Dev Loop (OS-specific commands: see the docs):

   ```bash
   .vaadin/vaadin-dev start
   ```

   The app is served at <http://localhost:8080>.

> **Do not** start the app via `./mvnw spring-boot:run` or an IDE run configuration — the daemon owns the process, and a second instance fails on the port conflict.

Useful commands: `status`, `apply`, `restart`, `stop`, `shutdown` (e.g. `.vaadin/vaadin-dev status`).

## Try it with a prompt

With the app running, give your agent this prompt:

```text
Please test the Vaadin Dev Loop in this project using a new view.
Make the following changes one after another, each as its own step, and
make each one live before moving on. Never restart the app yourself via
Maven or the IDE.

1. New view: Create HelloDevLoopView with route "" and RouteAlias "hello-devloop",
   containing an H1 "Hello Dev Loop" and a TextField for a name. Open the view in
   the browser.
2. CSS: Give the H1 a striking color in styles.css.
3. Logic: Change the H1 text to "Hello Dev Loop!" and the TextField placeholder
   to "Your name".
4. Structural: Add a new field to the view holding a "Greet" button that shows
   a notification "Hello <name>".
5. New Spring service: Create a GreetingService (@Service) with a method
   greet(String name), inject it into the view and use it in the button.
   For an empty or blank name, greet returns "Hello guest", otherwise "Hello <name>".
6. Error case: Intentionally introduce a compile error in HelloDevLoopView,
   make it live, then fix it again.

Verify visible changes in the browser. At the end, give me a table with:
Step | Path (HMR / Hotswap / Restart / Error) | Exit code | Duration | Reason,
if a restart was needed.
```

## Example result

Output from a run with Claude Code (Claude Opus 5.5, Vaadin 25.3.5, Spring Boot 4.1.2), each step sent as its own headless turn (`claude -p --resume`) in one session, so tokens can be attributed per step:

| Step | Path | Exit code | Duration | Without Dev Loop (measured) | Reason for restart |
|---|---|---|---|---|---|
| 1. New view | Hotswap | 0 | 0.87 s | ~6 s (compile + DevTools restart) | – (new route: page reload needed to show it) |
| 2. CSS color | HMR | 0 | 0.01 s | ~5 s (compile + manual reload) | – |
| 3. Text + placeholder | Hotswap | 0 | 0.61 s | ~5.7 s (compile + DevTools restart) | – |
| 4. Button field + notification | Hotswap | 0 | 0.62 s | ~5.5 s (compile + DevTools restart) | – |
| 5. `GreetingService` + injection | Restart (automatic) | 0 | 3.25 s | ~5.8 s (compile + DevTools restart) | New Spring bean — component scanning only runs at startup |
| 6a. Compile error | Error | 1 | – (not printed on failure) | 1.3 s compile failure + unwanted DevTools restart (see below) | – (`';' expected`; app keeps last good bytes) |
| 6b. Fix | Hotswap | 0 | 0.61 s | ~5.8 s (compile + DevTools restart) | – |

"Without Dev Loop" was measured in a second run of the same steps with the Dev Loop removed: the app ran via `./mvnw spring-boot:run` with Spring Boot DevTools, and each change was made live with `./mvnw -o compile` (DevTools then restarts the app; CSS needs a browser reload). Times are from compile start until the change was live, read from log timestamps (≈1 s precision). Every restart drops the UI state. In real applications with more beans, JPA etc., startup takes considerably longer, so the gap grows.

### Token usage per step (with Dev Loop)

| Step | Agent time | Turns | Output tokens | Input tokens | of which from cache | Cost (API list price) |
|---|---|---|---|---|---|---|
| 1. New view | 34.5 s | 16 | 2,443 | 611,184 | 570,620 | $0.49 |
| 2. CSS color | 19.5 s | 11 | 1,461 | 334,568 | 331,338 | $0.12 |
| 3. Text + placeholder | 16.5 s | 10 | 1,483 | 235,787 | 232,858 | $0.10 |
| 4. Button field + notification | 33.8 s | 15 | 2,238 | 380,029 | 375,094 | $0.16 |
| 5. `GreetingService` + injection | 22.7 s | 13 | 1,937 | 269,637 | 265,317 | $0.13 |
| 6a. Compile error | 14.6 s | 7 | 1,049 | 212,411 | 210,232 | $0.08 |
| 6b. Fix | 15.7 s | 9 | 1,412 | 218,995 | 216,521 | $0.09 |
| **Total** | **157.3 s** | **81** | **12,023** | **2,262,611** | **2,201,980** | **$1.17** |

Input tokens count the full context again on every model call (turn); ~97 % of them are cache reads, which are billed at a fraction of the normal input price. Step 1 includes the initial context (system prompt, loading the Dev Loop skill, reading the project). Agent time is the agent's total wall time for the step (reading, editing, `apply`, browser check), not just the Dev Loop duration. Exit codes 0 are as reported by the agent; it could not print `$?` under its permission settings.

### Token usage without Dev Loop

Same steps, same model, headless per step, classic cycle (`mvn compile` + DevTools restart):

| Step | Agent time | Turns | Output tokens | Input tokens | of which from cache | Cost (API list price) |
|---|---|---|---|---|---|---|
| 1. New view | 39.7 s | 13 | 3,100 | 426,375 | 387,898 | $0.45 |
| 2. CSS color | 18.5 s | 7 | 1,132 | 267,726 | 265,337 | $0.10 |
| 3. Text + placeholder | 19.9 s | 8 | 1,423 | 337,879 | 333,306 | $0.13 |
| 4. Button field + notification | 27.2 s | 12 | 2,111 | 498,891 | 490,265 | $0.21 |
| 5. `GreetingService` + injection | 34.1 s | 16 | 2,731 | 642,916 | 633,592 | $0.26 |
| 6a. Compile error | 44.2 s | 12 | 3,426 | 814,720 | 804,872 | $0.31 |
| 6b. Fix | 23.6 s | 8 | 1,609 | 536,339 | 530,786 | $0.18 |
| **Total** | **207.2 s** | **76** | **15,532** | **3,524,846** | **3,446,056** | **$1.63** |

| Total | With Dev Loop | Without Dev Loop | Difference |
|---|---|---|---|
| Agent time | 157.3 s | 207.2 s | +32 % |
| Output tokens | 12,023 | 15,532 | +29 % |
| Input tokens | 2,262,611 | 3,524,846 | +56 % |
| Cost (API list price) | $1.17 | $1.63 | +40 % |

Where the difference comes from: every turn re-sends the whole context, and without the Dev Loop the context grows faster (≈46k vs. ≈28k input tokens per turn on average) — the agent reads Maven output, application logs and timestamps to find out whether a restart has finished, instead of getting one verdict line from `apply`. The biggest gap is the error case (6a): the failed `mvn compile` deleted `Application.class`, DevTools restarted the app on the incomplete classes, and it ran for ~45 s without its app shell and styles — the agent spent extra turns investigating. With the Dev Loop, the app simply kept its last good bytes. Steps 1 and 2 were slightly cheaper without the Dev Loop (no skill to load, and the CSS change needed no restart); from step 3 on, the classic cycle costs more per step.

These are single runs; agent runs vary noticeably from run to run, so treat the percentages as indicative, not exact.

The Playwright page did not survive between the headless turns, so the CSS push (step 2) and the hotswap in step 3 were verified after a fresh page load; in step 4 the button appeared in the already-open page without a reload. Only adding a new Spring bean triggered a restart, which the daemon performed by itself.
