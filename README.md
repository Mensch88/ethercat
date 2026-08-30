# IgH EtherCAT Master — Mensch88 fork

> **This fork is in maintenance mode.**
>
> It is kept for one target: the legacy **Ubuntu 18.04 + Linux 5.4 + Xenomai 3** stack.
> If you are not on that stack, use the **Vectioneer fork** instead:
>
> **<https://git.vectioneer.com/pub/etherlab>** — branch `stable/vectioneer`
>
> ```
> git clone -b stable/vectioneer https://git.vectioneer.com/pub/etherlab.git
> ```
>
> A full-history mirror of that fork, with credit and branch layout documented, is at
> [Mensch88/etherlab-vectioneer](https://github.com/Mensch88/etherlab-vectioneer) —
> use it if you want the tree on GitHub, but Vectioneer's server stays authoritative.

## What this tree is

IgH EtherCAT Master 1.5.2 with Gavin Lambert's unofficial patchset applied (see
[`README-lambert.md`](README-lambert.md)), plus local fixes. Upstream's own README is
unchanged, in [`README`](README).

### The mailbox and SDO work — the bulk of the patchset

Most of the patchset is not small fixes; it is a rework of how the master shares each
slave's mailbox between the state machines that all want to use it. In stock 1.5.2 the
CoE mailbox is contended between the master's own internal SDO traffic and the
application's SDO requests, and a read can consume a response that belonged to someone
else. The patchset addresses this on several levels:

* **Route responses by protocol type** so an FSM cannot swallow another protocol's reply
  (`base/0019`, `mbox_prot`).
* **A per-slave mailbox read lock** — `ec_read_mbox_locked()` / `ec_read_mbox_lock_clear()`,
  used at roughly 140 call sites — so only one read is ever in flight for a given slave.
* **Move internal SDO and SDO-dictionary handling out of `fsm_master` into `fsm_slave`**
  (`features/parallel-slave/0005`). This is the one that removes the fight over the CoE
  mailbox between internal and external SDO requests outright, rather than guarding it,
  and lets the work proceed in parallel across slaves.
* **Move `fsm_slave_scan` and `fsm_slave_config` there too** (`0006`), so scanning and
  configuration also parallelise.
* Supporting fixes: clear mailboxes after a re-scan (`0022`), don't abandon the mailbox
  FSMs early (`0027`), wait for the SDO dictionary before processing requests (`0021`),
  fetch the dictionary only on demand (`0024`), ignore mailbox settings from a blank
  EEPROM (`0025`).
* **SDO complete access** — `ecrt_master_sdo_upload_complete()`,
  `ecrt_slave_config_create_sdo_request_complete()`, `ecrt_sdo_request_index_complete()`,
  and `ecrt_sdo_request_write_with_size()` (`features/complete/*`).
* **A mailbox gateway server** (`features/mbg/*`).

### Local fixes on top

Chiefly the distributed-clock `app_time_sent` correction: without it the master computes
the DC system-time offset against a `jiffies`-corrected application time rather than the
time the read datagram actually went on the wire, and the reference slave starts roughly
one cycle out of lock.

That one is not local to this fork any more. It travels as
`base/0002-junyuan-dc_sync_issues.patch` in the patchset, was applied to the 1.5 line as
*Distributed Clock fixes from Jun Yuan*, and reached the official IgH tree as `17eddce6`
in June 2022. Two older patches from the same author are in upstream as well: an `e1000e`
ethtool fix correcting which operations are refused while EtherCAT owns the NIC
(`170110f7`, 2012), and a lifetime fix in `ecrt_master_sdo_download_complete()`, which had
been keeping an asynchronously-released, kref-counted SDO request on the stack
(`4a858fc9`, 2011).

## Why move to Vectioneer

Vectioneer's fork is the **same lineage** — 1.5.2 plus the same patchset — maintained
about four years further. Their `master/` still received commits in 2026; the last
change here before this notice was `415e9477`, January 2022.

It is a practical migration, not a rewrite:

* **No application source change.** Same ioctl version magic (`36`), so an application
  built against this fork runs against Vectioneer once rebuilt. Checked against one real
  application: all 38 `ecrt_*` entry points it uses have identical signatures,
  `ec_slave_config_state_t` still carries `error_flag` / `ready` / `position`,
  `ecrt_sdo_request_read()` still returns `void`, and `ECRT_VERSION` still takes three
  arguments.
* **A superset, not a trade.** Diffing `master/` against this tree: 47 files changed,
  +3531 / −909, and every added file is purely additive.
* **The whole mailbox/SDO stack above is present**, call site for call site, along with
  SII override and the DC correction — the latter with its original authorship intact.
* **The mailbox lock is better there.** This tree takes a per-slave `rt_mutex`
  (`mbox_sem`) to guard `read_mbox_busy`. Vectioneer drops the mutex and makes the flag
  itself atomic — `atomic_cmpxchg(&slave->read_mbox_busy, 0, 1)`, inlined in `slave.h`.
  Same semantics, nothing to block on.

## Why not official IgH stable-1.6

Upstream took *part* of the patchset, which is worth being precise about. stable-1.6
**does** have protocol-based mailbox routing and **does** have the `fsm_slave` refactor.
What it does not have:

| | this fork | Vectioneer | stable-1.6 |
|---|---|---|---|
| per-slave mailbox read lock (`ec_read_mbox_locked`) | yes | yes (lock-free) | **no** |
| SDO complete access API | yes | yes | **no** |
| mailbox gateway | yes | yes | **no** |
| SII override (`--enable-sii-override`) | yes | yes | **no** |
| DC `app_time_sent` correction | yes | yes | **no** |

The two that block us outright:

* **SII override.** Some vendors ship devices whose SII/EEPROM image is wrong. The
  workaround is to generate a correct image from the vendor's ESI/XML and have the master
  load it from a directory in preference to the device itself. There is no equivalent in
  stable-1.6.
* **The DC `app_time_sent` correction.**

Forward-porting is also not incremental: the `fsm_slave` refactor upstream landed as a
large restructuring, so the port is all-or-nothing rather than a series of cherry-picks.

For balance — 1.6 does have one thing worth wanting that neither this fork nor Vectioneer
offers: the 2024 batch that changed several `ecrt_*` functions from `void` to `int`, so an
application can find out *why* an operation failed. It is an ABI break against everything
built on this line.

## What exists only here

* **Linux 5.4 device drivers**, required by the Ubuntu 18.04 + Xenomai 3 target —
  `5b9c96b6` *add 8139too for kernel 5.4*, and `8e9efbe1` for r8169. Neither official
  `stable-1.6` nor Vectioneer has a 5.4 `8139too`; upstream went from 4.19 straight to
  5.10. (5.15 sources for `8139too`, `e1000`, `e1000e` and `igc` were imported from
  upstream 1.6.12 and are here as well.)
* `415e9477` — an `EC_EOE` guard in `master/rtdm_xenomai_v3.c`, needed to build
  `--disable-eoe` under Xenomai RTDM.

## License

Unchanged from upstream. See [`COPYING`](COPYING) and [`COPYING.LESSER`](COPYING.LESSER).
Use of the EtherCAT technology and brand remains subject to the industrial property
rights of Beckhoff Automation GmbH.
