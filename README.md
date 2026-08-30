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

## What this tree is

IgH EtherCAT Master 1.5.2 with Gavin Lambert's unofficial patchset applied (see
[`README-lambert.md`](README-lambert.md)), plus local fixes — most notably the
distributed-clock `app_time_sent` correction. Without it the master computes the DC
system-time offset against a `jiffies`-corrected application time rather than the time
the read datagram actually went on the wire, and the reference slave starts roughly one
cycle out of lock.

Upstream's own README is unchanged, in [`README`](README).

## Why move to Vectioneer

Vectioneer's fork is the **same lineage** — 1.5.2 plus the same patchset — maintained
about four years further. Their `master/` still receives commits in 2026; the last
change here before this notice was `415e9477`, January 2022.

It is a practical migration, not a rewrite:

* **No application source change.** Same ioctl version magic (`36`), so an application
  built against this fork runs against Vectioneer once rebuilt. Checked against one real
  application: all 38 `ecrt_*` entry points it uses have identical signatures,
  `ec_slave_config_state_t` still carries `error_flag` / `ready` / `position`,
  `ecrt_sdo_request_read()` still returns `void`, and `ECRT_VERSION` still takes three
  arguments.
* **A superset, not a trade.** Diffing `master/` against this tree: 47 files changed,
  +3531 / −909, and every added file is purely additive. Nothing this fork depends on
  was dropped.
* **The fixes from here are there.** SII override and the DC `app_time_sent` correction
  are both present, the latter with its original authorship intact.

## Why not official IgH stable-1.6

Two things stable-1.6 does not have, both of which we need:

* **SII override.** Some vendors ship devices whose SII/EEPROM image is wrong. The
  workaround is to generate a correct image from the vendor's ESI/XML and have the
  master load it from a directory in preference to the device itself.
  `--enable-sii-override` does not exist in stable-1.6.
* **The DC `app_time_sent` correction**, and the patchset's mailbox-flush handling.

Forward-porting is also not incremental: 1.6 moved slave scanning and configuration into
`fsm_slave` across a large refactor, so the port is all-or-nothing rather than a series
of cherry-picks.

For balance — 1.6 does have one thing worth wanting that neither this fork nor
Vectioneer offers: the 2024 batch that changed several `ecrt_*` functions from `void` to
`int`, so an application can find out *why* an operation failed. It is an ABI break
against everything built on this line.

## What exists only here

* **Linux 5.4 device drivers**, required by the Ubuntu 18.04 + Xenomai 3 target.
  (5.15 sources for `8139too`, `e1000`, `e1000e` and `igc` were imported from upstream
  1.6.12 and are here as well.)
* `415e9477` — an `EC_EOE` guard in `master/rtdm_xenomai_v3.c`, needed to build
  `--disable-eoe` under Xenomai RTDM.

## License

Unchanged from upstream. See [`COPYING`](COPYING) and [`COPYING.LESSER`](COPYING.LESSER).
Use of the EtherCAT technology and brand remains subject to the industrial property
rights of Beckhoff Automation GmbH.
