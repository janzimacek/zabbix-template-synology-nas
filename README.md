# Zabbix Template: Synology NAS Health by SNMP

A Zabbix 8.0 template for monitoring Synology NAS hardware, disks, RAID/volume state, capacity and temperatures through SNMP.

The template uses Synology enterprise OIDs under `1.3.6.1.4.1.6574` and standard HOST-RESOURCES-MIB uptime. It was tested with:

- Synology DS418
- DSM 7.4
- four SATA disks
- one volume at Synology volume table index `0`
- Zabbix 8.0
- SNMPv2c

> This is a fixed-layout template for four disks and one volume. Other Synology models may expose different numbers of disks, volumes or fan sensors. Review the OIDs before using the template on a different layout.

## Features

### NAS system health

- ICMP availability
- hardware uptime
- overall Synology system status
- system temperature
- power status
- fan 1 and fan 2 status
- model
- DSM version

### Four disks

For Drive 1 through Drive 4:

- model
- Synology disk status
- Synology disk health
- temperature
- bad-sector count
- identify-failure count

### Volume and RAID

- volume name
- RAID/volume status
- total capacity
- free capacity
- calculated used capacity
- calculated utilization percentage

The RAID/volume value map includes:

| Value | State |
|---:|---|
| 1 | Normal |
| 2 | Repairing |
| 3 | Migrating |
| 4 | Expanding |
| 5 | Deleting |
| 6 | Creating |
| 7 | RAID syncing |
| 8 | Parity checking |
| 9 | Assembling |
| 10 | Canceling |
| 11 | Degraded |
| 12 | Crashed |

### Triggers

The template reports:

- NAS unreachable by ICMP
- no SNMP data for 15 minutes
- system status not normal
- power failure
- fan failure
- high and critical system temperature
- disk status not normal
- disk health warning
- high disk temperature
- bad sectors
- identify failures
- volume in a non-normal operational state
- volume degraded or crashed
- volume utilization above 80 percent
- volume utilization above 90 percent

Capacity thresholds:

- above 80%: Warning
- above 90%: High

Temperature thresholds:

- disk above 50 C for 10 minutes: Warning
- system above 60 C for 10 minutes: Warning
- system above 70 C for 10 minutes: High

### Graphs

- system and disk temperatures
- volume utilization
- used and free volume capacity

### Dashboard

The included **Synology NAS overview** dashboard contains:

- NAS status
- model and DSM version
- system temperature
- volume utilization
- RAID / Volume 1 status
- health of Drive 1 through Drive 4
- temperature graph
- volume utilization graph
- capacity graph
- active NAS problems

Dashboard health colors:

- green: normal
- orange: warning, repair or maintenance state
- red: critical, failing, degraded or crashed

## Requirements

- Zabbix Server and Frontend 8.0
- SNMP enabled in Synology DSM
- UDP/161 reachable from the Zabbix server or proxy
- ICMP reachable if the ICMP item and trigger are used
- a Synology NAS exposing enterprise OIDs below `1.3.6.1.4.1.6574`

## Enable SNMP on Synology DSM

Open:

```text
Control Panel -> Terminal & SNMP -> SNMP
```

Enable SNMPv2c or SNMPv3. For SNMPv2c, use a dedicated read-only community instead of `public` and restrict UDP/161 to the Zabbix server or proxy with the DSM firewall.

Example read-only community name:

```text
CHANGE_ME_READ_ONLY
```

Do not place the actual community in the template YAML or in a public repository.

## Verify SNMP before importing

Replace `COMMUNITY` and `NAS_ADDRESS` with local values:

```bash
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.2.1.1
```

Check the Synology enterprise branch:

```bash
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.4.1.6574
```

Check true hardware uptime:

```bash
snmpget -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.2.1.25.1.1.0
```

For the tested four-disk layout, verify these tables:

```bash
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.4.1.6574.2.1.1
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.4.1.6574.3.1.1
```

Expected disk indexes are `0`, `1`, `2` and `3`. The expected volume index is `0`.

## Import the template

In Zabbix:

```text
Data collection -> Templates -> Import
```

Import:

```text
template_synology_nas_health_snmp.yaml
```

The template name is:

```text
Synology NAS health by SNMP
```

## Create the host

Example host configuration:

```text
Host name: NAS-SYNOLOGY
Visible name: Synology NAS
Host groups: Storage, Servers
```

Add an SNMP interface:

```text
Type: SNMP
IP address: NAS management address
Port: 161
SNMP version: SNMPv2
Community: {$SNMP_COMMUNITY}
```

Create the community as a host-level macro:

```text
Macro: {$SNMP_COMMUNITY}
Type: Secret text
Value: local read-only community
```

Do not add the community to the template itself.

Link the template:

```text
Synology NAS health by SNMP
```

Optionally link Zabbix's official `Interfaces by SNMP` template for network-interface discovery and traffic monitoring. This Synology template does not monitor network throughput.

## Validate collected data

After linking the template, open:

```text
Monitoring -> Latest data
```

Verify at minimum:

- `Synology: ICMP availability`
- `Synology: Hardware uptime`
- `Synology: System status`
- `Synology: System temperature`
- `Synology: Drive 1 health` through `Drive 4 health`
- `Synology: Volume 1 status`
- `Synology: Volume 1 free space`
- `Synology: Volume 1 total space`
- `Synology: Volume 1 utilization`

The calculated used-space and utilization items need valid `free` and `total` values first. A temporary `not enough data` message immediately after linking the template is expected until both source items have been collected.

## Security and privacy

The published YAML contains no:

- IP address
- SNMP community
- hostname from the monitored environment
- Synology serial number
- disk serial numbers
- email address
- MAC address
- personal name
- site or location name

Keep credentials and local addresses at host level. Before publishing future exports, search the YAML for private RFC1918 addresses, community strings, serial numbers, hostnames and email addresses.

## Known limitations

- The template is not low-level-discovery based. It is fixed to four disks and one volume.
- Fan 2 may be unsupported on Synology models with one fan. Disable the item and trigger if the OID is absent.
- Disk indexes and volume indexes may differ on another model or after major storage-layout changes.
- Synology MIB values can change between DSM versions.
- `Volume 1 status` represents the Synology RAID/volume state exposed by the MIB. It does not replace backups or DSM Storage Manager checks.
- The template does not monitor packages, Hyper Backup jobs, snapshots, UPS state or network throughput.

## Troubleshooting

### SNMP items are unsupported

Verify:

- SNMP is enabled in DSM
- the host has an SNMP interface
- UDP/161 is reachable
- the interface community references the correct host-level secret macro
- the Zabbix server or proxy is permitted by DSM Firewall

### Disk items are unsupported

Run:

```bash
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.4.1.6574.2.1.1
```

Confirm that disk indexes `0` to `3` exist.

### Volume items are unsupported

Run:

```bash
snmpwalk -v2c -c COMMUNITY NAS_ADDRESS 1.3.6.1.4.1.6574.3.1.1
```

Confirm that the volume row uses index `0`.

### Calculated utilization says not enough data

Collect these source items first:

```text
Synology: Volume 1 free space
Synology: Volume 1 total space
```

Then execute the calculated used-space and utilization items or wait for the next interval.

## Versioning

Recommended first public release:

```text
v1.0.0
```

Use semantic versioning:

- patch release for trigger, dashboard or documentation fixes
- minor release for backward-compatible metrics
- major release for item-key, macro or layout changes

## License

MIT is a practical license for a public Zabbix template. Add a `LICENSE` file to the repository if MIT is selected.
