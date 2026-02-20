# Trading as File System (TaFS)

*`ls` is your query. `touch` is your API. `rm` is your cancel.
Standard Unix tools are the client.*

TaFS is an experimental architecture that maps trading intent,
position state, and account state directly onto a filesystem.

Instead of imperative API calls, TaFS treats trading as a **declarative
desired-state system**.

------------------------------------------------------------------------

## Quick Example

    /tradefs/rakuten
      spot/
        orders/
          7203.1.2858
        pf/
          7203.2.2850
        cash.1200000

      margin/
        orders/
          6758.-1.2950.o
        pf/
          6758.-1.2940
        marginfree.900000

-   Creating files expresses **intent**
-   File presence expresses **state**
-   Rename/delete expresses **lifecycle**
-   A reconciliation engine converges broker reality toward filesystem
    state

------------------------------------------------------------------------

# 6H1W

## What

A filesystem-native declarative trading control plane.

## Why

-   eliminate SDK/API coupling
-   maximize auditability
-   unify strategy, execution, and state
-   enable AI-native automation
-   leverage OS primitives (atomic rename, permissions, watchers)

## Who

-   discretionary traders
-   quant developers
-   infra engineers
-   AI trading agents

## When

Most useful in semi-systematic discretionary trading and personal
trading infrastructure.

## Where

Local machine, VPS, network FS, or FUSE virtual filesystem.

## How

A reconciliation engine monitors filesystem changes and synchronizes
broker state accordingly.

## How Much

Minimal: POSIX filesystem + watcher + execution engine.

------------------------------------------------------------------------

# Design Philosophy

## Declarative Trading

Orders describe **desired exposure** instead of imperative actions.

## Filesystem as Control Plane

Filesystem becomes the single source of truth for intent and state.

## Separation of Intent and State

    orders/ = desired
    pf/     = actual
    git log = history

## Idempotency

Repeated reconciliation produces identical results.

## Unix-native Semantics

-   create → submit
-   delete → cancel
-   rename → state transition
-   commit → checkpoint

------------------------------------------------------------------------

# Filename Grammar

## Orders

    <code>.<signed_lot>[.<price>][.<suffix>]

-   signed lot represents target exposure
-   `p` used for decimal prices
-   suffix: `o` open, `c` close (margin), `stp` stop

## Positions

    <code>.<signed_lot>.<avg_price>

-   positive = long
-   negative = short
-   zero → file removed

## Account State

    cash.1200000
    marginfree.900000
    equity.2300000

------------------------------------------------------------------------

# Advanced Orders

A file is an atomic order. A directory is a compound order.

## Stop

    orders/7203.1.2800.stp

Same grammar as basic orders — type expressed as a suffix, consistent
with margin mode (`.o`, `.c`).

## OCO (One Cancels Other)

    orders/7203.oco/
      -1.2900
      -1.2800.stp

The directory name carries the instrument code. Children inherit it
and only specify what differs: lot, price, and type.

When one leg fills, the reconciliation engine cancels the other.

## IFDOCO (If Done, One Cancels Other)

    orders/7203.ifdoco/
      1.2858/
        -1.2900
        -1.2800.stp

The IF-leg (`1.2858/`) is a directory because it has dependents.
When it fills, its children activate as an OCO group.

Nesting depth = dependency depth. No sequence prefixes needed.

## Key Properties

-   **Code appears once** — the group directory scopes the instrument;
    children inherit it
-   **Nesting = dependency** — children activate when their parent fills;
    the filesystem tree *is* the execution graph
-   **Type as suffix** — `.stp`, `.o`, `.c` follow the same convention;
    no price = market, plain price = limit
-   **File = atomic order, directory = compound order** — one rule,
    no special cases

------------------------------------------------------------------------

# Order Lifecycle

The existing primitives already cover the full lifecycle — no
additional directories needed.

| Event    | Filesystem effect                                     |
|----------|-------------------------------------------------------|
| Submit   | file created in `orders/`                             |
| Fill     | file removed from `orders/`, `pf/` updated           |
| Cancel   | user deletes file from `orders/`                      |
| Reject   | engine deletes file from `orders/`, reason in commit message |

`orders/` is pure intent. `pf/` is pure state. `git log` is history.
The directory tree only reflects what is true right now.

Per-order risk management is expressed through compound orders — an
IFDOCO is a risk-managed position (entry with profit target and stop
loss built in).

------------------------------------------------------------------------

# Primitives

TaFS does not invent new infrastructure. It maps trading onto
primitives that already exist in POSIX filesystems and Git.

## Filesystem

| Primitive | Trading use |
|---|---|
| File create/delete | order submit / cancel |
| Atomic rename (`mv`) | state transition without race conditions |
| Directories | namespaces (broker, asset class) and compound orders |
| Nesting | dependency between order legs |
| Permissions (`chmod 444`) | lock an order — prevent accidental cancel |
| Timestamps (`mtime`) | submission time, last update — free metadata |
| Symlinks | aliases, cross-references between orders and positions |
| Extended attributes (`xattr`) | hidden metadata (broker order ID, notes) |
| File watchers (`fsevents`/`inotify`) | reconciliation engine trigger |
| File content | optional annotation (filename is identity, content is metadata) |
| Dotfiles | config and metadata that don't affect trading state |
| Mount points | different brokers as different mounts |

## Git

The TaFS directory is a Git repository. Git replaces custom logging
and provides audit, branching, and time-travel for free.

| Primitive | Trading use |
|---|---|
| Commits | reconciliation checkpoints — atomic state snapshots |
| Commit messages | event descriptions (fill, rejection reason) |
| `git log` | full audit trail — replaces `.log/` |
| `git diff` | what changed between any two points |
| `git blame` | what created or modified each order |
| `git revert` | undo a specific change |
| `git bisect` | find when something went wrong |
| **Branches** | `main` = live, `paper` = simulation, `backtest/2024q1` = replay |
| Tags | milestones — end of trading day, strategy change points |
| Hooks | `pre-commit` validates order grammar, `post-commit` triggers engine |
| Worktrees | multiple strategies running in parallel from one repo |

Branches are especially powerful: paper trading, backtesting, and live
trading are the same spec, same directory structure, different branches.
`git diff main..paper` compares simulated vs live portfolio.

------------------------------------------------------------------------

# Architecture

    Filesystem + Git (desired state + history)
            ↓
    Reconciliation Engine
            ↓
    Broker

------------------------------------------------------------------------

# Scope

TaFS is a **filesystem representation spec** — it defines how trading
intent and state map onto files and directories.

What belongs in the filesystem (this spec):

-   order intent (`orders/`)
-   position state (`pf/`)
-   account state (`cash.*`, `marginfree.*`, `equity.*`)
-   history (`git log`)

What belongs in the engine (implementation concern):

-   risk limits (max exposure, daily loss caps)
-   execution policies (TWAP, VWAP, slicing)
-   broker configuration and credentials
-   drift detection and error recovery

What can be built on top (higher-level abstractions):

-   target allocation (desired portfolio → engine generates orders)
-   watchlists and alerts
-   signal ingestion from external systems
-   AI-native controllers

------------------------------------------------------------------------

# Prior Art

TaFS combines ideas from systems that use filesystem structure as a
data model with the declarative reconciliation pattern from
infrastructure-as-code. No existing system combines all three; no
trading system uses this pattern.

| System | What TaFS borrows |
|---|---|
| **cgroupfs** | `mkdir` creates a resource, `rmdir` destroys it — directory existence *is* the operation |
| **Maildir** | one file = one message, filename = metadata, moving between directories = state transition |
| **Plan 9 / 9P** | every resource has a filesystem interface — the structure is the schema |
| **GitOps** | Git as single source of truth, continuous reconciliation toward desired state |
| **Terraform** | declarative desired-state with explicit diff and apply |

Where TaFS differs: Terraform and GitOps store data *in* file content
(YAML, HCL). TaFS stores data *in* the filesystem structure itself —
filenames, directories, presence, and absence. The filesystem is not
a container for configuration; it is the configuration.

------------------------------------------------------------------------

# Benefits

-   deterministic audit trail
-   simple mental model
-   OS-level reliability primitives
-   AI-agent friendly interface
-   reproducibility
-   declarative automation

------------------------------------------------------------------------

# Trade-offs

-   reconciliation complexity
-   drift detection required
-   compound orders need nesting
-   not suited for HFT

------------------------------------------------------------------------

# Future Directions

-   FUSE virtual trading filesystem
-   multi-broker reconciliation
-   AI-native controllers
-   target allocation layer (declare desired portfolio, engine converges)

------------------------------------------------------------------------

# Status

Early experimental concept.
