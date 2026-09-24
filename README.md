# Questify

Questify turns a real-world goal into an RPG-style learning quest. Give it something you want to accomplish; for example, “Learn LangGraph”. and it searches the web for related material, then returns a quest with a title, difficulty, time estimate, objectives, a boss battle, XP, and recommended resources.

## Status

| Piece | Where it lives | What it does today |
| --- | --- | --- |
| Web app | [`src/app`](src/app) | Next.js starter page at `/`. The quest form and API client are not wired up yet. |
| Quest service | [`questify-flash.tar.gz`](questify-flash.tar.gz) | Runpod serverless handler that accepts a goal and returns quest JSON. |

## Architecture

```text
goal
  │
  ▼
Runpod Flash handler (Python)
  │
  ├─ You.com Search  →  main query + "tutorial guide resources"
  │
  └─ Quest builder   →  difficulty, objectives, boss battle, XP, resources
```

1. The handler receives `{ "goal": "..." }`.
2. It queries the [You.com Search API](https://you.com) twice: once for the goal, and once for tutorials and guides.
3. It merges results, drops duplicate URLs, and builds the quest from those hits.
4. Difficulty, time, XP, objectives, and the boss battle come from keyword heuristics and templates over the goal and search snippets. Resource type (`video`, `course`, `docs`, or `article`) is inferred from the URL and title.
5. If search returns nothing, the handler still returns a generic quest with empty `resources`.

The Next.js app is the intended front end. It does not call Runpod yet. A future integration would send the goal to the Flash endpoint (using `RUNPOD_ENDPOINT_ID`) and render the JSON below.

### Quest response

```json
{
  "title": "Building with LangGraph",
  "difficulty": "Intermediate",
  "estimated_time": "1–2 weeks",
  "objectives": [],
  "boss_battle": "",
  "xp": 700,
  "resources": [
    { "title": "", "url": "", "type": "article" }
  ]
}
```

| Field | Values |
| --- | --- |
| `difficulty` | `Beginner`, `Intermediate`, or `Advanced` |
| `estimated_time` | 4–6 hours, 1–2 weeks, or 3–4 weeks |
| `xp` | 300, 700, or 1500 (500 on the fallback quest) |
| `resources[].type` | `video`, `course`, `docs`, or `article` |
| `objectives` | Up to four actionable steps |
| error | `{ "error": "Missing required field: goal" }` when `goal` is empty |

## Tech stack

**Web app**

- [Next.js](https://nextjs.org) 16 (App Router) and React 19
- TypeScript
- Tailwind CSS 4
- Geist via `next/font`

**Quest service** (inside `questify-flash.tar.gz`)

- Python 3.11
- [Runpod](https://www.runpod.io) serverless SDK
- `requests` for You.com Search

There is no database, auth, or persistence. Each request is stateless.

## Project layout

```text
src/app/                 Next.js App Router (layout, home page, global styles)
public/                  Static assets
questify-flash.tar.gz    Flash image source: handler.py, Dockerfile, requirements.txt
next.config.ts
package.json
```

Flash archive contents:

```text
flash/handler.py
flash/Dockerfile
flash/requirements.txt
flash/README.md
```

## Run the web app locally

Requires Node.js 20 or newer.

```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

| Script | Purpose |
| --- | --- |
| `npm run dev` | Development server |
| `npm run build` | Production build |
| `npm run start` | Serve the production build |
| `npm run lint` | ESLint |

## Run the quest service locally

Extract the archive, then call `generate_quest` without starting Runpod:

```bash
mkdir -p flash && tar -xzf questify-flash.tar.gz
cd flash
pip install -r requirements.txt
YOU_API_KEY=your_key python -c "
from handler import generate_quest
import json
print(json.dumps(generate_quest('Learn LangGraph'), indent=2))
"
```

`YOU_API_KEY` is optional for a smoke test. Without it, search returns no hits and you get the fallback quest.

## Deployment

### Web app

Deploy the Next.js app to [Vercel](https://vercel.com) (or any Node host):

```bash
npm run build
npm run start
```

When the UI calls Runpod, set `RUNPOD_ENDPOINT_ID` in that environment.

### Quest service (Runpod)

From the extracted `flash/` directory:

```bash
docker build -t your-registry/questify-flash:latest .
docker push your-registry/questify-flash:latest
```

Create a Runpod Serverless endpoint from that image. The container runs `python -u handler.py`, which starts `runpod.serverless` with `handler`.

Set on the endpoint:

| Variable | Used for |
| --- | --- |
| `YOU_API_KEY` | You.com Search (`X-API-Key`). Required for real resources. |

Invoke the endpoint with input `{ "goal": "Learn LangGraph" }`.

## Environment variables

| Variable | Where | Required |
| --- | --- | --- |
| `YOU_API_KEY` | Runpod endpoint (or local handler) | For live search results |
| `RUNPOD_ENDPOINT_ID` | Web app, once it calls the endpoint | For the future UI integration |

`.env*` files are gitignored. Do not commit API keys.

## License

Private project (`"private": true` in `package.json`).
