# SONiC Bulk Counter Design #

## Table of Content

### Revision

 | Rev |     Date    |       Author       | Change Description                |
 |:---:|:-----------:|:------------------:|-----------------------------------|
 | 0.1 |             |      Junchao Chen  | Initial version                   |

### Scope

This document is the design document for bulk counter feature on SONiC.

### Definitions/Abbreviations

N/A

### Overview

PR https://github.com/opencomputeproject/SAI/pull/1352/files introduced new SAI APIs that supports bulk stats:

- sai_bulk_object_get_stats
- sai_bulk_object_clear_stats

SONiC flex counter infrastructure shall utilize bulk stats API to gain better performance. This document discusses how to integrate these two new APIs to SONiC. 

### Requirements

- Syncd shall use bulk stats APIs based on object type. E.g. for a counter group that queries queue and pg stats, queue stats support bulk while pg stats does not, in that case queue stats shall use bulk API, pg stats shall use non bulk API
- For a certain object type in a counter group, it shall use bulk stats only if:
  - The stats capability for each counter IDs shall match the stats mode of the counter group
  - Each object queries exactly the same counter IDs. (Requirement from function signature of sai_bulk_object_get_stats and sai_bulk_object_clear_stats)
- Syncd shall automatically fall back to old way if bulk stats APIs are not supported
- Syncd shall utilize API sai_query_stats_capability to query bulk capability. Syncd shall treat counter as no bulk capability if API sai_query_stats_capability return error except SAI_STATUS_BUFFER_OVERFLOW (SAI_STATUS_BUFFER_OVERFLOW requires a retry with larger buffer)
- Syncd shall call bulk stats API in flex counter thread and avoid calling it in main thread to make sure main thread only handles short and high priority tasks. (This is the default behavior in flex counter infrastructure)
- In phase 1, the change is limited to syncd only, no CLI/swss change. Syncd shall deduce the bulk stats mode according to the stats mode defined in FLEX DB:
  - SAI_STATS_MODE_READ -> SAI_STATS_MODE_BULK_READ
  - SAI_STATS_MODE_READ_AND_CLEAR -> SAI_STATS_MODE_BULK_READ_AND_CLEAR

### Architecture Design

For each counter group, different statistic type is allowed to chooose bulk or non-bulk API based on SAI capability.

![architecture](/doc/bulk_counter/bulk_counter.svg).

> Note: In the picture, pg/queue watermark statistic use bulk API and buffer watermark statistic uses non-bulk API. This is just an example to show the design idea. 

### High-Level Design

Changes shall be made to sonic-sairedis to support this feature. No CLI change. No DB schema change.

> Note: Code present in this design document is only for demonstrating the design idea, it is not production code.

#### sonic-sairedis

##### Bulk Statistic Context

A new structure shall be added to FlexCounter class. 

This structure is created because:

- Meet the signature of sai_bulk_object_get_stats and sai_bulk_object_clear_stats
- Avoid constructing these information each time collecting statistic. The bulk context shall only be updated under below cases:
  - New object join counter group. E.g. adding a new port object.
  - Existing object leave counter group. E.g removing an existing port object.
  - Other case such as counter IDs is updated by user.

```cpp
struct BulkStatsContext
{
    sai_object_type_t object_type;
    std::vector<sai_object_id_t> object_vids;
    std::vector<sai_object_key_t> object_keys; 
    std::vector<sai_stat_id_t> counter_ids;
    std::vector<sai_status_t> statuses;
    std::vector<uint64_t> counters;
    std::shared_ptr<sai_stat_capability_list_t> stats_capas;
};
```
- object_type: object type.
- object_keys: objects that participate the bulk call. E.g. for port, SAI object id value shall be put into sai_object_key_t structure.
- counter_ids: SAI statistic IDs that will be queried/cleared by the bulk call.
- statuses: SAI bulk API return value for each object.
- counters: counter values that will be fill by vendor SAI.
- stats_capas: stats capability for each statitstic IDs for current object type.

The flow of how to updating bulk context will be discussed in following section.

For a given object type, diffrent object instance may support different stats capability, so, a list of BulkStatsContext shall be added to FlexCounter class for each object type.

```cpp

std::vector<BulkStatsContext> m_portBulkContexts;
std::vector<BulkStatsContext> m_priorityGroupBulkContexts;
...

```

##### Update Bulk Context

1. New object join counter group.

![Add Object Flow](/doc/bulk_counter/object_join_counter_group.svg).

2. Existing object leave counter group, related data shall be removed from bulk context.

##### Statistic Collect

![Collect Counter Flow](/doc/bulk_counter/counter_collect.svg).

### SAI API

SAI APIs shall be used in this feature:

- sai_bulk_object_get_stats
- sai_bulk_object_clear_stats

### Configuration and management

N/A

### Warmboot and Fastboot Design Impact

No extra logic on SONiC side is needed to handle warmboot/fastboot.

- As fastboot dealys all counters querying, this feature does not affect fastboot. 
- For warmboot, it is vendor SAI implementation's responsible to make sure that there must be no error if warmboot starts while bulk API is called.

### Restrictions/Limitations

- Bulk collect attribute value is not supported
- Buffer pool stats is not support for bulk because different buffer pool may have different stats mode. E.g. pool1 has mode SAI_STATS_MODE_READ, pool2 has mode SAI_STATS_MODE_READ_AND_CLEAR. The new SAI bulk API only allows specify one stats mode.
- Maximum object number at one bulk call is a limitation based on vendor implementation.

### Performance Improvement

A rough test has been done on Nvidia platform for queue. 

- Non bulk API: get stats for one queue takes X seconds; get stats for 32 port * 8 queue is 256X seconds;
- Bulk API: get stats for one queue takes Y seconds; get stats for 32 port * 8 queue is almost Y seconds;

X is almost euqal to Y. So, more object instances, more performance improvement. 

### Testing Requirements/Design

As this feature does not introduce any new function, unit test shall be good enough to cover the code changes and new sonic-mgmt/VS test cases will be added.

#### Unit Test cases

- test_update_bulk_context
- test_bulk_collect_stats
