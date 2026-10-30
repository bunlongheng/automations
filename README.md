<div align="center">
  <img src="docs/icon.png" alt="Automations" width="96" height="96" />
  <h1>Automations</h1>
  <p><em>An AI automation platform: describe a trigger-to-action in plain text and Claude wires it</em></p>
  <p><a href="https://github.com/bunlongheng/automations">Repo</a> &middot; <a href="https://bunlongheng.com/projects?name=automations">Portfolio</a></p>
  <img src="docs/social-preview.png" alt="Automations - preview" width="820" />
</div>

---

<div align="center">

# Automations

<p align="center"><img src="docs/hero.png" alt="automations" width="100%"></p>


Visual node-graph automation flow builder - drag trigger and action nodes onto a React Flow canvas, wire them with gradient edges, and a VPS-side pipeline polls, executes, and delivers the output.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![React](https://img.shields.io/badge/React-19-149eca?logo=react)
![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178c6?logo=typescript)
![React Flow](https://img.shields.io/badge/React%20Flow-%40xyflow%2Freact-ff0071)
![Tailwind CSS](https://img.shields.io/badge/Tailwind%20CSS-3-38bdf8?logo=tailwindcss)
![Tests](https://img.shields.io/badge/tests-vitest%20%2B%20playwright-6e9f18)

</div>

## Contents

- [Features](#features)
- [How it works](#how-it-works)
- [Tech stack](#tech-stack)
- [Getting started](#getting-started)
- [Environment variables](#environment-variables)
- [Project layout](#project-layout)
- [License](#license)

## Features

- **Drag-and-drop canvas** (`components/FlowCanvas.tsx`, `@xyflow/react`) - drag an integration from the sidebar onto the canvas, pick "Add as Trigger" or "Add as Action" in a two-column modal, and the node drops in with an auto-centered layout (triggers left, actions right).
- **12 integrations, 5 categories** (`data/integrations.ts`) - YouTube, Gmail, GitHub, Slack, Calendar, Webhook, Stickies, Diagram, Mind Map, Claude, Philips Hue, and Open Claw, each with its own typed trigger/action list.
- **Gradient bezier edges** (`components/edges/GradientEdge.tsx`) - connections animate and are colored by a gradient between the source and target integration's brand color.
- **Auto-connect + auto-name** - dropping exactly one trigger and one action wires them automatically; the flow name is derived live from the chain (e.g. `YouTube Liked > Diagram`), and `Cmd+S` saves it.
- **Per-node config panel** (`components/panels/NodeConfigPanel.tsx`) - configure trigger conditions and action parameters without leaving the canvas.
- **Code view** (`components/panels/CodePanel.tsx`) - generates a readable DSL rendering of the current graph (`automation { trigger { } | action { } }`, chain-aware, with syntax highlighting) that can be copied with one click.
- **Mobile wizard** (`components/MobileWizard.tsx`) - a step-based, canvas-free builder as the small-screen alternative to drag-and-drop.
- **Connections panel** (`components/ConnectionsPanel.tsx`) - Google (YouTube/Gmail/Calendar) and GitHub OAuth, plus a live "check connections" ping that verifies each integration is actually reachable, not just marked connected in the DB.
- **Run tracking** - the automations list shows `total_runs`, `success_runs`, and `last_run` per flow, and inactive automations render desaturated.
- **Live status via SSE** (`app/api/events/route.ts`) - proxies a Server-Sent Events stream from the VPS pipeline to the browser.
- **YouTube pipeline** (`server/pipeline.js`) - watches liked videos, pulls the transcript through a 4-method fallback chain (yt-dlp with cookies/residential proxy -> multi-client Innertube -> public API -> paid transcriptapi.com), summarizes it with Claude, and delivers the result to Stickies while generating a mind map and a Mermaid sequence diagram from the same summary in parallel.
- **MCP server** (`mcp/index.js`) - exposes `list_automations`, `get_automation`, `toggle_automation`, `delete_automation`, `process_youtube_video`, `unlike_youtube_video`, `list_youtube_likes`, `get_youtube_status`, `check_connections`, `list_connections`, and `connect_integration` as tools so an agent (e.g. Claude Code) can drive automations directly.

## How it works

The app is split across two processes: a Next.js UI/API layer (canvas, OAuth, config) and an always-on Express server on a VPS that owns the poll loop, Postgres writes, and the YouTube pipeline. The Next.js side never talks to Postgres directly - every read/write proxies through the VPS over a bearer token.

```mermaid
flowchart LR
    Canvas["FlowCanvas\n(React Flow, client)"] -->|drag / connect / save| API["Next.js API routes"]
    API -->|OAuth| OAuth["Google + GitHub OAuth"]
    API -->|"Bearer VPS_AUTH_TOKEN"| VPS["Express server\n(server/index.js, VPS)"]
    VPS --> Watcher["watcher.js\npoll liked videos"]
    Watcher --> Pipeline["pipeline.js"]
    Pipeline -->|transcript| Fallback["yt-dlp -> Innertube ->\npublic API -> transcriptapi"]
    Pipeline -->|summarize + diagram| Claude["Anthropic Claude API"]
    Pipeline --> DB[(Postgres)]
    DB --> Sticky["Stickies note"]
    DB --> MindMap["Mind map"]
    DB --> Diagram["Sequence diagram"]
    VPS -->|SSE| API
    MCP["mcp/index.js\nMCP server"] -->|tool calls| API
    MCP -->|tool calls| VPS
```

## Tech stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router) |
| UI | React 19, Tailwind CSS, `motion` |
| Language | TypeScript |
| Graph editor | `@xyflow/react` (React Flow) |
| OAuth | Google + GitHub (per-integration connections: YouTube/Gmail/Calendar/GitHub) |
| Backend pipeline | Node.js + Express, `pg` (Postgres), on a separate VPS process |
| AI | Anthropic Claude API - transcript summarization and Mermaid diagram generation |
| Automation surface | Model Context Protocol server (`@modelcontextprotocol/sdk`) |
| Testing | Vitest (unit), Playwright (E2E), MSW, Testing Library |
| CI | GitHub Actions - typecheck, lint, test on every push/PR |

## Getting started

```bash
git clone https://github.com/bunlongheng/automations.git
cd automations
npm install
cp .env.example .env.local   # fill in the values, see below
npm run dev                  # starts on port 3008
```

```bash
npm run build       # production build
npm run start        # start production server
npm run lint          # eslint
npm run typecheck    # tsc --noEmit
npm run test           # vitest run
npm run test:watch   # vitest, watch mode
npm run test:coverage # vitest run --coverage
npm run test:e2e     # playwright test
npm run test:all      # vitest + playwright
```

The poll loop, SSE stream, and Postgres writes run separately on the VPS (`server/index.js` - `npm start` inside `server/`), not on Vercel.

## Environment variables

Copy `.env.example` to `.env.local`. Required vars:

- `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` - OAuth for YouTube, Gmail, Calendar
- `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` - GitHub OAuth
- `ANTHROPIC_API_KEY` - Claude API for transcript summarization and diagram generation
- `VPS_URL` / `VPS_AUTH_TOKEN` - the automations pipeline backend
- `STICKIES_URL` / `STICKIES_TOKEN` - Stickies API for posted summaries
- `NEXT_PUBLIC_APP_URL` - public app URL (OAuth callbacks, MCP)
- `TRANSCRIPTAPI_KEY`, `YOUTUBE_COOKIES_FILE`, `YTDLP_PROXY` - optional transcript fallbacks
- `ALLOWED_ORIGIN`, `CHECK_INTERVAL_SEC` - set on the VPS side, not Vercel

## Project layout

```
automations/
  app/              # Next.js App Router - pages + API routes (auth, automations, connections, youtube, gmail, events)
  components/       # FlowCanvas, node/edge types, panels, sidebar, connections + mobile wizard
  data/             # integrations.ts - trigger/action catalog for all 12 integrations
  lib/              # OAuth state, VPS client helpers
  server/           # standalone Express app - poll loop, YouTube pipeline, Postgres writes (deployed to VPS)
  mcp/              # MCP server exposing automations as agent tools
  types/            # shared Automation type
  tests/            # Vitest + Playwright suites
```

## License

MIT - see [LICENSE](LICENSE).

---

Built by [Bunlong Heng](https://www.bunlongheng.com) | [GitHub](https://github.com/bunlongheng/automations)
</content>

---

<p align="center">
  <sub>Built by <a href="https://bunlongheng.com">Bunlong Heng</a> &middot; <a href="https://bunlongheng.com/projects/automations">See it in my portfolio &rarr;</a></sub>
</p>

---

<div align="center">

<a href="https://bunlongheng.com"><img src="https://img.shields.io/badge/-bunlongheng.com-3A3A3C?style=for-the-badge&amp;labelColor=2A2A2C&amp;logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABwAAAAcCAMAAABF0y%2BmAAADAFBMVEXx8vLq6v%2F19fX09PT8%2Ff309PT5%2BfnAwMBMaXHx8vL19fX5%2Bfny8vL09PT39%2Ff6%2Bfn6%2Bvr09PTx8fH4%2BPn%2F8vLy8%2FP09PTz8%2FPy8vL%2F%2F%2F%2Fz9PTz9PP19fby8vP19PXz8%2FT19%2Fb08%2FTx8vLy8vL09vby8%2FL29vf19fTv8PDz9vb7%2Bvn%2F%2Ff%2F9%2Ff3w8vH29fb%2F%2FP339%2Ff5%2Bfn59%2Ff19PT09PR7rao3VF3%2F%2Fv8AKDR6sqsBV1vv8fL4%2BPiux8b8%2Bvq90NDy8%2FR%2BlpppnZyZqrACLTxclZJSkI9en5ssZ2luqaUAeWgqbW4BX1gCfG0%2Fa3QaVG0MN0UKNUKUpKsGpYMAKjoRQ1UDf3AQRloJqIgEm32d0cUBg3ElbnQIln8eVWgrmIdP0KzF5d5s2acdfHwppJCP5MMXl4Tz8%2FLv8%2FP39%2FiewL5%2BqahYf4IrTFZkk5KlxsREiIYAHiqDnKFmnJowc3IdVllvpKK7zc3D2tnk5%2BmuycdwpKKuub2yv8Ly%2B%2FkAMT7z%2Bvjq7e09gH93q6kGdWpGgoJZmJSlw8M7c3WwyceTw76w1M8mYmpFfX8lW2AFOEpGiYhxqqRrqaRb2rdIsZ8AVE91jpU7d3t3saqBoKhako4AWVEcUlg3aWwDupAHb2YwmIkPP1ERSVA7WmZ8tq8HT14oZW%2BYuLsMl3wQT10ROksENUc8Z3MJSGVLh4dqpJ%2BXr7U51awUgHM4v6ad1MkPuJZAiowVd3IEhnZH0aoYrI05p5sDtYwkzZ8EPViYxsRel5USbHAIgmwIM0gQamzU4uF2malQi4oWQFQ1zaYrp4UXlYwfiHoyxqFE1q4qeX4yn5JIxKkdrX1b1rAbWGvd5Oh91sRijpomsJEVc3Ukc34dqIcwuJVKwZIXkoQZcXlNv5g%2BqZUZvpNl2rPG9N0WkIoNqouW4sVPwp%2BM0b6h0ssfq4d%2F1r8hpo%2Fh9e4AamUjnI9avaBixqgklYWSwL4xtItUxplYl5qZ58mq78wMgnYXfnlOmJaR1L%2Bj78ny9fS%2FrXQPAAAAFXRSTlP7Brvx%2FsJhAgD87r4U72C4uGH8vhRDodYHAAAACXBIWXMAAAsTAAALEwEAmpwYAAAC6UlEQVQokS2Sd3BUVRSHb0KS3QRCiZw5t%2By9j33zXjZkyUs2uekhEAi9dwRCL4JUEQRFAQUs9A4WOopKU0GpFoqCBZUivaqoFKkWLIS5GX7%2FfnPO78yZj%2Fj8UTViK6HggvNgSGuluEZRKbZqlN9H%2FFWiWUzIFQK5fhjOIRTDoqv4ia86S4q3KSByHdSqIhxofBKr5iNxLMm2bQqWVkorJ%2BAkO46nkQJhceSRmCTDvKJIJBLJzMxMLU6POJ4eBPF1KpNEbdt2Cms4LCcnZ%2FX6D3d%2BsKJJ7VSlNLoqgbjUtmkKKxlVmpf3bsFXW3d1b77gueJAQIPQBCilkMKemNi23Ts9T35%2FaMe2rsvmTypylGUFCSIYWJLXrm2nUwOPfNO5w%2B73Okxnjqe0JigQEL3JizoeONr%2F2u99vju4fd1rdZnyspQmnGvAUBprWNqz348%2FhXN%2F6P1%2By4Uvs%2FJkpRQRwpJBRVn7vf16Dfitz5XevQqWt3yalXtKaRIUQmglWfsu5y7%2BMrDs0TMftWo1ZVy64xioOVpcIdt8%2Fvqvf9%2F859KJsrLmrzA%2BOKC0IkFEtJza7JOfr%2FYd0P9G98tdOxe0Hp9qJrOIBgDuOeEtp%2F%2B4d%2FvWX%2F%2F9u%2B%2FtlaUTmOWZg7SQEEyuFe52%2BML9O3cZO%2F5xl06zn2VcadOJUqJTzt7a82XfP7sx9vmGNm1m1MstV4IHCVAAoTLqL1372ddn%2Fz%2B2v%2FWq12c1fSpsaeQWAZAANntxzuI3e3zbo8XGpm%2FMnNqiXrh%2BljaPd6UUabkjps19tdGnX2zKb9b4pTFDCzOMLmYtpW5h%2BpCRTeYtyV%2BT36zx808OL4xYGoFrkqDTbJpSXLdRgwbPZL%2BQnT129OOPFVEEsAMJpHKdeKMJgJSAiALdjJANQKnRJI4R6boAKEwLCiGoBBegQjBftQo1KQWOlplFKSV9qGaF1NpAE7QQESTlMSy6pp%2F4%2FFFVYxMNRBTaiM0tkImxNaL8vgfDR8gvoYRaxgAAAABJRU5ErkJggg%3D%3D" alt="bunlongheng.com"></a>
<a href="https://www.linkedin.com/in/bunlongheng/"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"></a>
<a href="https://www.instagram.com/ibunlong/"><img src="https://img.shields.io/badge/Instagram-C13584?style=for-the-badge&logo=instagram&logoColor=white" alt="Instagram"></a>
<a href="mailto:bheng.code@gmail.com"><img src="https://img.shields.io/badge/Email-2E7D32?style=for-the-badge&logo=gmail&logoColor=white" alt="Email"></a>

<br>

Built by **[Bunlong](https://bunlongheng.com)** &nbsp;&middot;&nbsp; [more apps](https://bunlongheng.com/projects)

</div>
