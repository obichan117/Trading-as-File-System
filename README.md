# Trading as File System (TaFS)

**TaFS (Trading as File System)** is an experimental architecture that
maps trading intent, position state, and account state directly onto a
filesystem.

Instead of imperative API calls, TaFS treats trading as a **declarative
desired-state system**.

------------------------------------------------------------------------

## Quick Example

    /tradefs/rakuten
      .log/

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
    .log/   = history

## Idempotency

Repeated reconciliation produces identical results.

## Unix-native Semantics

-   create → submit
-   delete → cancel
-   rename → state transition
-   append → event stream

------------------------------------------------------------------------

# Filename Grammar

## Orders

    <code>.<signed_lot>[.<price>][.<mode>]

-   signed lot represents target exposure
-   `p` used for decimal prices
-   mode (margin): `o` open, `c` close

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

## Stop

    7203.1.stp_2800

## OCO

    orders/OCO_7203/
      a.7203.-1.lmt_2900
      b.7203.-1.stp_2800

## IFDOCO

    orders/IFDOCO_7203/
      1.7203.1.lmt_2858
      2a.7203.-1.lmt_2900
      2b.7203.-1.stp_2800

------------------------------------------------------------------------

# Architecture

    Filesystem (desired state)
            ↓
    Reconciliation Engine
            ↓
    Broker

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
-   complex conditional orders need grouping
-   not suited for HFT

------------------------------------------------------------------------

# Future Directions

-   FUSE trading filesystem
-   multi-broker reconciliation
-   Git time-travel state
-   AI-native controllers

------------------------------------------------------------------------

# Status

Early experimental concept.
