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

### Pressure, not load average

The `STALL` row is [PSI](https://docs.kernel.org/accounting/psi.html): the share
of the last ten seconds in which tasks were blocked waiting for CPU, memory or
disk. It is what the warnings are built on, because
[load average counts uninterruptible tasks](https://lwn.net/Articles/759658/)
and so cannot tell CPU contention from IO stall. Memory `full` pressure is the
kernel's own definition of thrashing, which is exactly the state WSL2 lands in
when it runs out of headroom. On a kernel without PSI, wslps falls back to the
old load-average and D-state guesses.

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
| `wslps idle` | Processes doing nothing, or left behind by a closed shell |
| `wslps projects` | What each project costs, and the ports it owns |
| `wslps doctor` | Whether this WSL box is configured to survive your work |
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
| `wslps kill project <name>` | Stop everything running in one project |

Flags: `-y` skip confirmation, `-9` SIGKILL instead of SIGTERM, `-d` dry run.

## The useful bits

### `wslps dash` is the live view

```
 ⠏ WSLPS overview   Mon 31 Aug 13:44:22  up 6h39m  every 2s
 RAM █████████████▏░░░░░░░░░░░░   51%     4.9G of 9.7G     ▄▄
 SWP ██████▌░░░░░░░░░░░░░░░░░░░   25%     2.0G of 8.0G     ▂▂
 CPU █░░░░░░░░░░░░░░░░░░░░░░░░░    4% load 0.30 0.70 0.94 on 8 cores ▁▁
 all clear

 LISTENING PORTS
     PORT      PID      RAM   CPU%  PROCESS
       53        -        -   0.0%  (another user - run: sudo wslps ports)
     3111    76147     1.7G   0.0%  next-server (v15.5.12)
     5432        -        -   0.0%  (another user - run: sudo wslps ports)
     9009    69952    72.6M   0.0%  node ~/.npm/_npx/6ddf87659f2ad8a4/node_modules/.bin/mcp-ser...

 GROUPS
  GROUP        COUNT       RAM     SWAP   CPU%  SHARE
  dev-server       4      2.0G        -   0.0%  ███████░░░░░░░░░░░░░░░░░
  browser         26      1.5G     535M  11.5%  ██████░░░░░░░░░░░░░░░░░░
  ai-agent         5      1.5G     162M  10.0%  █████░░░░░░░░░░░░░░░░░░░
  mcp             31      922M     1.2G   0.0%  ███░░░░░░░░░░░░░░░░░░░░░
  node             2      223M        -   0.0%  █░░░░░░░░░░░░░░░░░░░░░░░
  ... 2 more, press 5

 TOP BY RSS
      PID      RAM             SWAP   CPU%         UPTIME GROUP      COMMAND
❯   76147     1.7G ████████       -   0.0% ░░░░░      46m dev-server next-server (v15.5.12)
    69791     512M ██░░░░░░       -   4.0% ███░░      54m ai-agent   claude
      783     407M ██░░░░░░   71.0M   5.5% █████    6h33m ai-agent   claude
    22975     300M █░░░░░░░   91.5M   0.5% ░░░░░    4h25m ai-agent   claude
   119207     259M █░░░░░░░       -   6.0% █████       1m browser    node ~/projects/budget-...
    64879     255M █░░░░░░░       -   0.0% ░░░░░    1h12m ai-agent   codex
   119255     187M █░░░░░░░       -   0.5% ░░░░░       1m browser    chrome-headless-shell -...
  ... 8 more

 up/dn move  x kill  t tree  s sort:rss  / filter  1-5 views  + - rate  p pause  ? help  q quit
```

The overview is the one-shot report, live: the same ports, groups and top
processes blocks, in the same order. Each block gives up rows to the one below
it as the window gets shorter, so nothing is ever cut off the bottom - press
`4` or `5` to see a trimmed block in full.

It samples every couple of seconds and animates between samples, so a bar
moving is a real change rather than a screen that blinked. The three sparklines
on the right are the last 32 samples on a fixed 0-100% scale, so a flat 40% RAM
looks flat instead of pegged.

What it shows that the one-shot report cannot:

- **real CPU per process** - the delta in `/proc/<pid>/stat` between two
  samples, not the lifetime average `ps` reports, so a process that was busy an
  hour ago no longer looks busy now
- **disk IO per process** - `read_bytes` and `write_bytes` from
  `/proc/<pid>/io`, so the `IO/s` column is real block IO rather than page-cache
  hits. It answers the question the `io` pressure figure raises
- **new processes** - a pid that appeared since the last sample is highlighted
- **a colour per group** - `mcp` red, `ai-agent` orange, `dev-server` azure,
  `browser` blue, `editor` violet, `database` gold, `webserver` teal, `node`
  green, `other` grey. The same colour follows a group into every table, in the
  dashboard and in the one-shot report; `wslps help` prints the legend
- **the same five views** you already have as commands, on keys `1`-`5`

| Key | Does |
| --- | --- |
| `1` `2` `3` `4` `5`, `tab` | Overview, processes, idle, ports, groups |
| arrows or `j` `k`, `pgup` `pgdn` | Move the selection |
| `s` | Sort by RAM, CPU, swap, disk IO or uptime |
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

### `wslps projects` answers "which checkout is costing me"

```
PROJECTS (processes grouped by the project directory they run in)
   PROCS       RAM      SWAP  PORTS          PROJECT
      13      3.1G     72.6M  3111           ~/projects/budget-tracker-2026
      33      672M      1.3G  -              ~/ag-projects/whitelabel-sites
       8     34.2M         -  -              ~/projects/wslps
      24      1.5G      437M  -              (not in a project)
      57      318M     22.1M  -              (cwd not readable - other users)
     135      5.6G      1.9G
```

Every process is attributed to the directory it was started in, walked up to
the nearest `.git`, `package.json`, `go.mod`, `Cargo.toml` or `composer.json`.
The five node processes behind one dev server collapse onto one row, with the
port it serves next to it.

The two buckets at the bottom are deliberate. `(not in a project)` is a process
whose working directory has no project marker above it; `(cwd not readable)` is
another user's process, which the kernel will not let you inspect. Neither is
guessed at.

`wslps kill project <name>` stops everything in one checkout, with the same
preview, confirmation and guards as every other kill.

### `wslps doctor` checks the box, not the processes

```
WSL DOCTOR (configuration, not processes - nothing here changes anything)

  CONFIG
    file                 /mnt/c/Users/253071.CDiomampo/.wslconfig
    memory               10GB
    processors           8
    swap                 8GB
    swapFile             D:\\wsl-swap.vhdx
    autoMemoryReclaim    gradual
    sparseVhd            true
    guest sees           9.7G RAM, 8 cores, 8.0G swap

  HOST
    windows RAM          31.7G
    WSL cap              10.0G = 31% of host RAM

  DISK
    ext4.vhdx            124G on disk   /mnt/d/wsl/Ubuntu/ext4.vhdx
    used inside          112G of 1007G
    host drive free      47.6G

  PROJECTS
    on windows fs        none - everything is on ext4

  PRESSURE  (stalled share of the last 10s)
    cpu / mem / io       0.0%  0.0%  0.0%

  FINDINGS
    ok    autoMemoryReclaim=gradual - freed memory returns to Windows
    ok    sparseVhd=true - the VHDX shrinks as you delete files

  edit /mnt/c/Users/253071.CDiomampo/.wslconfig then run: wsl --shutdown  (from Windows) to apply
```

Everything else in wslps answers "what is running". This answers "is this
instance set up to survive it", which is the question you have at 3am after the
box has already fallen over:

- the `.wslconfig` memory cap, read from Windows and compared against what the
  guest actually sees and against host RAM. No cap means WSL helps itself to
  [half of the host](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)
- whether `autoMemoryReclaim` is on, since without it
  [freed memory never goes back to Windows](https://devblogs.microsoft.com/commandline/memory-reclaim-in-the-windows-subsystem-for-linux-2/)
- whether `sparseVhd` is on, and how much room the VHDX has left to grow on its
  host drive. It grows on demand and [does not shrink on its own](https://vramlab.com/posts/wsl2-sparse-vhd-cannot-compact/)
- any project sitting under `/mnt`, where every file operation crosses the 9P
  bridge and metadata-heavy work runs
  [orders of magnitude slower](https://dev.to/nomurasan/why-wsl2-is-slow-on-mntc-and-how-to-find-the-exact-operation-costing-you-time-40o7)

It reads and reports. It changes nothing.

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

```
IDLE - alive a while, doing nothing, or left behind (kill candidates)
  idle: up > 20 min, CPU < 1% right now, RAM > 20M
  orphan: its parent is gone, so nobody is watching it any more
      PID      RAM     SWAP    CPU%   UPTIME  WHY         COMMAND
    76147     2.2G        -    0.0%    2h04m  idle        next-server (v15.5.12)
    30218     179M     334M    0.0%    5h21m  idle        chrome --type=renderer --crashpad-handler...
    76121     164M        -    0.0%    2h04m  idle        node ~/projects/budget-tracker-2026/node_...
    76337     157M        -    0.0%    2h04m  idle        node ~/projects/budget-tracker-2026/.next...
    76084     150M        -    0.0%    2h04m  idle        pnpm dev
```

Two different questions, one list:

- **idle** - alive over 20 minutes, over 20M resident, and using no CPU *right
  now*. The CPU figure is a delta between two samples taken 300ms apart, not
  the lifetime average `ps` reports, so a process that was busy an hour ago and
  is quiet now is correctly listed
- **orphan** - its parent is gone, which normally means you closed the terminal
  it was started in and it kept running. Only groups you start by hand are
  considered: plenty of system daemons sit at ppid 1 by design and are not
  abandoned at all

The dashboard's idle view (key `3`) runs the same predicate and shows the same
`WHY` column, so the two never disagree about what counts as a kill candidate.

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
- **`idle` costs 300ms.** It takes two snapshots to get a real CPU number, so it is slower than the other one-shot commands by exactly that gap.
- **Orphan detection looks for ppid 1.** Under a process supervisor or an agent that makes itself a subreaper, an abandoned process is reparented to that instead of to init, and will not be flagged.
- **`IO/s` only covers your own processes.** `/proc/<pid>/io` needs ptrace access; other users' processes report `-`. The read is skipped entirely for commands that do not show the column.
- **`projects` can only see your own processes.** A working directory is readable for processes you own; anything else lands in the `(cwd not readable)` bucket rather than being guessed at.
- **`doctor` needs Windows interop** for host RAM, the active Windows profile and the VHDX location. Without it, it falls back to searching for a `.wslconfig` and says so when more than one profile has one, and the VHDX is matched by distro name in the install path or reported as not located. It never pairs another distro's disk with this one's usage.
- **PSI needs a kernel built with `CONFIG_PSI`.** Every current WSL2 kernel has it. Without it, wslps falls back to load average and the D-state count.
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
