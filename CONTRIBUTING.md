# Contributing to Cosign

Thanks for helping. This guide covers running Cosign locally and sending a change.

## Where changes go

Cosign is published in the [Raycast Store](https://www.raycast.com/chrismessina/cosign).
Open pull requests against [`raycast/extensions`](https://github.com/raycast/extensions/tree/main/extensions/cosign),
not this repository: Raycast reviews every change there, and merged changes come back here in a
sync pull request. Open [issues](https://github.com/chrismessina/raycast-cosign/issues) here or in
`raycast/extensions`, whichever you prefer.

## Run it locally

You need [Raycast](https://www.raycast.com/) and Node.js 22.22.2 or later.

1. In Raycast, find Cosign in the Store and choose **Fork Extension** from the action panel.
   Raycast checks out only this extension's folder from `raycast/extensions`.
2. In that folder, run `npm ci`, then `npm run dev`. Raycast opens the development version, and
   saving a file reloads it.

You also need a [Cosign](https://cosign.co) account. The first command that loads your Cosign data
opens a sign-in window (Sign Out doesn't, and a background refresh never does).

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | Run the extension in Raycast and reload on save |
| `npm run build` | Build the distribution bundle |
| `npm run lint` | Run Raycast's ESLint and Prettier checks |
| `npm run fix-lint` | Fix what the linter can fix |
| `npm test` | Run the unit tests in `test/` with `node --test` |
| `npm run publish` | Open or update the pull request in `raycast/extensions` (`ray publish`) |
| `prepublishOnly` | Stops `npm publish`: the extension goes to the Raycast Store, never to the npm registry |

## How it works

The extension talks to Cosign's MCP server (`https://api.cosign.co/mcp`), not a REST API.

- `src/lib/cosign-mcp.ts` handles sign-in (OAuth with PKCE through Raycast's `OAuth.PKCEClient`,
  after registering a client at `/register`) and `callTool()`, which sends a JSON-RPC `tools/call`
  request. On a 401 or 404 (an expired token or a dropped session) it reopens the session and
  retries once.
- `src/lib/api.ts` wraps the MCP tools the commands use (`fetchPerson`, `fetchOrganization`,
  `fetchFunding`, `fetchJobs`, `fetchInvestments`, `fetchConversations` and so on), with the result
  types in `src/lib/types.ts`.
- `src/lib/favorites.ts` keeps Favorites in two private Cosign lists, so they show up in the Cosign
  web app too. `src/lib/recents.ts` keeps recent searches in LocalStorage.
- `src/lib/format.ts` and `src/lib/inbox.ts` hold the pure formatting and inbox helpers the tests
  in `test/` cover.
- Each command is a file at the top of `src/`; shared views and actions are in `src/components/`,
  with the search list in `src/components/SearchList.tsx`.

## Before you open a pull request

- `npm run build`, `npx tsc --noEmit` and `npm run lint` all pass, and so does `npm test`.
- You tried the change in Raycast, including its error and empty states.
- `CHANGELOG.md` has a new entry at the top, dated `{PR_MERGE_DATE}`. Raycast fills in the date
  when the pull request merges.
- New or changed screenshots go in `metadata/`, taken with Raycast's Window Capture.
