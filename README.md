# Vaadin Dev Loop – Spring Boot based Example

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

Output from a run with Claude Code (Vaadin 25.3.5, Spring Boot 4.1.2):

| Step | Path | Exit code | Duration | Reason for restart |
|---|---|---|---|---|
| 1. New view | Hotswap | 0 | 0.69 s | – (new route: page reload needed to show it) |
| 2. CSS color | HMR | 0 | 0.07 s | – |
| 3. Text + placeholder | Hotswap | 0 | 0.63 s | – |
| 4. Button field + notification | Hotswap | 0 | 0.66 s | – |
| 5. `GreetingService` + injection | Restart (automatic) | 0 | 5.19 s | New Spring bean — component scanning only runs at startup |
| 6a. Compile error | Error | 1 | 0.08 s | – (`';' expected`; app keeps last good bytes) |
| 6b. Fix | Hotswap | 0 | 0.64 s | – |

Steps 2–4 went live in the open page without a reload. Only adding a new Spring bean triggered a restart, which the daemon performed by itself.
