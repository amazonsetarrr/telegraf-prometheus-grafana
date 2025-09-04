# Cisco NXOS SNMP Monitoring Guide

**Cisco NXOS systems provide extensive SNMP monitoring capabilities through both standard SNMP MIBs and Cisco-specific extensions.** This comprehensive guide covers all essential OIDs for monitoring CPU utilization, memory usage, disk usage, temperature sensors, and fan status, with complete implementation details and NXOS-specific considerations. The monitoring framework uses ENTITY-MIB as the foundation, with Cisco proprietary MIBs providing enhanced capabilities through the common entPhysicalIndex linking mechanism.

NXOS supports comprehensive system monitoring through a layered approach combining standard IETF MIBs with Cisco enterprise extensions. All Cisco MIBs use the base enterprise OID `1.3.6.1.4.1.9` and integrate seamlessly with the physical entity framework defined in ENTITY-MIB. This architecture enables correlation across multiple MIBs using the entPhysicalIndex as the primary linking key, supporting complex chassis architectures in Nexus 5000, 7000, and 9000 series platforms.

## CPU utilization monitoring

### Primary CPU monitoring OIDs

**CISCO-PROCESS-MIB** provides the most comprehensive CPU monitoring for NXOS systems with historical averages and multi-CPU support.

**Core CPU utilization objects:**
- **5-second average**: `1.3.6.1.4.1.9.9.109.1.1.1.1.6.1` (cpmCPUTotal5secRev)
  - Data Type: Gauge32 (0-100 percent)
  - Description: Overall CPU busy percentage in the last 5 seconds
  - Access: read-only

- **1-minute average**: `1.3.6.1.4.1.9.9.109.1.1.1.1.7.1` (cpmCPUTotal1minRev)
  - Data Type: Gauge32 (0-100 percent)
  - Description: Overall CPU busy percentage in the last 1 minute
  - Access: read-only

- **5-minute average**: `1.3.6.1.4.1.9.9.109.1.1.1.1.8.1` (cpmCPUTotal5minRev)
  - Data Type: Gauge32 (0-100 percent)
  - Description: Overall CPU busy percentage in the last 5 minutes
  - Access: read-only

**Alternative NXOS-specific CPU OIDs:**
- **System CPU utilization**: `1.3.6.1.4.1.9.9.305.1.1.1.0` (cseSysCPUUtilization)
  - Data Type: Gauge32 (percentage)
  - Description: Overall system CPU utilization from CISCO-SYSTEM-EXT-MIB
  - Platform support: Confirmed on Nexus 5000 series, likely supported across all NXOS platforms

### Multi-CPU indexing scheme

For systems with multiple CPUs or line cards, CISCO-PROCESS-MIB uses the **cpmCPUTotalTable** indexed by cpmCPUTotalIndex. **Physical entity mapping** occurs via `cpmCPUTotalPhysicalIndex` (OID: `1.3.6.1.4.1.9.9.109.1.1.1.1.2`) which links to the ENTITY-MIB entPhysicalIndex for component identification.

```
Example multi-CPU polling result:
1.3.6.1.4.1.9.9.109.1.1.1.1.8.1 = Gauge32: 6    (Supervisor CPU)
1.3.6.1.4.1.9.9.109.1.1.1.1.8.8 = Gauge32: 1    (Line card 1 CPU)
1.3.6.1.4.1.9.9.109.1.1.1.1.8.9 = Gauge32: 2    (Line card 2 CPU)
```

## Memory usage monitoring

### CISCO-PROCESS-MIB memory objects

**Standard memory monitoring:**
- **Memory used**: `1.3.6.1.4.1.9.9.109.1.1.1.1.12.1` (cpmCPUMemoryUsed)
  - Data Type: Gauge32 (bytes)
  - Description: Total physical memory currently in use
  - Access: read-only

- **Memory free**: `1.3.6.1.4.1.9.9.109.1.1.1.1.13.1` (cpmCPUMemoryFree)
  - Data Type: Gauge32 (bytes)
  - Description: Total physical memory currently available
  - Access: read-only

**Alternative system memory OID:**
- **System memory utilization**: `1.3.6.1.4.1.9.9.305.1.1.2.0` (cseSysMemoryUtilization)
  - Data Type: Gauge32 (percentage)
  - Description: Overall system memory utilization percentage

### Enhanced memory pool monitoring

**CISCO-ENHANCED-MEMPOOL-MIB** provides detailed memory pool statistics with support for multiple memory types:

**Memory pool table** (Base OID: `1.3.6.1.4.1.9.9.221.1.1.1`):
- **Pool type**: `1.3.6.1.4.1.9.9.221.1.1.1.1.2` (cempMemPoolType)
  - Data Type: Integer32
  - Values: 2=processorMemory, 11=reservedMemory, 12=imageMemory

- **Pool used**: `1.3.6.1.4.1.9.9.221.1.1.1.1.7` (cempMemPoolUsed)
  - Data Type: Gauge32 (bytes)
  
- **Pool free**: `1.3.6.1.4.1.9.9.221.1.1.1.1.8` (cempMemPoolFree)
  - Data Type: Gauge32 (bytes)

**High-capacity counters** for systems with >4GB memory:
- **HC pool used**: `1.3.6.1.4.1.9.9.221.1.1.1.1.18` (cempMemPoolHCUsed)
  - Data Type: Counter64 (bytes)
  
- **HC pool free**: `1.3.6.1.4.1.9.9.221.1.1.1.1.20` (cempMemPoolHCFree)
  - Data Type: Counter64 (bytes)

### Memory utilization calculation

```python
memory_used = snmp_get(memory_used_oid)
memory_free = snmp_get(memory_free_oid)
total_memory = memory_used + memory_free
utilization_percent = (memory_used / total_memory) * 100
```

## Disk usage monitoring

### HOST-RESOURCES-MIB disk monitoring

**Storage table** (Base OID: `1.3.6.1.2.1.25.2.3`):
- **Storage index**: `1.3.6.1.2.1.25.2.3.1.1` (hrStorageIndex)
  - Data Type: Integer32
  - Description: Unique identifier for storage entry

- **Storage type**: `1.3.6.1.2.1.25.2.3.1.2` (hrStorageType)
  - Data Type: AutonomousType
  - Description: Type of storage (disk, memory, removable, etc.)

- **Storage description**: `1.3.6.1.2.1.25.2.3.1.3` (hrStorageDescr)
  - Data Type: DisplayString
  - Description: Textual description of storage

- **Allocation units**: `1.3.6.1.2.1.25.2.3.1.4` (hrStorageAllocationUnits)
  - Data Type: Integer32
  - Description: Size of allocation units in bytes

- **Storage size**: `1.3.6.1.2.1.25.2.3.1.5` (hrStorageSize)
  - Data Type: Integer32
  - Description: Size of storage in allocation units

- **Storage used**: `1.3.6.1.2.1.25.2.3.1.6` (hrStorageUsed)  
  - Data Type: Integer32
  - Description: Number of allocation units used

### Disk storage specifics

**Disk storage table** (Base OID: `1.3.6.1.2.1.25.3.6`):
- **Disk access**: `1.3.6.1.2.1.25.3.6.1.1` (hrDiskStorageAccess)
  - Values: 1=readWrite, 2=readOnly

- **Disk media**: `1.3.6.1.2.1.25.3.6.1.2` (hrDiskStorageMedia)
  - Values: 3=hardDisk, 4=floppyDisk, 5=opticalDiskROM

- **Disk capacity**: `1.3.6.1.2.1.25.3.6.1.4` (hrDiskStorageCapacity)
  - Data Type: Integer32 (KBytes)

### UCD-SNMP-MIB disk monitoring

For Unix-like systems, **UCD-SNMP-MIB** provides additional disk monitoring:

**Disk table** (Base OID: `1.3.6.1.4.1.2021.9.1`):
- **Disk path**: `1.3.6.1.4.1.2021.9.1.2` (dskPath)
  - Data Type: DisplayString

- **Total disk size**: `1.3.6.1.4.1.2021.9.1.6` (dskTotal)
  - Data Type: Integer32 (KBytes)

- **Available space**: `1.3.6.1.4.1.2021.9.1.7` (dskAvail)
  - Data Type: Integer32 (KBytes)

- **Used space**: `1.3.6.1.4.1.2021.9.1.8` (dskUsed)
  - Data Type: Integer32 (KBytes)

- **Usage percentage**: `1.3.6.1.4.1.2021.9.1.9` (dskPercent)
  - Data Type: Integer32 (0-100 percent)

## Temperature sensor monitoring

### CISCO-ENTITY-SENSOR-MIB temperature objects

**Primary temperature monitoring** uses CISCO-ENTITY-SENSOR-MIB (Base OID: `1.3.6.1.4.1.9.9.91.1.1.1`):

- **Sensor type**: `1.3.6.1.4.1.9.9.91.1.1.1.1.1` (entSensorType)
  - Data Type: SensorDataType
  - Value: 8 = celsius for temperature sensors

- **Sensor scale**: `1.3.6.1.4.1.9.9.91.1.1.1.1.2` (entSensorScale)
  - Data Type: SensorDataScale
  - Values: 8=milli (10^-3), 9=units (10^0)

- **Sensor precision**: `1.3.6.1.4.1.9.9.91.1.1.1.1.3` (entSensorPrecision)
  - Data Type: SensorPrecision (-8 to 9)
  - Description: Decimal places of precision

- **Sensor value**: `1.3.6.1.4.1.9.9.91.1.1.1.1.4` (entSensorValue)
  - Data Type: SensorValue (Integer32)
  - Description: Current sensor reading in scaled units

- **Sensor status**: `1.3.6.1.4.1.9.9.91.1.1.1.1.5` (entSensorStatus)
  - Data Type: SensorStatus
  - Values: 1=ok, 2=unavailable, 3=nonoperational

### Temperature indexing and name resolution

**Complex indexing patterns**: NXOS uses sophisticated indexing schemes where sensor indexes don't directly correlate to readable names. **Temperature values often appear with indexes like 21598, 101021590, or 120021591**. To resolve sensor names, correlate with ENTITY-MIB using the same index:

**Entity physical name**: `1.3.6.1.2.1.47.1.1.1.1.7.{index}` (entPhysicalName)

```
Example temperature monitoring:
1.3.6.1.4.1.9.9.91.1.1.1.1.4.21598 = INTEGER: 47000 (47°C)
1.3.6.1.2.1.47.1.1.1.1.7.21598 = STRING: "Temperature Sensor 1"
```

### Temperature conversion calculation

NXOS typically returns temperature values in **millidegrees Celsius**:

```python
# Convert NXOS temperature reading
celsius = snmp_value / 1000.0
fahrenheit = (celsius * 9/5) + 32
```

## Fan status monitoring

### CISCO-ENTITY-FRU-CONTROL-MIB fan objects

**Fan monitoring** uses CISCO-ENTITY-FRU-CONTROL-MIB (Base OID: `1.3.6.1.4.1.9.9.117`):

- **Fan tray operational status**: `1.3.6.1.4.1.9.9.117.1.4.1.1.1` (cefcFanTrayOperStatus)
  - Data Type: Integer32
  - Values: 1=unknown, 2=up, 3=down, 4=warning
  - Access: read-only

**Enhanced fan monitoring** (NXOS 9.2(1)+):
- **Fan speed**: `1.3.6.1.4.1.9.9.117.1.4.1.1.2` (cefcFanSpeed)
  - Data Type: Gauge32 (RPM)
  - Access: read-only

- **Fan speed percentage**: `1.3.6.1.4.1.9.9.117.1.4.1.1.3` (cefcFanSpeedPercent)
  - Data Type: Gauge32 (0-100 percent)
  - Access: read-only

### Power supply monitoring

**Power supply status** (same MIB):
- **FRU power operational status**: `1.3.6.1.4.1.9.9.117.1.1.2.1.2` (cefcFRUPowerOperStatus)
  - Data Type: Integer32
  - Values: 1=offEnvOther, 2=on, 3=offAdmin, 4=offDenied, 5=offEnvPower, 6=offEnvTemp, 7=offEnvFan, 8=failed, 9=onButFanFail

- **FRU power administrative status**: `1.3.6.1.4.1.9.9.117.1.1.2.1.1` (cefcFRUPowerAdminStatus)
  - Data Type: Integer32
  - Values: 1=on, 2=off, 3=inlineAuto, 4=inlineOn, 5=powerCycle
  - Access: read-write

### Entity-based fan monitoring

**Alternative approach** using standard ENTITY-SENSOR-MIB for fan speed:
- **Fan RPM sensors**: Look for `entPhySensorType` value of 10 (rpm)
- **Fan sensor value**: `1.3.6.1.2.1.99.1.1.1.4.{index}` (entPhySensorValue)
- **Data Type**: Integer32 (direct RPM reading)

## NXOS-specific considerations and variations

### Platform differences

**Nexus 5000 series**: Limited MIB subset support, different sensor indexing schemes, some enterprise MIBs not available.

**Nexus 7000/9000 series**: Full MIB support, complex chassis indexing for multi-supervisor configurations, line card specific monitoring capabilities.

### Version-specific limitations

**NXOS 6.x**: Different memory baseline utilization (60-70%), limited SNMPv3 functionality in early versions.

**NXOS 7.x**: Enhanced capabilities, ~80% memory baseline utilization, improved MIB support.

**NXOS 8.5.1**: **Critical encryption bug (CSCvy23094)** - DES encryption parameter prevents ISSD/downgrade. Use AES-128 encryption instead.

**NXOS 9.2(1)+**: Enhanced fan monitoring with speed reporting, fixes for encryption issues.

### SNMP configuration requirements

**Packet size considerations**: Default PDU size is 1500 bytes. For bulk operations, increase using:
```
snmp-server packetsize 4096
```
Range: 484 to 17382 bytes.

**Security configuration**: NXOS supports SNMPv1, v2c, and v3 with VRF-aware implementation and context mapping for multi-instance environments.

### VDC and context support

**Virtual Device Context (VDC)** monitoring requires **CISCO-VDC-MIB** (OID: `1.3.6.1.4.1.9.9.774`):
- **VDC ID**: `1.3.6.1.4.1.9.9.774.1.1.1.1.1` (ciscoVdcId)
- **VDC name**: `1.3.6.1.4.1.9.9.774.1.1.1.1.2` (ciscoVdcName)
- **VDC state**: `1.3.6.1.4.1.9.9.774.1.1.1.1.3` (ciscoVdcState)

## Data types and indexing schemes

### Common SNMP data types

**Counter32**: 32-bit unsigned counters (0 to 4,294,967,295) that increase monotonically and wrap at maximum value. Used for interface statistics and accumulating values.

**Counter64**: 64-bit unsigned counters essential for high-speed interfaces (GigE and above) to prevent frequent wrapping issues.

**Gauge32**: 32-bit unsigned values that can increase or decrease, representing instantaneous measurements like CPU utilization, memory usage, and temperature readings.

**Integer32**: Signed 32-bit integers (-2,147,483,648 to 2,147,483,647) used for status values, enumerated types, and configuration parameters.

**DisplayString**: ASCII text strings for descriptive information like interface names, sensor descriptions, and system identifiers.

### Indexing patterns

**entPhysicalIndex**: Primary linking mechanism across all Cisco MIBs, providing hierarchical tree structure representing physical containment relationships (chassis → modules → submodules → sensors).

**Table indexing**: Sequential integers starting from 1 for most table entries, with persistence requirements across agent restarts for consistent monitoring.

**Sparse indexing**: Many Cisco MIBs use sparse indexing where not all possible index values have corresponding entries, requiring walk operations rather than direct gets.

## Implementation examples and best practices

### Production monitoring configuration

**Telegraf SNMP configuration:**
```toml
[[inputs.snmp]]
agents = ["10.1.1.100"]
version = 3
sec_name = "monitor_user"
auth_protocol = "SHA"
auth_password = "auth_password"
sec_level = "authPriv"
priv_protocol = "AES"
priv_password = "priv_password"

[[inputs.snmp.field]]
name = "CPU1m"
oid = "1.3.6.1.4.1.9.9.109.1.1.1.1.7.1"

[[inputs.snmp.field]]
name = "MemUsed"
oid = "1.3.6.1.4.1.9.9.109.1.1.1.1.12.1"

[[inputs.snmp.table]]
name = "temperature"
[[inputs.snmp.table.field]]
name = "temp_value"
oid = "1.3.6.1.4.1.9.9.91.1.1.1.1.4"
```

### Counter wrap handling

**Proper counter arithmetic** for 32-bit interfaces:
```python
def calculate_rate(previous_value, current_value, time_delta):
    max_counter = 2**32 - 1
    if current_value >= previous_value:
        delta = current_value - previous_value
    else:
        # Counter wrapped
        delta = (max_counter - previous_value) + current_value + 1
    return delta / time_delta
```

### Performance optimization

**Polling frequency recommendations**:
- System metrics (CPU, memory): 1-5 minutes
- Interface counters: 30 seconds - 2 minutes  
- Environmental sensors: 5-15 minutes
- Hardware status: 10-30 minutes

**Bulk operations**: Use SNMP GETBULK for table walks to reduce network overhead and improve polling efficiency.

## Conclusion

Cisco NXOS provides comprehensive SNMP monitoring through a combination of standard IETF MIBs and Cisco proprietary extensions. **The key to successful implementation lies in understanding the entPhysicalIndex correlation mechanism** that links data across multiple MIBs, proper handling of version-specific limitations, and implementing robust counter arithmetic for high-speed interfaces.

**Critical success factors** include using SNMPv3 with proper authentication and encryption, implementing appropriate polling intervals based on metric volatility, leveraging bulk operations for table walks, and correlating complex sensor indexes with readable names through ENTITY-MIB queries. The monitoring framework scales effectively across Nexus platforms from 5000 to 9000 series, with enhanced capabilities in newer software versions supporting detailed environmental monitoring and hardware health visibility essential for data center operations.