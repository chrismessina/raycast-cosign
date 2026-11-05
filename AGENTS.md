# Cosign extension: notes for contributors

Paths are repo-relative. This file lives in the standalone mirror only; it is gitignored so it stays out of the Store PR.

## How it talks to Cosign

- `src/lib/cosign-mcp.ts` is the only network layer. It speaks MCP over HTTP to `https://api.cosign.co/mcp`
  and signs in with OAuth PKCE and dynamic client registration. The authorize and token requests send
  `resource=https://api.cosign.co/mcp`.
- `callTool` opens one initialized session per command process and sends `client_type: "app"` with every call, so
  Cosign needs no `llm_model` or `reason_for_call`. It returns `structuredContent.data` (every tool answers
  `{ schemaVersion: "2.0.0", data }` or `{ schemaVersion, error }`); a tool error throws its message plus `retryAfter`.
- The tool catalog is Cosign's v2 (2026-11-03): no old names survive. Grouped tools take `request: { action, args }`
  (`read_company`, `read_conversations`, `search_companies`, `research_investors`, `find_lists`, `read_list`,
  `edit_list_entries`). Every tool advertises an `outputSchema` in `tools/list`; read it before guessing a shape.
- The scope is `member:read member:write`. Write access is used for favorites and for accepting or archiving
  connection requests. Changing `SCOPE` forces a re-registration and one re-consent.
- Every request after `initialize` sends `mcp-protocol-version` set to the version the server negotiated. A refresh
  that runs in the background (`LaunchType.Background`, the menu bar) never opens a browser sign-in; it throws
  `SIGN_IN_REQUIRED` instead.
- `src/lib/api.ts` wraps the tools we call; `src/lib/types.ts` holds the parts of each output schema we use.
- People: `handle` is now the X handle (`lookup_person` returns it with its "@"), and `profileUrl` from
  `lookup_person` is the x.com page. The Cosign page is built from the nominee id (`nomineeUrl()` in
  `src/lib/inbox.ts`). Someone not on Cosign has no `nomineeId`, so search rows fall back to the handle as the id.
- Companies: `read_company` refuses investment firms with `{ error }`; `fetchOrganization` then reads
  `research_investors` `firm_profile`. Open roles come from `search_jobs` with `criteria.employerName`, because
  there is no company-id filter.

## Rate limits

Cosign has three account-wide budgets: semantic discovery (30/min, 200/hour, 600/day; topic search and job search),
other AI operations (3/min, 20/hour, 60/day) and list-moderation writes (30/min, 200/hour, 600/day). Name lookups
and ordinary favorite writes spend neither AI pool. Searches stay explicit (Enter) and pagination stays explicit
("Load More"). The investments view pages by scroll (`usePromise` pagination); funding follows its cursor to the end (50-page cap; a leftover cursor shows "Some older rounds aren’t shown").

## Favorites

Favorites live in two private Cosign lists, "Raycast Favorites: People" and "Raycast Favorites: Companies"
(`src/lib/favorites.ts`). The list ids are cached in LocalStorage. People are removed by `nomineeId`; a company entry
is removed by the list entry's wrapper `nominee.nomineeId`, not by `organizationId`. Cosign refuses investing firms
in the companies list.

## UI conventions

- Failure toasts come from `@chrismessina/raycast-kit` (`showError`, `failToast`), and cached loads use
  `onError: onErrorToast({ title })`. A message a person can't act on (Cosign's "Output validation error") is
  thrown as one short sentence with the raw text as `cause` (`serverFault()` in `src/lib/cosign-mcp.ts`); the
  kit copies the cause under Copy Error and never shows it in the toast. Error screens copy the same text through
  `clipboardText()` in `src/lib/errors.ts`.
- Each search command takes an optional `query` argument, which `SearchList`'s `initialQuery` runs once on open.
- `src/components/SearchList.tsx` drives all three search commands: Enter to search, the bar clears and results stay
  so typing filters them, and the selected row's profile loads after a 250 ms debounce.
- `src/components/view-actions.tsx` holds the "View" action section: Show/Hide Sidebar and Show/Hide Description.
  The description setting is one shared `useCachedState` key, so it applies in every list.
- `src/components/investments.tsx` groups an investors list into one section per round, newest first, then by
  amount. Rows with no round go in a "Stage Unknown" section at the bottom.

## Inbox

- `src/inbox.tsx` lists conversations (`read_conversations` `inbox`) and pending connection requests
  (`browse_network`). Cosign exposes no tool to send or start a message, so replying opens the thread in the browser.
  Reading a thread (`read_conversations` `messages`) does not mark it read.
- A request is identified by its `reviewKey`, not by `actorUserId`: a company sender and its operator can share a
  user id. A sender can recur across pages; `fetchNetworkRequests` keeps one row per `reviewKey` and merges the
  signal kinds. Accept takes `actorTarget.nomineeId` (a person) or `actorTarget.organizationId` (a company) and
  answers with the new chat's `{ threadId, url }`. Archive sends `senderUserId` and `reviewKey`.
- `src/inbox-menu-bar.tsx` refreshes every 10 minutes with one conversation read plus up to five request-page reads, and never reads thread bodies. Its count is
  `unreadCount()` in `src/lib/inbox.ts`; message previews are cut with `truncate()` in `src/lib/format.ts`.
- Server text is untrusted. Links to Cosign go through `absolute()` (https on cosign.co only, else the home page); the one exception is an X
  profile link, which `xProfile()` builds from the handle with the handle URL-encoded. Names and
  other short text put into Markdown go through `escapeMd()`, which backslash-escapes ASCII punctuation so nothing
  forms a link, list, heading or HTML. Brackets become `&#91;`/`&#93;` and parentheses stay as they are, because
  Raycast renders `\(…\)` and `\[…\]` as LaTeX. Message bodies and previews are written in Markdown on Cosign, so
  `quote()` keeps their formatting: it block-quotes every line (CR, LF and CRLF normalized) so a body cannot forge a
  sender line, and turns images into text so opening a thread never loads a remote picture.
