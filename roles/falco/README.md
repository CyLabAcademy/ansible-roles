# falco

## Description

Installs [Falco](https://falco.org) to watch a host's syscalls and record anything matching its
rules. **It records and does nothing else.** It kills, blocks, and notifies nothing, and alerts
never leave the host except by whatever ships its log file. That shipping is someone else's job:
on the cork workers it is the [cloudwatch_agent](../cloudwatch_agent/README.md) role.

It is written for hosts that run untrusted code on a shared kernel, where the question is whether
something got out of a container, and it should run after whatever installs the container
runtime. Falco's container plugin reads `/var/run/docker.sock` for each container's name and
image.

What it sets up:

- **Falco from the falcosecurity apt repository**, pinned to `falco_version` and held, so an
  `apt upgrade` cannot move it. The pin also covers the rules, since the stable ruleset ships in
  the same package.
- **The modern eBPF driver** (`falco-modern-bpf.service`). It is compiled into the binary, so it
  needs no kernel headers, DKMS or download. The kmod units and `falcoctl-artifact-follow` are
  masked. The follower would otherwise replace the rules from the internet on its own schedule.
- **Alerts as JSON, one per line, in `falco_log_path`, and nowhere else.** The stdout, syslog,
  HTTP, program and gRPC outputs are off, and so is the web server. Falco's own lifecycle messages
  (start, rules loaded, errors) still go to the journal through stderr.
- **Metrics snapshots in `falco_metrics_path`.** These include the syscall drop counters. A flood
  can make Falco drop events, and its drop *alert* only fires at priority `debug`, so these
  counters are what show it went blind. A host whose snapshots stop has lost its Falco.
- **Local rules from `falco_rules_src`**, installed into `/etc/falco/rules.d/`. Each file is
  validated against the stable ruleset before it is put in place.

The package's `falco.yaml` and `falco_rules.yaml` are left untouched. Configuration goes in a
drop-in under `/etc/falco/config.d/`.

To see what fired:

```shell
$ sudo tail -f /var/log/falco/events.json | jq -c '{time, priority, rule, output}'
$ sudo journalctl -u falco-modern-bpf     # Falco itself: driver, rules loaded, errors
```

## Role Variables

| Name | Description | Default |
| --- | --- | --- |
| `falco_version` | Falco package version to install and hold | `0.45.2` |
| `falco_rules_src` | Local rules files (controller paths) to install into `/etc/falco/rules.d/` | `[]` |
| `falco_priority` | Lowest rule priority written out | `notice` |
| `falco_log_path` | The alert file, JSON lines | `/var/log/falco/events.json` |
| `falco_metrics_path` | The metrics snapshot file; empty disables metrics | `/var/log/falco/metrics.json` |
| `falco_metrics_interval` | How often a metrics snapshot is written | `15m` |
| `falco_log_rotate_days` | Daily rotations of either file kept on the host | `7` |
