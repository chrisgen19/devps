# wslps

[![ci](https://github.com/chrisgen19/wslps/actions/workflows/ci.yml/badge.svg)](https://github.com/chrisgen19/wslps/actions/workflows/ci.yml)

See what is running inside your WSL2 instance, what it actually costs, and stop it safely.

A single dependency-free bash script. No install step beyond dropping it on your `PATH`.

```
 WSL RESOURCES  Fri 28 Aug 15:19  up 5 hours, 15 minutes

  RAM    ###################.....  79%   7.7G used of 9.7G  (2.0G available)
  SWAP   ######################## 100%   8.0G used of 8.0G  (1.1M free)
  LOAD   ##......................   7%   0.58 / 1.58 / 13.28  across 8 cores

  ! swap is 99% full - the box is paging heavily
  try: wslps idle    then: wslps kill port <n>

GROUPS
  GROUP         COUNT       RAM      SWAP
  dev-server       12      3.8G      3.5G   ############
  ai-agent         11      1.8G      387M   ######
  editor           25      1.1G      1.6G   ####
  mcp              37      673M      1.5G   ##
  browser          13      247M      633M   #
  database         15      139M     30.4M
  other            82     99.3M     62.1M
  node              5      284K      293M
  TOTAL           200      7.9G      8.0G
```

## Why

WSL2 runs behind a memory cap, and when you cross it the box does not fail loudly. It swaps, then it thrashes, then everything feels broken for reasons `htop` does not make obvious.

The usual tools do not help much here:

- `free -h` tells you swap is full but not who filled it
- `htop` sorts by RAM, and the process eating your swap is often near the bottom of that list
- `ps aux | grep node` returns fifty rows of identical-looking node processes

`wslps` answers the question you actually have: **which of the things I started is costing me, and can I safely stop it.**

The `SWAP` column is the point. A dev server you have not touched in five hours can be sitting on 2.9G of swap while showing under 1G of RAM, which makes it nearly invisible to every other tool.

## Install

```bash
curl -fsSL https://raw.githubusercontent.com/chrisgen19/wslps/main/wslps -o ~/.local/bin/wslps
chmod +x ~/.local/bin/wslps
```

Make sure `~/.local/bin` is on your `PATH`. If it is not, add this to `~/.zshrc` or `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Or clone and symlink:

```bash
git clone https://github.com/chrisgen19/wslps.git
ln -s "$PWD/wslps/wslps" ~/.local/bin/wslps
```

## Commands

### Looking

| Command | What it does |
| --- | --- |
| `wslps` | Full report: memory, ports, groups, top processes |
| `wslps top [N]` | N biggest processes by RAM (default 15) |
| `wslps swap [N]` | N biggest swap hogs (default 15) |
| `wslps idle` | Long-running processes doing nothing |
| `wslps group <name>` | Every process in one group |
| `wslps ports` | Everything listening |
| `wslps port <n>` | Who owns port n, its project dir, its tree |
| `wslps tree <pid>` | Process tree under a pid |
| `wslps dash [secs]` | Live interactive dashboard, `q` to quit (default 2) |
| `wslps watch [secs]` | Alias for `dash` |
| `wslps help` | All of the above |

### Stopping

| Command | What it does |
| --- | --- |
| `wslps kill port <n>` | Stop whatever serves port n, plus its children |
| `wslps kill pid <n>` | Stop one process plus its children |
| `wslps kill group <name>` | Stop every process in a group |

Flags: `-y` skip confirmation, `-9` SIGKILL instead of SIGTERM, `-d` dry run.

## The useful bits

### `wslps dash` is the live view

```
 ⠋ WSLPS overview   Mon 31 Aug 13:15:17  up 6h10m  every 2s
 RAM ██████████▋░░░░░░░░░░░░░░░   41%     4.0G of 9.7G     ▃▃
 SWP ██████▋░░░░░░░░░░░░░░░░░░░   26%     2.0G of 8.0G     ▂▂
 CPU ▊░░░░░░░░░░░░░░░░░░░░░░░░░    3% load 1.74 3.63 2.69 on 8 cores ▁▁
 all clear

 TOP BY RSS
      PID      RAM             SWAP   CPU%         UPTIME GROUP      COMMAND
❯   76147     1.6G ████████       -   0.0% ░░░░░      17m dev-server next-server (v15.5.12)
    69791     496M ██░░░░░░       -   6.0% █████      25m ai-agent   claude
      783     378M ██░░░░░░   80.5M   2.5% ██░░░    6h04m ai-agent   claude
    22975     300M █░░░░░░░   91.7M   2.0% ██░░░    3h56m ai-agent   claude
    64879     260M █░░░░░░░       -   5.5% █████      43m ai-agent   codex
    76121     165M █░░░░░░░       -   0.0% ░░░░░      17m dev-server node ~/projects/budget-...
    76084     155M █░░░░░░░       -   0.0% ░░░░░      17m node       pnpm dev
    69936     143M █░░░░░░░       -   0.0% ░░░░░      25m mcp        chrome-devtools-mcp
    76337     133M █░░░░░░░       -   0.0% ░░░░░      16m dev-server node ~/projects/budget-...
    69935     108M █░░░░░░░       -   0.0% ░░░░░      25m mcp        node ~/.npm/_npx/9833c1...

 up/dn move  x kill  t tree  s sort:rss  / filter  1-5 views  + - rate  p pause  ? help  q quit
```

It samples every couple of seconds and animates between samples, so a bar
moving is a real change rather than a screen that blinked. The three sparklines
on the right are the last 32 samples on a fixed 0-100% scale, so a flat 40% RAM
looks flat instead of pegged.

What it shows that the one-shot report cannot:

- **real CPU per process** - the delta in `/proc/<pid>/stat` between two
  samples, not the lifetime average `ps` reports, so a process that was busy an
  hour ago no longer looks busy now
- **new processes** - a pid that appeared since the last sample is highlighted
- **the same five views** you already have as commands, on keys `1`-`5`

| Key | Does |
| --- | --- |
| `1` `2` `3` `4` `5`, `tab` | Overview, processes, idle, ports, groups |
| arrows or `j` `k`, `pgup` `pgdn` | Move the selection |
| `s` | Sort by RAM, CPU, swap or uptime |
| `/` | Filter on command, pid or group (empty line clears it) |
| `x` | Stop the selection - `X` for SIGKILL |
| `t` | Process tree for the selection |
| `p` `r` | Pause sampling, resample now |
| `+` `-` | Refresh interval |
| `?` `q` | Help, quit |

`x` and `t` drop out of the dashboard and run the ordinary `wslps kill` and
`wslps tree`, so the preview, the confirmation and every guard below still
apply. The dashboard itself never signals anything.

Colour and block glyphs turn themselves off when the output is not a terminal,
when `NO_COLOR` is set, or when the locale is not UTF-8. `WSLPS_ASCII=1` keeps
the colour but draws with `#` and `.`. Over a pipe, `wslps dash` falls back to
reprinting the plain report on a timer.

### `wslps port <n>` finds the project, not just the process

```
PORT 3111
  listener pid : 105371
  command      : next-server (v15.5.12)
  started      : Fri Aug 28 12:40:37 2026
  project dir  : /home/you/projects/budget-tracker
  server root  : 105283  <- what "wslps kill port 3111" targets
```

It walks **up** from the socket holder to the top of the server, so `kill port 3111` takes the whole `pnpm dev` -> `sh -c next dev` -> `next-server` -> postcss workers chain instead of orphaning the wrapper. It stops climbing at a bare interactive shell, so it can never walk up into your terminal.

### `wslps idle` is the "what did I forget about" view

Lifetime CPU under 1%, alive over 20 minutes, over 20M resident:

```
      PID      RAM     SWAP    CPU%   UPTIME  COMMAND
   171822    93.0M     2.2M    0.1%      44m  node ~/projects/budget-tracker/.next/pos...
     4233    30.0M     298M    0.2%    5h05m  chrome-devtools-mcp
    87286    21.3M    76.5M    0.1%    2h53m  npm exec chrome-devtools-mcp@latest ...
  stopping all 8 would return 370M RAM and 510M swap
```

### `wslps swap` shows who is actually paged out

The process making your machine feel slow is usually not the one at the top of `htop`.

## Kill safety

Every kill previews the exact victims and how much they free, then asks:

```
WILL STOP  (SIGTERM)
      PID      RAM   UPTIME  COMMAND
   105283     128K    2h37m  pnpm dev
   105332      12K    2h37m  sh -c next dev --turbopack --port 3111
   105371     2.7G    2h37m  next-server (v15.5.12)
   105541    58.0M    2h37m  node ~/projects/budget-tracker/.next/postcss.js ...
  9 process(es), frees about 3.0G RAM and 711M swap

Proceed? [y/N]
```

Three guards, in order:

1. **pid 1 is refused outright.** `refusing to kill pid 1 (init) - that would take down WSL`
2. **Your own session is refused.** `pid 175132 is your own shell or session - refusing`
3. **Protected subtrees are never expanded.** `kill group ai-agent` reaps other agents but leaves the session you are typing in, and its children, alone.

Guard 3 matters more than it looks. Without it, `kill pid 1` would skip init and then expand to every one of its descendants, which is the entire system.

Use `-d` first when you are unsure. The preview is the safety net.

## Groups

Processes are bucketed by command line, first match wins:

| Group | Matches |
| --- | --- |
| `mcp` | Anything with `mcp` in the command line |
| `ai-agent` | claude, codex, cursor-agent |
| `dev-server` | next, vite, nuxt, astro, webpack, remix, turbo |
| `browser` | chrome, chromium, firefox, playwright, puppeteer |
| `editor` | vscode-server, cursor-server |
| `database` | postgres, mysqld, mariadb, redis, mongod |
| `webserver` | apache2, nginx, php-fpm, httpd |
| `node` | Any other node, npm, pnpm, yarn, bun |
| `other` | Everything else |

Kernel threads are excluded.

## Notes and caveats

- **Group RAM totals over-count slightly.** Forked processes share pages that get counted once per process, so `TOTAL` sits a little above `free -h`. Treat the numbers as relative, not exact.
- **`kill group` is broad by design.** `kill group ai-agent` can target fifty-plus processes. Preview with `-d`.
- **`-9` skips cleanup.** Next dev servers flush caches on SIGTERM. Try the polite signal first; `wslps` only suggests `-9` if something ignored it.
- **Ports owned by other users show `-` for pid.** Run `sudo wslps ports` to see them.
- **`idle` is a heuristic.** It uses lifetime average CPU, so a process that was busy an hour ago and is idle now may not appear.
- Set `NO_COLOR=1` for plain output when piping or logging.

## Requirements

- bash 4+
- `procps` (`ps`, `pgrep`)
- `iproute2` (`ss`)
- awk

All present by default on Ubuntu and Debian WSL images.

## Related

If WSL2 is panicking rather than just thrashing, the cause is usually Windows host **commit** exhaustion rather than guest RAM. Cap the VM in `%USERPROFILE%\.wslconfig`:

```ini
[wsl2]
memory=10GB
processors=8
swap=8GB

[experimental]
autoMemoryReclaim=gradual
sparseVhd=true
```

Verify the host side with `Get-CimInstance Win32_PageFileUsage` (active pagefiles) rather than `Win32_PageFileSetting` (merely configured). A pagefile registered on a non-system drive can be silently deleted at every boot and contribute nothing to the commit limit.

## Development

```bash
shellcheck --severity=style wslps scripts/bump scripts/selftest
scripts/selftest
```

`scripts/selftest` is the whole test suite and runs anywhere. It only ever
signals throwaway processes it starts itself. Alongside the read-only commands
and input validation, it asserts the kill guards still hold: that pid 1 is
refused, that the caller's own session is refused, that a dry run signals
nothing, and that a real kill actually reaps a throwaway process.

Both the ci and release workflows call that same script, so a tagged release
cannot publish a build whose guards have regressed. Tag pushes do not trigger
`ci.yml`, and GitHub will not make the release job wait on a run for the same
commit on `main`, so the release workflow has to run the suite itself.

### Releasing

```bash
scripts/bump patch      # 2.0.0 -> 2.0.1, commits and tags
git push origin main && git push origin v2.0.1
```

Pushing a `v*` tag builds a GitHub Release. The release workflow refuses any tag
that disagrees with `VERSION` inside the script, so a published build can never
report a version it is not.

## License

MIT. See [LICENSE](LICENSE).
