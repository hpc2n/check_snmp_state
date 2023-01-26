# check_snmp_state
SNMP walk-capable monitoring plugin for Icinga/Nagios

# Motivation

There are a lot of devices out there that only provides detailed status of
the internal hardware. While from the same vendor, the actual configuration
such as number of CPUs or memory modules might differ which causes
headaches when trying to set up snmp monitoring with decent coverage for
those systems.

While the original `snmp_check` plugin does a decent job for checking
thresholds, it's found lacking when monitoring discrete states due to:

- No possibility to report WARNING state.
- You must specify a static set of OIDs (items) to check.

This utility adds the possibility to define WARNING states, and also
leverages `snmpwalk` / `snmpbulkwalk` to check the status of an entire
subset of components. To make it easier it also supports specifying the MIB
directory, eliminating the need to use the `env MIBDIR=/path/to/mibs
check_snmp ...` hack.

# Performance

The original `snmp_check` plugin is a C program executing the standard
`snmpget` utility. This plugin also executes the standard SNMP utilities,
and while written in Perl the performance impact is negligible compared to
the original `snmp_check`.

## Performance tips

A common misconception is that all SNMP MIB files should be placed in a
single directory. While this works for smaller setups, when adding MIBs for
a wider range of devices you will find that there are conflicts and also
that simply parsing all MIB files becomes CPU intensive on the monitoring
host.

A better strategy is to collect the needed MIBs in separate directories for
each class of devices.

# Requirements

- Perl
  - Including `Monitoring::Plugin`
- Standard Net-SNMP tools for `snmpget` et al.
- Recommended: MIB files for your devices
  - While using numeric OIDs/values works, the user experience is much
    better when using MIBs.

# License

[GNU Affero General Public License v3.0 (AGPLv3)](LICENSE)

For more information, see:

https://www.gnu.org/licenses/agpl-3.0.html

https://choosealicense.com/licenses/agpl-3.0/

# Limitations

This utility is not intended for checking thresholds such as fan speeds
etc, use the original `check_snmp` plugin for that.

# Usage

While the arguments are heavily inspired by the original `check_snmp`
plugin, the behavior might differ.

See the help as shown by `./check_snmp_state -h` for the complete set of
arguments.

To debug issues, it's recommended to run manually using the `-vv` option
to enable verbose/debug output.

## Examples

### Check memory device health on a Lenovo XClarity equipped server

Check all memoryHealthStatus items, this uses `snmpbulkwalk`.

We expect `Normal` to signify an `OK` state. Everything else will get an
`CRITICAL` status with the extra tag `UNDEFINED` in the output.

Since newer XClarity firmwares only supports SNMP v3, that's what we use.

```
$ ./check_snmp_state -M /path/to/mibs/lenovo/xcc -P 3 -U snmpuser \
  -H hostname-xcc -O Normal -W -o LENOVO-XCC-MIB::memoryHealthStatus
SNMP_STATE OK - 6 OK
OK: LENOVO-XCC-MIB::memoryHealthStatus.1 Normal
OK: LENOVO-XCC-MIB::memoryHealthStatus.2 Normal
OK: LENOVO-XCC-MIB::memoryHealthStatus.3 Normal
OK: LENOVO-XCC-MIB::memoryHealthStatus.4 Normal
OK: LENOVO-XCC-MIB::memoryHealthStatus.5 Normal
OK: LENOVO-XCC-MIB::memoryHealthStatus.6 Normal
```

### Check memory device health on a Dell iDRAC equipped server

Check all memoryDeviceStatus items. This uses `snmpbulkwalk`. Here we leverage that the MIB file
enumerates the status, so we can actually tell the plugin what to expect as
ok/warning/critical states.

```
$ ./check_snmp_state -M /path/to/mibs/idrac -H hostname-idrac \
   -O ok -w nonCritical -c critical -c nonRecoverable \
   -W -o IDRAC-MIB-SMIv2::memoryDeviceStatus
SNMP_STATE OK - 16 OK
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.1 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.2 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.3 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.4 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.5 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.6 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.7 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.8 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.9 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.10 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.11 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.12 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.13 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.14 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.15 ok
OK: IDRAC-MIB-SMIv2::memoryDeviceStatus.1.16 ok
```

### Check global and combined status items on a Dell iDRAC equipped server

Check all global/combined/rollup/summary states explicitly, this uses
`snmpget` on the specific OIDs.

Here we leverage that the MIB file enumerates the status, so we can
actually tell the plugin what to expect as ok/warning/critical states.

```
$ ./check_snmp_state -M /path/to/mibs/idrac -H hostname-idrac \
   -O ok -w nonCritical -c critical -c nonRecoverable \
   -o IDRAC-MIB-SMIv2::globalSystemStatus.0 -o IDRAC-MIB-SMIv2::globalStorageStatus.0 \
   -o IDRAC-MIB-SMIv2::systemStateGlobalSystemStatus.1 -o IDRAC-MIB-SMIv2::systemStatePowerSupplyStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStateVoltageStatusCombined.1 -o IDRAC-MIB-SMIv2::systemStateAmperageStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStateCoolingDeviceStatusCombined.1 -o IDRAC-MIB-SMIv2::systemStateTemperatureStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStateMemoryDeviceStatusCombined.1 -o IDRAC-MIB-SMIv2::systemStateChassisIntrusionStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStatePowerUnitStatusCombined.1 -o IDRAC-MIB-SMIv2::systemStateCoolingUnitStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStateProcessorDeviceStatusCombined.1 -o IDRAC-MIB-SMIv2::systemStateBatteryStatusCombined.1 \
   -o IDRAC-MIB-SMIv2::systemStateTemperatureStatisticsStatusCombined.1
SNMP_STATE OK - 15 OK
OK: IDRAC-MIB-SMIv2::globalSystemStatus.0 ok
OK: IDRAC-MIB-SMIv2::globalStorageStatus.0 ok
OK: IDRAC-MIB-SMIv2::systemStateGlobalSystemStatus.1 ok
OK: IDRAC-MIB-SMIv2::systemStatePowerSupplyStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateVoltageStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateAmperageStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateCoolingDeviceStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateTemperatureStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateMemoryDeviceStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateChassisIntrusionStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStatePowerUnitStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateCoolingUnitStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateProcessorDeviceStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateBatteryStatusCombined.1 ok
OK: IDRAC-MIB-SMIv2::systemStateTemperatureStatisticsStatusCombined.1 ok
```
