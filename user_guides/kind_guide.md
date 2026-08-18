# KIND*AI on the BERDL Lakehouse — User Guide

KIND*AI is the graphical, natural-language front door to the lakehouse: ask a
research question and watch an AI co-scientist (a KOROS/Claude Code session)
work it — plan, decisions, and results surfaced in a web UI — plus one-click
data portal apps (Lakehouse Explorer, Diaspora, ENIGMA Strata, Function
Junction, Fungal Jungle, GenePool, genKnown, Plant Terra). No terminal, no
installation: it's built into the notebook image and starts automatically.

## Who gets it

Users holding a KIND role in KBase Auth2 — **`KBASE_STAFF`** or
**`BERDL_KIND_USER`**. Everyone else keeps the plain JupyterLab experience,
unchanged.

## Getting in

1. Go to **https://hub.berdl.kbase.us** and log in.
2. Your server starts (KIND boots with it and pre-warms the app portals).
3. You land on the KIND UI: `https://hub.berdl.kbase.us/user/<you>/proxy/8770/`.

If you land on JupyterLab instead despite holding the role: **log out of the
hub, log back in, then stop + start your server** (Hub Control Panel) — in
that order; the role is stamped into your hub session at login. Only the hub
session needs it — your KBase login is untouched, so logging back in is one
click.

**JupyterLab hasn't gone anywhere:** use the Jupyter button in KIND's top
bar, or go to `https://hub.berdl.kbase.us/user/<you>/lab` directly — explicit
`/lab` URLs always take you to Lab.

## One-time setup: an LLM credential

Driving sessions requires a credential **BERDL does not provide**. Either:

- run `claude` in a hub terminal and log in with a Claude Pro/Max account, or
- put a CBORG key in `~/.env`:
  `CBORG_API_KEY=...` (and `CBORG_BASE_URL=https://api.cborg.lbl.gov/v1`).

KIND's **⚙ doctor** panel tells you exactly what's missing.

## Where your work lives

Everything survives pod restarts and idle culls: research arcs under
`~/koros/runs`, KIND session state under `~/.local/share/king/runs`, both on
your persistent home. Stopping your server kills an *in-flight* agent turn,
but recorded arc state is resumable.

## Troubleshooting

| Symptom | Fix |
|---|---|
| I land on JupyterLab though I have the role | Log out of the hub → log in (one click — your KBase login persists) → stop + start your server (Hub Control Panel), **in that order** |
| Error page right after my server starts | KIND is still booting (~15 s) — reload |
| KIND looks old / no Jupyter button in the top bar | An earlier manual KING install is shadowing the built-in one. In a hub terminal run: `for f in ~/.local/bin/king* ~/.local/bin/kind*; do if [ -L "$f" ] && [[ $(readlink "$f") == $HOME/king/* ]]; then rm -v "$f"; fi; done` then stop + start your server once more. Your `~/king` checkout and all research state stay untouched. (Skip this if you're actively developing KING — it's your build, on purpose.) |
| "KBase token expired" banner though I just logged in | Stop + start your server; if it persists, log out/in first |
| An app tile won't load | Give it ~45 s (cold start), then relaunch from the switchboard |


## For KING developers

Manual development is unchanged: `serve.sh` from your own checkout still
works, and the built-in autostart defers to any server you're already
running. Note the flip side — leftover `~/.local/bin/king*` symlinks from an
old install shadow the built-in KIND (see the troubleshooting row above).
