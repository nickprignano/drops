# Instructions for AI assistants

Drops is an original four-in-a-row browser game: plain JavaScript and SVG, no build step, served
as static files plus one Cloudflare Worker for online rooms. The plan is in
[docs/SPEC.md](docs/SPEC.md); read it first.

- It's original work under the MIT license. No name, look, variant names or trademarks from the
  classic boxed game, and nothing copied from it beyond the genre rules. The words test keeps the
  banned names out of served files.
- Made for all ages, and kids play it online. No accounts, names, chat, spectators, analytics or
  third-party scripts. Online rooms keep only what the spec lists and delete it when they close.
  A change that adds any of these, or weakens the room rate limits or message limits, needs the
  owner's explicit yes.
- The server is the referee: every move is checked by the shared rules module in the Worker.
- Nothing leaves the player's device except moves in an online room and the pinned theme files.
- Never put API keys, account or tenant IDs, domains other than the game's own, IPs, emails,
  secrets, or employer, customer or private-project names in the repo, commits, issues or PR
  text.
- Never ask for, show or store a secret in chat. The owner runs the step that handles it.
- Propose changes as pull requests. Never change a real account, deployment or spend setting
  without the owner's explicit yes.

## How to make a change

Four rules after the community "Karpathy guidelines" (MIT), reworded for agents that work unattended.

- Say what you assumed. When the ask leaves a detail open, pick the reasonable default, say which,
  and keep going. Stop to ask only when the choice changes the goal, something the owner will
  notice, or a step no one can undo; if no one is there to answer, do the reversible part and leave
  the question in the PR body. If a simpler way exists than the one asked for, say so.
- Write the least code that does the job: no features, options or abstractions nobody asked for,
  and no handling for cases that can't happen.
- Touch only what the task needs and match the style around it. Every changed line should trace
  back to the ask. Mention unrelated dead code instead of deleting it; remove only what your own
  change left unused.
- Before you start, name the check that proves the task is done (a failing test, the "Checks to
  run" below, an expected output), and don't call it done until it passes in front of you.

## Checks to run

```bash
node tools/sw-version.mjs   # after changing any file the site serves (from build 1)
node --test
```
