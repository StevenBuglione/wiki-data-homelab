---
title: "UPS Sizing and Graceful Shutdown Testing for a Self-Hosted Rack"
description: "Practical reference for estimating UPS runtime, mapping critical loads, integrating UPS monitoring, and validating graceful shutdown and restart behavior for homelab servers, NAS devices, and network gear."
tags:
  - "research"
  - "homelab"
area: general
status: active
difficulty: intermediate
review_status: needs_review
generated_by: omg-wiki-research
human_reviewed: false
last_verified: 2026-05-25
confidence: medium
sources:
  - title: "www.se.com"
    url: "https://www.se.com/us/en/work/products/tools/ups-selector/"
    accessed: 2026-05-25
  - title: "www.eaton.com"
    url: "https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html"
    accessed: 2026-05-25
  - title: "www.cyberpowersystems.com"
    url: "https://www.cyberpowersystems.com/tools/runtimes/"
    accessed: 2026-05-25
  - title: "networkupstools.org"
    url: "https://networkupstools.org/docs/user-manual.chunked/ar01s06.html"
    accessed: 2026-05-25
  - title: "networkupstools.org"
    url: "https://networkupstools.org/docs/man/upsmon.html"
    accessed: 2026-05-25
  - title: "networkupstools.org"
    url: "https://networkupstools.org/docs/man/upssched.html"
    accessed: 2026-05-25
  - title: "www.truenas.com"
    url: "https://www.truenas.com/docs/scale/systemsettings/services/upsservices/"
    accessed: 2026-05-25
  - title: "www.truenas.com"
    url: "https://www.truenas.com/docs/scale/systemsettings/services/upsservicesscreen/"
    accessed: 2026-05-25
  - title: "datatracker.ietf.org"
    url: "https://datatracker.ietf.org/doc/html/rfc1628"
    accessed: 2026-05-25
  - title: "www.cyberpowersystems.com"
    url: "https://www.cyberpowersystems.com/products/software/power-panel-business-software/power-panel-business/"
    accessed: 2026-05-25
  - title: "www.shopulstandards.com"
    url: "https://www.shopulstandards.com/ProductDetail.aspx?productId=UL1778"
    accessed: 2026-05-25
  - title: "www.energystar.gov"
    url: "https://www.energystar.gov/products/uninterruptible_power_supplies"
    accessed: 2026-05-25
  - title: "csrc.nist.gov"
    url: "https://csrc.nist.gov/pubs/sp/800/34/r1/final"
    accessed: 2026-05-25
  - title: "docs.docker.com"
    url: "https://docs.docker.com/engine/storage/drivers/overlayfs-driver/"
    accessed: 2026-05-25
  - title: "docs.docker.com"
    url: "https://docs.docker.com/engine/storage/volumes/"
    accessed: 2026-05-25
---

# UPS Sizing and Graceful Shutdown Testing for a Self-Hosted Rack

## Summary

A UPS plan for a self-hosted rack is a load plan, a runtime plan, and a recovery plan. The UPS should be selected from the protected load in watts and the runtime target, not from a vague rack size or the largest VA number on sale. Vendor selectors and runtime calculators use estimated or entered load, model characteristics, and battery configurations to estimate usable runtime, which is why the first task is an inventory of actual devices and expected draw [1][2][3]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

The practical goal is rarely to run the rack indefinitely. For a homelab, the UPS should absorb brief utility interruptions, keep the network path alive long enough for notifications and shutdown signals, allow NAS and server workloads to flush data, and leave enough reserve to avoid a hard battery cutoff. Contingency planning is stronger when each service has a stated RPO and RTO before the drill, because a power test should prove whether the rack behavior meets service priorities rather than only proving that a battery turns on [12][13]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

- Assumptions: the rack uses certified plug-in UPS equipment installed according to the manufacturer instructions, no hardwired electrical work is covered, the shutdown controller and management network can remain on protected power, and command examples use placeholders that must be replaced with the site’s real UPS name and hostnames [4][11]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Inventory the load as modem/router, core switch, UPS monitor, NAS, virtualization hosts, application hosts, and noncritical convenience devices. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Set a runtime target such as 10 minutes for orderly shutdown, 30 minutes for network-only continuity, or a longer target only when the battery and load data support it. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Treat the first controlled drill as the acceptance test: on-battery detection, shutdown signal, service stop, NAS safe mode, UPS output behavior, power return, boot, and application health must all be visible in logs. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

## Decision Matrix

The first decision is what stays on battery. Network edge gear and the UPS monitor usually earn the highest priority because the shutdown path often depends on them. NAS devices and hypervisors are next because they hold state and need a graceful stop. Lab workstations, PoE cameras, build nodes, and convenience devices should be justified explicitly or left off the UPS so they do not steal runtime from storage and network control paths [1][12]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

The second decision is how the rack learns about the UPS. Vendor tools can be appropriate when the rack is mostly one vendor ecosystem; CyberPower PowerPanel Business, for example, documents monitoring, alerts, event logs, battery tests, and command execution for protected systems. NUT is the best general cross-vendor pattern when the operator wants one primary UPS-connected host and multiple secondary clients. TrueNAS can participate directly because its UPS service is NUT-based and exposes master/slave behavior and shutdown controls [4][7][8][10]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

- | Decision | Good default | When to choose differently | Evidence | [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- | Runtime target | Size for a controlled shutdown plus a small reserve | Choose longer runtime only when network uptime is the explicit objective and runtime curves support the load | [1][2][3] | [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- | Monitor role | One primary/master host connected by USB or SNMP, with secondary clients following it | Use appliance-managed UPS only when the appliance can reliably notify every dependent host | [4][5][7][8] | [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- | Shutdown trigger | Low-battery plus an earlier on-battery timer for conservative racks | Use low-battery only when runtime is long, batteries are healthy, and the drill proves sufficient shutdown time | [4][6] | [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- | Container hosts | Stop applications in dependency order and confirm persistent volumes | Add custom stop hooks only when the application cannot survive normal service shutdown | [14][15] | [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

## Reference Architecture

A conservative homelab design uses one UPS control path. The UPS connects to a primary monitor by USB or network management, the primary publishes UPS state to secondary clients, and each protected server executes its own graceful shutdown command when the primary declares forced shutdown or the configured condition is met. NUT documents primary and secondary monitor roles, the forced-shutdown condition, and the shutdown command behavior that turns UPS status into operating-system action [4][5]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

The protected topology should not depend on equipment that loses power early. Put the UPS monitor, router, core switch, and any NAS or host that must receive shutdown messages on protected power. RFC 1628 provides the standardized SNMP language for battery state, estimated minutes remaining, and load, while TrueNAS documents a NUT-based service model for NAS participation. That makes the architecture portable: USB-attached UPS for a small rack, SNMP-managed UPS for a larger rack, and secondary clients for everything that must shut down before battery exhaustion [7][8][9]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

- UPS: line-interactive or online unit selected from actual load and runtime target; do not publish model-specific runtime promises unless they come from that model’s official runtime data. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Primary monitor: NUT primary, vendor monitoring host, or NAS master with stable power, local logs, and notification reachability. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Secondary clients: hypervisors, Docker hosts, NAS peers, and application servers that subscribe to the primary and execute local shutdown commands. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Network path: modem, router, switch, and management DNS/notification dependencies remain on the highest-priority load tier if shutdown messages cross the network. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Evidence path: logs from UPS software, NAS service, systemd or init, container orchestrator, and application health checks are retained after the drill. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

## Restore Runbook

Graceful shutdown testing is a restore drill, not just a battery drill. The maintenance window should begin with a current backup posture, a list of protected loads, a chosen trigger, and a rollback plan. NUT exposes status values such as line state, battery charge, estimated runtime, UPS load, nominal power, and delay timers; those values make the test observable before any shutdown command is issued [4]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

A safe baseline sequence is: verify the UPS and clients are communicating; notify users; stop or drain noisy workloads; trigger the documented shutdown path; observe secondary clients stopping; confirm NAS safe mode or shutdown; confirm UPS output behavior only if the design requires it; restore utility power; and verify boot order and service health. Forced shutdown with `upsmon -c fsd` is appropriate only in a maintenance window because it drives the real shutdown sequence. For early shutdown, `upssched` can start a timer on an on-battery event and cancel it if utility power returns before the timer expires [4][5][6][8]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

- Status check template: run `upsc <ups_name>@localhost` for the full driver-reported variable list or `upsc <ups_name>@localhost ups.status` for a single status value, then record `battery.charge`, `battery.runtime`, and `ups.load` if the driver exposes them [4]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Monitor role template: `MONITOR <ups_name>@<primary_host> 1 <monitor_user> <secret> primary` on the primary and `MONITOR <ups_name>@<primary_host> 1 <monitor_user> <secret> secondary` on clients, with local secrets and hostnames adjusted for the rack [5]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Shutdown command template: `SHUTDOWNCMD "/sbin/shutdown -h +0"` or the distribution-supported equivalent, recorded in configuration management rather than typed during an outage [5]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Early-timer template: `AT ONBATT * START-TIMER rack_shutdown 300`, `AT ONLINE * CANCEL-TIMER rack_shutdown`, and a command script case that calls `upsmon -c fsd` for `rack_shutdown` after the timer expires [6]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Driver-path dry run: use NUT’s documented `upsdrvctl -t shutdown` test path before any live output-off drill, and do not run a real UPS load-off test until the protected systems have already demonstrated clean shutdown [4]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Evidence to capture: utility-loss time, ONBATT time, timer start, FSD or low-battery time, each host shutdown time, NAS safe-mode or shutdown time, UPS output-off time, utility-return time, boot time, and application health-check time. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

## Failure Scenarios

Power-event failures are usually coordination failures. The UPS may be healthy, but a secondary client may never receive the forced-shutdown state, a NAS may still be flushing writes, or the monitor host may depend on an unprotected switch. NUT documents communication-loss and parent-process states, primary and secondary behavior, and forced shutdown; those states should appear in the failure-mode checklist instead of being treated as surprises [4][5]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

The restart side deserves equal attention. A rack that shuts down cleanly but returns in the wrong order can still break services. Storage, DNS, identity, databases, container volumes, and application dependencies need a simple boot sequence and health checks. Docker documents that volumes are the preferred persistence boundary and that the container writable layer is destroyed with the container, so a power drill should confirm that stateful applications are using durable storage before the outage is simulated [14][15]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

- UPS communication loss: unplugging USB or breaking SNMP during a non-shutdown test should produce a visible NOCOMM-style alert without immediately creating a surprise rack shutdown [5]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Battery depletes earlier than expected: the early timer and low-battery trigger should still leave enough time for the slowest NAS or hypervisor shutdown [4][6][8]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Primary monitor unavailable: a secondary client plan is incomplete if the only host that can issue shutdown commands is itself already down. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Network path unavailable: switch, router, DNS, or VLAN dependencies that carry shutdown messages need protected power or a local fallback. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- NAS not ready: storage clients should stop before the NAS enters safe mode, and the NAS should not be the last device to learn that the rack is shutting down [7][8]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Containers lose state: any service writing important data into the container writable layer instead of a volume must be fixed before the live drill [15]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)
- Power returns mid-shutdown: the runbook should allow shutdown to finish and then validate boot order rather than racing to restart every service immediately [4]. [4](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html) [5](https://networkupstools.org/docs/man/upsmon.html)

## Operational Checklist

The checklist should be run before buying the UPS, after installing monitoring, after changing major loads, and after replacing batteries. The load inventory should record device name, power source, measured or estimated watts, role in shutdown, service owner, RPO, RTO, and restart dependency. Vendor tools can estimate runtime once the protected load is known, but the rack’s observed drill remains the acceptance evidence [1][2][3][13]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

The drill should be boring. Every actor should know what happens before the power event: how alerts arrive, which hosts shut down first, how long the NAS needs, which applications are health-checked, and what threshold aborts the exercise. CyberPower’s monitoring documentation and TrueNAS’s UPS service references are useful examples of the operational surfaces to include: dashboard status, recent events, UPS information, battery tests, shutdown mode, shutdown timer, shutdown command, and power-off behavior [8][10]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

- Preflight: confirm backups, UPS battery status, load below planned threshold, monitor credentials, notification path, current system updates, and maintenance-window approval. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Inventory: record actual draw where measured, otherwise use conservative estimates and keep nameplate ratings as upper-bound notes rather than runtime predictions [1]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Configuration: verify NUT or vendor software starts on boot, monitor roles match the design, config files are not world-readable, and remote monitor credentials are not defaults [4][5][8]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Application stop order: stop write-heavy apps, databases, containers, hypervisors, and storage clients before NAS safe mode or shutdown. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Storage check: verify Docker volumes or bind mounts for stateful data, `docker info` storage driver details where relevant, and no manual edits inside `/var/lib/docker` [14][15]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Postflight: compare observed shutdown and restart times to RPO/RTO, capture logs, update runtime assumptions, and create a follow-up issue for any missed dependency. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

## Common Pitfalls

The common sizing mistake is treating the VA rating as a runtime promise. Runtime is a function of the protected watts, battery condition, UPS model, and add-on battery configuration, so the article should not publish universal minute estimates. Schneider Electric, Eaton, and CyberPower all steer users toward load-aware selection or runtime data rather than a one-size-fits-all rule [1][2][3]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

The common software mistake is assuming the shutdown command is harmless because it is text in a configuration file. `upsmon -c fsd` is a live control action, monitor credentials can authorize shutdown behavior, and TrueNAS warns about unsupported auxiliary parameters. Docker adds a separate trap: stateful application data should not live only in a container writable layer when volumes are the supported persistence boundary [4][5][8][15]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

- Do not put every rack device on the UPS just because there are open outlets; noncritical loads can erase the runtime margin needed by storage and network control paths. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Do not use unsafe wiring advice, battery modification advice, or outlet-strip daisy-chain guidance in the page; keep safety guidance at manufacturer instructions and certified equipment boundaries [11]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Do not depend on cloud-only notifications during an outage unless the modem, router, DNS path, and switch remain powered long enough to send them. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Do not assume USB device naming remains stable after reboot; document how the chosen UPS driver identifies the device and test after restarts [4][7]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Do not skip restart validation; a clean shutdown followed by a broken boot order is still a failed recovery drill. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Do not change Docker storage drivers casually; Docker documents that changing the storage driver can make existing local containers and images inaccessible without prior save or migration [14]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

## Maintenance Notes

UPS maintenance is not only replacing batteries. The operational record should track load changes, event logs, battery tests, configuration changes, and the last successful shutdown-and-restart drill. Vendor monitoring software can expose event logs and battery tests, NUT can expose runtime and load state, and TrueNAS can expose NAS-specific UPS shutdown options; those surfaces should be reviewed when the rack changes [4][8][10]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

A useful cadence is lightweight but consistent: monthly status review, quarterly notification and monitor-role test, semiannual tabletop or timer-only review, and an annual controlled shutdown drill or a drill after any battery replacement, UPS replacement, NAS migration, hypervisor replacement, or major load increase. NIST contingency-planning guidance supports testing and evaluation against operational priorities, so each maintenance cycle should end with the same question: did the observed behavior still meet the service RPO and RTO [13]? [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

- Monthly: record UPS load, battery charge, estimated runtime, alerts, and event-log changes. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- Quarterly: verify notification recipients, monitor credentials, primary/secondary roles, and that each protected host still runs its local shutdown agent. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- After battery replacement: repeat runtime estimation and at least one controlled shutdown-path test before trusting the new battery set. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- After load changes: rerun the vendor runtime calculator or runtime graph comparison and update the protected-load inventory [1][2][3]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- After platform changes: recheck Docker storage driver, persistent volumes, NAS UPS service settings, and application boot order [8][14][15]. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
- After every drill: preserve logs, observed times, deviations, and the next corrective action in the wiki page or linked runbook. [1](https://www.se.com/us/en/work/products/tools/ups-selector/) [2](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)

## Sources

1. [www.se.com](https://www.se.com/us/en/work/products/tools/ups-selector/)
2. [www.eaton.com](https://www.eaton.com/us/en-us/products/backup-power-ups-surge-it-power-distribution/backup-power-ups/ups-runtime-graphs.html)
3. [www.cyberpowersystems.com](https://www.cyberpowersystems.com/tools/runtimes/)
4. [networkupstools.org](https://networkupstools.org/docs/user-manual.chunked/ar01s06.html)
5. [networkupstools.org](https://networkupstools.org/docs/man/upsmon.html)
6. [networkupstools.org](https://networkupstools.org/docs/man/upssched.html)
7. [www.truenas.com](https://www.truenas.com/docs/scale/systemsettings/services/upsservices/)
8. [www.truenas.com](https://www.truenas.com/docs/scale/systemsettings/services/upsservicesscreen/)
9. [datatracker.ietf.org](https://datatracker.ietf.org/doc/html/rfc1628)
10. [www.cyberpowersystems.com](https://www.cyberpowersystems.com/products/software/power-panel-business-software/power-panel-business/)
11. [www.shopulstandards.com](https://www.shopulstandards.com/ProductDetail.aspx?productId=UL1778)
12. [www.energystar.gov](https://www.energystar.gov/products/uninterruptible_power_supplies)
13. [csrc.nist.gov](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
14. [docs.docker.com](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)
15. [docs.docker.com](https://docs.docker.com/engine/storage/volumes/)
