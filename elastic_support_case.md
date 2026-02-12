# Elastic Support Case: Filebeat Conditional Index Routing Not Working

## Environment
- **Filebeat Version**: 8.19.4
- **Elasticsearch Version**: 8.19.4
- **Operating System**: Windows
- **Deployment**: On-premises
- **Cluster**: hselkprod.hedgeserv.net:9243

## Issue Summary
Filebeat conditional index routing via `output.elasticsearch.indices` is not functioning as expected. All events are being routed to the first index pattern regardless of the `when.contains` conditions specified in the configuration.

## Expected Behavior
Based on the configuration in `output.elasticsearch.indices`, events should be routed to different indices based on field conditions:

1. Events with `app-name: "blueprism"` → `filebeat-8.19.4` (first condition)
2. Events with `service.name: filebeat-logs` → `filebeat-8.19.4` (second condition)
3. Events with `service.name: "netapp-metrics"` → `filebeat-8.19.4-inframon` (third condition)
4. Events with `service.name: "solidfire-metrics"` → `filebeat-8.19.4-inframon` (fourth condition)
5. Events with `service.name: "inframon-logs"` → `filebeat-8.19.4-inframon` (fifth condition)

## Actual Behavior
**All events are being routed to `filebeat-8.19.4` regardless of the field values**, indicating that the conditional logic is either:
- Not being evaluated correctly
- Being ignored entirely
- Falling through to a default behavior

## Configuration Details

### Original Configuration (Not Working)
```yaml
output.elasticsearch:
  hosts: ["https://hselkprod.hedgeserv.net:9243"]
  username: "filebeat_internal"
  password: "${ES_PWD}"
  indices:
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        app-name: "blueprism"
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        service.name: filebeat-logs
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        service.name: "netapp-metrics"
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        service.name: "solidfire-metrics"
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        service.name: "inframon-logs"
  pipeline: filebeat_main
  ssl.verification_mode: "none"
```

### Field Configuration (Inputs)
All fields are configured with `fields_under_root: true`, which should place them at the root level of the event:

**BluePrism logs:**
```yaml
fields:
  app-name: blueprism
fields_under_root: True
```

**Filebeat logs:**
```yaml
fields:
  service.name: filebeat-logs
  service.type: filebeat
  service.environment: PROD
fields_under_root: True
```

**Infrastructure monitoring logs:**
```yaml
fields:
  service.name: "netapp-metrics"
  service.type: "netapp"
  service.environment: "prod"
fields_under_root: true
```

## Workaround That Works
When all index conditions are changed to route to the same index pattern (`filebeat-8.19.4-inframon`), the routing works correctly and all events successfully arrive at the specified index:

```yaml
output.elasticsearch:
  indices:
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        app-name: "blueprism"
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        service.name: filebeat-logs
    # ... (all pointing to same index pattern)
```

This confirms that:
1. Filebeat is successfully connecting to Elasticsearch
2. The field values are being populated correctly
3. The issue is specifically with conditional routing to **different** index patterns

## Questions for Elastic Support

1. **Is there a known issue in Filebeat 8.19.4** with conditional index routing where all events default to the first index pattern when multiple different patterns are specified?

2. **Field Reference Format**: Should the `when.contains` condition reference fields differently when `fields_under_root: true` is used? 
   - Current: `service.name: "netapp-metrics"`
   - Should it be: `[service.name]: "netapp-metrics"` or `[service][name]: "netapp-metrics"`?

3. **Conditional Logic Evaluation**: In what order are the `indices` conditions evaluated? Is it:
   - First match wins (stop on first true condition)?
   - Last match wins?
   - All conditions evaluated and most specific wins?

4. **Field Name Format**: Does Filebeat handle dotted field names (e.g., `service.name`) differently than hyphenated field names (e.g., `app-name`) in conditional routing?

5. **Debugging Recommendations**: What is the recommended way to debug conditional index routing issues? Are there specific log levels or debug output that would show:
   - Which conditions are being evaluated?
   - Which condition matched (if any)?
   - Why a particular index was selected?

## Additional Context

### Processors Configuration
We have several processors that might affect field availability:
```yaml
processors:
  - add_locale: ~
  - add_labels:
      labels:
        app_code: BPRBT
        datacenter: ES01
        domain: funddevelopmentservices.com
  - rename:
      when:
        not:
          has_fields: ["host.hostname"]
      fields:
        - from: "host.name"
          to: "host.hostname"
  - drop_fields:
      fields: ["agent.ephemeral_id","agent.hostname","agent.id","agent.name","agent.type","ecs.version","metricset.period","input.type"]
```

**Question**: Could the processors be executing after the index routing decision is made, causing field availability issues?

### ILM and Template Settings
```yaml
setup.ilm.enabled: false
setup.template.enabled: false
setup.dashboards.enabled: false
```

## Impact
- **Severity**: Medium
- **Business Impact**: Unable to properly segregate monitoring data (inframon) from application logs (blueprism, filebeat), leading to:
  - Inefficient index management
  - Complicated data lifecycle policies
  - Difficulty in applying different retention policies to different data types

## Request
Please investigate why conditional index routing is not working as expected in Filebeat 8.19.4 and provide guidance on the correct configuration syntax or identify if this is a bug requiring a patch.

## Attachments
- filebeat_original.yml (not working configuration)
- filebeat_working.yml (workaround configuration)

---
**Case Priority**: Medium  
**Preferred Contact Method**: Email  
**Environment Type**: Production
