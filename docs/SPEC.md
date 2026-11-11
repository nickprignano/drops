# Drops spec (v1)

Status: the owner answered every question on 2026-11-10; approved when this spec PR merges.
Red-teamed once with Fable on 2026-11-10 (findings and what
changed are at the end).

## What it is

A four-in-a-row game for `drops.prignano.ai`. Two players take turns dropping discs into a
7-column, 6-row board; a disc falls to the lowest empty space in its column. The first to line
up four of their own discs across, up and down, or on a diagonal wins. A full board with no
four is a draw.

Three ways to play:

1. **Play a friend online.** One player starts a room and gets a 6-character code (and a share
   link). The friend types the code on their own device, and they play in real time. No
   sign-in, no names, no chat. Rematches stay in the same room.
2. **Play the computer.** Easy, Medium or Hard, all on the device, offline.
3. **Pass and play.** Two people on one device (open question 2).

It's plain JavaScript and SVG with no build step, served as static files plus one small
Cloudflare Worker for online rooms. It installs as an app (PWA) on iPhone, iPad and Android
from day one. Made for all ages: kids will play it.

## Decisions so far

| # | Question | Answer |
| --- | --- | --- |
| 1 | Kind of game | Four-in-a-row with a friend by room code, plus play against the computer (owner, 2026-11-10) |
| 2 | Online play | Built on the chess app's live-room design: a 6-character code, a Cloudflare Durable Object per room, the server checks every move, no sign-in, nothing about the players kept (owner, via the coordinator, 2026-11-10). Where this spec departs from chess, it says so |
| 3 | Name and look | Original. The classic product's name is a Hasbro trademark, so no part of that name, none of its variants' names, and none of its classic look (see Art) |
| 4 | Home | `nickprignano/drops`, served at `drops.prignano.ai`, deployed by Workers Builds on each merge to `main`. The owner creates the repo and adds the domain. Launch ends with the `content/games.json` PR in `prignanodotai-web` |
| 5 | Name | Drops, at `drops.prignano.ai` (owner, 2026-11-10) |
| 6 | Rest of the questions | Pass-and-play yes, no twist in v1, the creator picks who goes first and it swaps on rematch, room lookup without KV (owner, "defaults for the rest", 2026-11-10) |
| - | License | MIT. Every part is our own work, and nothing we ship forces GPL. The room code is ported from the chess repo (GPL-3.0-or-later): `worker/rooms.js`, `worker/room.js`, the rooms half of `worker/index.js`, `src/live.js` and `test/rooms.test.mjs`. Only the owner and Claude wrote those files and no third-party code is in them, so they can be released here under MIT. The chess rules library is not part of the port. The PWA files follow pinball's recipe (GPL-3.0-only) but are our own, and are copied without pinball's GPL header line |

## Prior art and licenses

Checked 2026-11-10 (GitHub search and our own KB prior-art script).

| Source | License | What we take |
| --- | --- | --- |
| The game itself | Not protectable | The rules (gravity drop, four in a row, 7 × 6) are a genre, like tic-tac-toe. It was solved in 1988, independently by James D. Allen and Victor Allis: the first player wins with perfect play by starting in the middle column |
| [PascalPons/connect4](https://github.com/PascalPons/connect4) and his "Solving Connect 4" tutorial | AGPL-3.0 | Ideas only (negamax with alpha-beta, center-first move order, a transposition table, bitboards). No code. AGPL would also bring network obligations we don't want |
| Other open-source four-in-a-row AIs on GitHub (many MIT, many unlicensed) | mixed | Nothing. The search is standard and short to write ourselves |
| A small room-code four-in-a-row web game found by the prior-art search (no license) | none | Confirms the idea is ordinary. Reference only |
| Our chess app, build 4 (PR #7) | ours | Codes, seat tokens, reconnect, message limits, expiry alarms, rate limits. **New here, not in chess:** rematch, leave, the tally, and asking the room directly instead of KV |
| Pinball PR #7 | ours | The PWA recipe (also in ai-brain's prompt library, "Make a static web game installable") |

## How a game looks and feels

### Art

- An original look in prignano-theme colors that follow light and dark mode. The board is a
  flat tray in the theme's card color with round holes and a thin border, not the classic blue
  upright frame. The discs are **blue** (the theme accent) and **orange**, never the classic
  red and yellow pair.
- Each disc also carries its own mark (blue: a small solid dot; orange: a ring), so color is
  never the only cue. Both colors and marks are checked for contrast against the tray in both
  themes (3:1 or better for the disc against the tray).
- A disc falls with a short ease-in drop and a small settle. The winning four get an outline
  and a line drawn through them; nothing flashes. `prefers-reduced-motion` places discs without
  the fall.
- The icon is an original drawing: four discs on a diagonal in a dark tray.

### Board and controls

- The board scales to fit: on a 360 px wide phone in portrait, each column is at least 44 px
  wide, so a child's finger can hit it. Tap or click anywhere in a column to drop. A ghost disc
  shows above the column you hover or touch.
- Keyboard: left and right arrows move a column cursor, Enter or Space drops, 1 to 7 drop in
  that column. M mutes.
- A full column ignores taps and its ghost disc doesn't appear.

### Screen readers

- The board is a row of 7 buttons, one per column, each read as "Column 3, 2 discs, 4 spaces
  left". A polite live region announces every drop ("Orange, column 3, row 2") and the result.
- A "Read the board" button speaks the board row by row.

### Sound

Synthesized with the Web Audio API, so there is nothing to license: a soft clack on each drop,
a short rising chime for a win, a low tone for a loss, a neutral chord for a draw. Muted until
the first tap, and the mute setting is saved on the device.

## Playing the computer

### Levels

All three run on the device in a Web Worker, so the board never freezes while the computer
thinks, and they work offline. Every level breaks ties between equally good moves with a
seeded random source, so games differ from one to the next and replay exactly from their seed.

| Level | How it plays |
| --- | --- |
| **Easy** | Takes a win in one only 25% of the time and blocks your win in one 25% of the time; otherwise picks a random column, weighted a little toward the middle. Test 2 holds it to "a beginner wins often" |
| **Medium** | Searches 4 moves ahead (negamax with alpha-beta) with a simple score for open threes and center discs. Always takes a win and always blocks a single threat |
| **Hard** | The same search, deepening move by move up to a position cap, with a transposition table and center-first move order. Strong, but not perfect |

- **Hard's cap comes from a measurement, not a guess.** Build 1 measures positions searched per
  second on a mid-range phone (the Playwright check logs it at phone size, and the owner's own
  phone confirms it) and sets the cap to what finishes inside 1.5 seconds. The 1.5-second time
  limit stays only as a safety net. The measured rate and the chosen cap are written into this
  spec in build 1.
- The computer waits at least 400 ms before dropping, so its move is visible.
- The player picks who goes first. The default is the player, since the first player has the
  edge.
- **Your record:** wins, losses and draws against each level, kept on the device.
- No perfect level in v1. Without a large opening book, a perfect search from the early
  positions is far slower than a move a child will wait for, and a game that always wins isn't
  fun for kids.

### Why the first draft's numbers changed

Fable modelled the levels with a simple engine (an evidence check, not our engine):

- Easy as first drafted (always takes its win, blocks half the time) beat a randomly playing
  child about 9 games in 10, and a child who always spots their own four but never blocks won
  only a third of the time. Easy at 25% and 25% lets that child win about half.
- A depth- or position-capped search is **not** reliably stronger than Medium at every cap:
  capped at 20,000 positions it scored 32% to 44% against Medium, at 100,000 about 81%, at
  fixed depth 8 about 93%. That is why Hard's cap is measured and Hard is tested at the cap
  that ships.

### How the search is checked

The engine is a pure module (`src/engine.js`) used by both the Web Worker and the tests. The
time limit only applies in the browser; tests use the shipped position cap alone, so they give
the same answer on any machine.

## Playing a friend online

### Flow

1. **Start a room.** Pick your color (blue, orange or random) and who goes first (you, your
   friend or random). The app shows the code in large letters, a **Copy link** button, and
   **Share** on phones (the browser's own share sheet, so we load no third-party script), with
   one line: "Anyone with this code can join, so share it only with your friend."
2. **Join.** The friend opens the link (`/?room=CODE` fills in the code) or types the code into
   **Join a room**.
3. **Play.** Each player sees "Your turn" or "Waiting for your friend". Players show as "You"
   and "Friend" with their disc color. There are no names and no chat. When the other player
   is disconnected, you see "Your friend is away"; you can keep waiting or leave.
4. **Rematch.** When a game ends, each player can tap **Rematch**. When both have, a new game
   starts in the same room and the other player goes first. A small tally ("You 2, Friend 1")
   stays for the life of the room.
5. **Leave.** **Leave room** asks "Leave this room?" and then closes the room for both players.
   The friend sees "Your friend left" and can start a new room.

### Server

Ported from the chess Worker, with our rules in place of chess's. Rematch, leave, the tally and
the direct room lookup are new.

- **Create:** `POST /api/rooms` returns a 6-character code from 31 letters and digits that
  can't be confused (no 0, 1, I, L or O), 887,503,681 codes, plus a secret seat token.
- **Lookup goes to the room itself, no KV.** `GET /api/rooms/CODE` and the WebSocket at
  `/api/rooms/CODE/ws` check the code's shape and the rate limit first, then ask that code's
  Durable Object, which answers "no such room" if it has nothing stored. This departs from
  chess (owner said yes, 2026-11-10). KV is eventually consistent and caches a miss for up to 60 seconds
  per Cloudflare location, so a friend on another network could be told a brand-new code
  doesn't exist; a Durable Object is consistent at once. It also removes KV's 1,000 writes a
  day free limit. The cost: a well-formed guessed code wakes an empty object for a moment and
  counts one Durable Object request; nothing is stored for it.
- **Rate limits:** two rate-limit bindings, keyed by the client's IPv4 address or its IPv6 /64
  (one home connection has a whole /64): making a room, 3 a minute; looking up a code, 20 a
  minute. Cloudflare counts these per location and approximately, so they slow a guesser
  rather than stop one. The real backstop on the free plan is the daily request limit (see
  Cost): with 3 rooms waiting for a second player, a guesser using every one of the 100,000
  daily requests would find one about every 8 years (887,503,681 ÷ 300,000 ≈ 2,960 days), and
  in doing so would switch online play off for the day. A guesser who did find a room could only
  take its empty seat and play a game: there is nothing to read or say.
- **The server is the referee.** One Durable Object per room keeps the room in its storage
  (memory is lost when an idle room hibernates), uses the WebSocket Hibernation API so a room
  waiting for a move costs nothing, and checks every drop with the same pure rules module the
  browser uses (`src/rules.js`). It rejects a move out of turn, a column outside 0 to 6, or a
  full column, and it decides wins and draws.
- **Seats are fixed by disc color for the life of the room.** Each game records who moved
  first, which flips on every rematch; the side to move comes from that plus the number of
  drops. Unlike chess, where a seat's color and turn order are the same thing, nothing about a
  seat changes on a rematch, so an open socket always controls the same color.
- **Rematch** is accepted only after a game has ended, is stored per seat, and is cleared when
  the new game starts. Tapping twice from one seat starts nothing.
- **Leave** closes the room at once through the same path as expiry: both sockets closed and
  the room's storage deleted. Nobody can sit down in a seat a leaver gave up.
- **Messages:** `hello` (with the seat token, or none to take the empty seat), `drop` (a
  column), `rematch`, `leave`. At most 2 messages a second and 1,000 per connection, each at
  most 200 characters; past that the socket is closed. A whole game is at most 42 drops.
- **Seats and reconnect:** the browser keeps its seat token in `localStorage` for that room
  code. Coming back with it restores the seat and the whole game, and closes any older socket
  with the same token. A third person is turned away: there are no spectators. The browser
  deletes its token for a code once the room says it's closed or gone.
- **No clocks and no claiming a win.** These are friendly games, and phones drop their
  connection when they lock, so being away is never a loss. A player who won't move can be left.
- **Expiry:** a room no one joins closes after 1 hour, any room 24 hours after its last move,
  and every room within 7 days. Closing deletes the room's storage, so nothing about it is kept.
- **Offline:** if the device is offline or the Worker answers "unavailable", the online button
  says so and points to the computer and pass-and-play. Neither of those needs the network.

### Cost

Nothing here can create a bill on the Workers Free plan: past a limit, calls fail with an
error until 00:00 UTC (Cloudflare's Durable Objects pricing page, checked 2026-11-10). The
limits are per account, so they are **shared with chess** (its live rooms, idea box and coach
budget).

| Free-plan limit, per day, whole account | Value | What uses it here |
| --- | --- | --- |
| Worker requests | 100,000 | Every `/api` call and socket open |
| Durable Object requests | 100,000 | Room creates, lookups, socket opens, messages, alarms |
| Durable Object rows written | to confirm in build 2 | One save per drop, rematch and seat change |

Counting each socket message as one request (the safe reading: Cloudflare's 20-to-1 ratio for
socket messages is stated for billing, and whether it applies to the free limit is checked in
build 2), a game is about 50 requests, so honest use runs out somewhere between 1,000 and
2,000 online games a day.

Said plainly: one person with a script can use up the day's requests and switch online play
off until midnight UTC, for this game and for chess, including the chess coach. That's
accepted because nothing is charged, and the computer and pass-and-play keep working. Moving
the account to the paid plan removes this backstop, so before that a Cloudflare rate-limiting
rule goes on `/api/rooms*` first. The plan is the owner's call.

## Privacy (kids will play)

- No accounts, names, chat, analytics, ads, trackers or third-party scripts. The prignano-theme
  header loads from jsDelivr pinned to a commit SHA with a subresource integrity hash, as
  chess does, and is cached for offline use. A Content-Security-Policy header lists the only
  allowed sources.
- Settings, your record and seat tokens stay in `localStorage` on the device. "Delete my data"
  in settings clears them.
- Online rooms hold only the moves, two random seat tokens, the rematch flags and the tally,
  and are deleted when the room closes. The Worker keeps no request logs (observability off).
  The connecting address is used only to count requests for the rate limit; we don't store it.
- A short About screen says this in plain words, with the MIT notice and a link to the source.

## Installable app

Pinball PR #7's recipe: manifest (`display: standalone`, no orientation lock), original icons
in any, maskable and Apple touch sizes, iOS home-screen tags, a service worker that caches
every served file with its list stamped by `tools/sw-version.mjs` and a test that fails when
the list is stale, an Update button on the home screen, safe-area padding, the site header
hidden in the installed app, and sound resumed after an iPhone locks. The service worker never
caches `/api/*`.

## Hosting

One Cloudflare Worker named after the repo, serving the folder as static assets and running
`worker/index.js` only for `/api/*` (`run_worker_first`), with the `ROOM` Durable Object
(SQLite-backed, so it works on the free plan) and the two rate-limit bindings. No KV.
`.assetsignore` leaves out tests, tools and docs. Workers Builds deploys each merge to `main`.

The owner creates the repo and adds the custom domain. Claude writes `docs/DEPLOY.md` the way
chess's is written, and sets up the Worker on the account only with his yes (through the
Cloudflare connector, as was done for Infinite Recursion). The last step of `docs/DEPLOY.md`
is the `content/games.json` PR in `prignanodotai-web`: the game goes in as hidden with an empty
url when the repo is created, and the launch PR sets its url and `live`. Its text there calls
the game "four in a row" and nothing else.

## Files

```
index.html, styles.css, manifest.webmanifest, sw.js, _headers, icons/
src/rules.js      board, drops, win and draw checks (shared with the Worker)
src/engine.js     search and the three levels (pure)
src/ai-worker.js  runs engine.js off the main thread
src/board.js      SVG board, input, drop animation, screen-reader buttons
src/live.js       room create and join, WebSocket, reconnect
src/audio.js      Web Audio sounds
src/main.js       screens: home, computer, pass-and-play, online, settings, about
worker/index.js   /api/rooms routes, rate limits
worker/rooms.js   room rules with no Cloudflare in them (tested in Node)
worker/room.js    the Durable Object
wrangler.jsonc, .assetsignore, docs/DEPLOY.md, tools/sw-version.mjs
test/             node --test
```

## Tests

`node --test` covers:

1. **Rules:** discs fall to the lowest empty space; a full column is refused; wins across, up
   and down, and on both diagonals, including at every edge; no false win that wraps from one
   column to the next; a full board with no four is a draw; a check against a slow,
   obviously-correct reference on 10,000 random positions.
2. **Computer** (all seeded, so the same on every run):
   - Medium and Hard always take a win in one and always block a single threat.
   - A scripted beginner (always takes its own four, never blocks, otherwise random) scores at
     least 40% against Easy over 2,000 games (wins plus half of draws).
   - Medium beats Easy at least 90% of the time over 200 games with the first player alternating.
   - Hard, at the cap that ships, plays Medium over a game count that runs in about a minute in
     CI. The threshold is 75% only if build 1 measures at least that on three seeds; otherwise
     the build thread reports the measured score and the owner picks the threshold.
   - The same seed always gives the same moves, and Hard stays within its cap.
3. **Rooms** (`worker/rooms.js` in Node, ported from chess's room tests): seats and join, a
   third person turned away, moves out of turn or into a full or missing column refused, the
   server finds wins and draws, the room's view never includes the other player's seat token,
   expiry times. New: after a rematch each seat still controls the same color and the other
   player moves first; rematch is refused during a game; one player tapping twice starts
   nothing; the tally counts; a leave closes the room.
4. **Words:** a short banned list (the trademarked name in its spellings, its owners' names,
   and the unambiguous names of its variants), stored encoded in the test, appears in no served
   file (everything not in `.assetsignore`). The docs may name the product when describing
   prior art.
5. **PWA:** the service worker's file list is current (from pinball).

**Done means:** all tests pass in CI, and before each build PR the build thread runs a scripted
Playwright check (not in CI, like chess) at phone and desktop sizes: a full game against each
level (logging Hard's positions per move at phone size), a pass-and-play game, and under
`wrangler dev` a two-browser online game with a rematch, a reconnect and a leave. Installing
and playing offline on real phones is a manual check listed in `docs/DEPLOY.md`. The
cross-network join (two devices on different networks) is part of that manual check, since
`wrangler dev` can't show it.

## Pull requests

After the owner creates the repo:

1. **Spec:** this file as `docs/SPEC.md`, `AGENTS.md`, `CLAUDE.md`, MIT `LICENSE`. Plus the
   hidden `games.json` entry in `prignanodotai-web`.
2. **Build 1, the game on one device:** rules, board, art, sound, the three computer levels
   with Hard's measured cap, pass-and-play, your record, About, the PWA recipe,
   `wrangler.jsonc` (static only for now), `docs/DEPLOY.md`, CI, tests 1, 2, 4 and 5.
3. **Build 2, online rooms:** the Worker, Durable Object and rate limits, create and join,
   share link, reconnect, rematch, leave, test 3, and the confirmed free-plan limits written
   into Cost.

Then the Cloudflare setup (with the owner's yes) and the launch `games.json` PR.

## Not in v1

A perfect "unbeatable" level, other board sizes or rule twists, timers, claiming a win from an
absent player, spectators, preset reactions or any chat, a shared leaderboard, matchmaking with
strangers, accounts, a shared rooms service for other games (copying the room code is about
300 lines; sharing it waits until a third game needs rooms), music. A later twist takes neither
the name nor the exact rule set of one of the classic product's named variants.

## Questions (answered)

All answered by the owner on 2026-11-10: the name below, and "defaults for the rest".

1. **Name.** Answered: **Drops** at `drops.prignano.ai`, repo `nickprignano/drops` (owner,
   2026-11-10). A quick search of app stores, Steam and itch.io before the spec PR; a close match
   in the same genre goes back to the owner.
2. **Pass and play on one device.** Answered: **yes**. It costs almost nothing and suits two
   kids at a table. (Chess dropped it, because chess has the computer and live rooms; here it
   is the simplest way for siblings to play.)
3. **A twist.** Answered: **none in v1**: the classic 7 × 6 game, done well. A twist can be a
   second mode later, like Divide in Infinite Recursion.
4. **Who goes first online.** Answered: the room's creator picks (default "random"), and it
   switches on each rematch.
5. **Room lookup without KV.** Answered: **yes**: ask the room's Durable Object directly instead
   of checking KV first as chess does, so a friend is never told a new code doesn't exist.

## Fable red team (2026-11-10)

| ID | Finding | Severity | Status |
| --- | --- | --- | --- |
| RT-1 | Easy as drafted beat a randomly playing child about 9 games in 10 | High | Fixed: Easy takes wins and blocks 25% each; beginner test at 40% |
| RT-2 | Hard's 75% threshold, its 2 million cap and the 200-game test didn't hold together (deterministic engines play only 2 distinct games; the test would take hours; a phone would ship a weaker Hard than tested; a capped search isn't always stronger) | High | Fixed: seeded tie-breaks, cap measured on a phone, Hard tested at the shipped cap, threshold confirmed by measurement or set by the owner |
| RT-3 | Rematch and leave aren't in chess, and a straight port (seat = color = turn) breaks on the first rematch | Medium | Fixed: seats fixed by color, first mover recorded per game, rematch and leave rules, token cleanup, new tests |
| RT-4 | The 3-minute claim let the player to move win because the other phone locked | Medium | Fixed: claim removed; "Your friend is away" and Leave instead |
| RT-5 | KV lookup-first can tell a friend "no such room" for up to a minute (eventual consistency, cached misses) | Medium | Fixed: lookup asks the Durable Object (owner said yes, question 5) |
| RT-6 | The "42 years" guessing claim rested on a per-address limit that IPv6 and a paid plan remove | Medium | Fixed: IPv6 keyed by /64, separate create limit, honest backstop figure, paid-plan rule, share-only-with-your-friend line |
| RT-7 | Cost counted one free-plan limit of several; KV writes (1,000 a day) were the tightest; limits are shared with chess | Medium | Fixed: limits table, KV dropped, shared-quota risk stated, 2 messages a second, rows-written to confirm in build 2 |
| RT-8 | License row's reason overstated who holds copyright and skipped pinball's GPL header | Low | Fixed |
| RT-9 | Words test said the repo never spells the name, but the docs do; variant names weren't covered | Low | Fixed |

Kept as is after the red team: the 6-character codes, no chat, names or spectators, the
2 colors with marks on a flat tray, the 7 × 6 rules, Medium at depth 4, the Web Worker and one
pure engine module, no clocks, the free plan, hibernation, MIT.
