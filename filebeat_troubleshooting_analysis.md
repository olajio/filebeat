# Filebeat Conditional Index Routing - Root Cause Analysis

## TL;DR - Most Likely Cause
**The issue is likely due to how `when.contains` evaluates dotted field names at the root level.**

When you use `fields_under_root: true`, fields with dots in their names (like `service.name`) may be stored as:
- A nested object: `service: { name: "value" }`
- OR a literal dotted field: `"service.name": "value"`

The `when.contains` condition expects a specific format, and there's likely a mismatch between how the field is stored vs. how you're referencing it.

## Detailed Analysis

### 1. Field Name Format Inconsistency

**The Problem:**
You're using two different field naming conventions:
- **Hyphenated**: `app-name` (works as a single field)
- **Dotted (ECS style)**: `service.name` (could be nested or flat)

**What's happening:**
```yaml
# These fields with hyphens work as expected:
fields:
  app-name: blueprism
fields_under_root: True

# These ECS dotted fields might not match correctly:
fields:
  service.name: "netapp-metrics"
  service.type: "netapp"
  service.environment: "prod"
fields_under_root: true
```

When Filebeat processes `service.name` with `fields_under_root: true`, it may create:
```json
{
  "service": {
    "name": "netapp-metrics",
    "type": "netapp",
    "environment": "prod"
  }
}
```

But your condition is looking for:
```yaml
when.contains:
  service.name: "netapp-metrics"  # This expects a flat field
```

### 2. Condition Evaluation Order

Filebeat evaluates `indices` conditions **from top to bottom** and uses **first match wins**.

**Current logic flow:**
1. Check: Does event contain `app-name: "blueprism"`? 
   - If YES → Route to `filebeat-8.19.4` (STOP)
   - If NO → Continue to next condition

2. Check: Does event contain `service.name: filebeat-logs`?
   - If condition fails to match properly → Continue

3. Check: Does event contain `service.name: "netapp-metrics"`?
   - If condition fails to match properly → Continue

4. If no conditions match → **Default to first index pattern** or error

**The critical issue:** If the `service.name` conditions aren't matching due to field format issues, events may be defaulting to the first index pattern.

### 3. Why the Workaround Works

When you set all indices to `filebeat-8.19.4-inframon`:
```yaml
indices:
  - index: "filebeat-%{[agent.version]}-inframon"
    when.contains:
      app-name: "blueprism"
  - index: "filebeat-%{[agent.version]}-inframon"  # Same index!
    when.contains:
      service.name: filebeat-logs
```

**It doesn't matter which condition matches** because they all route to the same destination. This masks the underlying condition matching issue.

## Solutions to Test

### Solution 1: Use Bracket Notation for Nested Fields

Try referencing the nested field structure explicitly:

```yaml
output.elasticsearch:
  indices:
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        app-name: "blueprism"
    
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        service.name: "filebeat-logs"
    
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        "[service][name]": "netapp-metrics"  # Bracket notation
    
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        "[service][name]": "solidfire-metrics"
    
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        "[service][name]": "inframon-logs"
```

### Solution 2: Use Equals Instead of Contains

The `when.contains` condition does substring matching. Try `when.equals` for exact matching:

```yaml
output.elasticsearch:
  indices:
    - index: "filebeat-%{[agent.version]}"
      when.equals:
        app-name: "blueprism"
    
    - index: "filebeat-%{[agent.version]}"
      when.equals:
        service.name: "filebeat-logs"
    
    - index: "filebeat-%{[agent.version]}-inframon"
      when.equals:
        service.name: "netapp-metrics"
```

### Solution 3: Flatten the Field Names (Recommended)

Change your dotted field names to use a different separator:

```yaml
# Instead of ECS-style dotted names:
fields:
  service.name: "netapp-metrics"
  service.type: "netapp"
  service.environment: "prod"

# Use hyphens or underscores:
fields:
  service-name: "netapp-metrics"
  service-type: "netapp"
  service-environment: "prod"
```

Then update your conditions:
```yaml
output.elasticsearch:
  indices:
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        service-name: "netapp-metrics"
```

### Solution 4: Add Explicit Field Mapping Processor

Use a processor to create a guaranteed flat field for routing:

```yaml
processors:
  # Add this BEFORE other processors
  - script:
      lang: javascript
      source: >
        function process(event) {
          var serviceName = event.Get("service.name");
          if (serviceName) {
            event.Put("routing_key", serviceName);
          }
          var appName = event.Get("app-name");
          if (appName) {
            event.Put("routing_key", appName);
          }
        }

output.elasticsearch:
  indices:
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        routing_key: "blueprism"
    
    - index: "filebeat-%{[agent.version]}"
      when.contains:
        routing_key: "filebeat-logs"
    
    - index: "filebeat-%{[agent.version]}-inframon"
      when.contains:
        routing_key: "netapp-metrics"
```

### Solution 5: Use Different Filebeat Instances (If All Else Fails)

Run separate Filebeat instances for different data types:
- One for blueprism/filebeat logs → outputs to `filebeat-8.19.4`
- One for inframon logs → outputs to `filebeat-8.19.4-inframon`

This is more operationally complex but guarantees separation.

## Debugging Steps

### Step 1: Enable Debug Logging

```yaml
logging:
  level: debug
  selectors: [publish, outputs]
```

This will show you:
- What index each event is being routed to
- Whether conditions are being evaluated

### Step 2: Test Field Availability

Add a temporary processor to dump field structure:

```yaml
processors:
  - script:
      lang: javascript
      source: >
        function process(event) {
          console.log("Event fields: " + JSON.stringify(event.fields));
          console.log("service.name: " + event.Get("service.name"));
          console.log("[service][name]: " + event.Get("[service][name]"));
          console.log("app-name: " + event.Get("app-name"));
        }
```

### Step 3: Check Actual Event Structure in Elasticsearch

Query one of the events and examine its structure:

```json
GET filebeat-8.19.4/_search
{
  "size": 1,
  "query": {
    "exists": {
      "field": "service"
    }
  }
}
```

Look at the `_source` to see if `service` is:
- `{"service": {"name": "value"}}` (nested)
- `{"service.name": "value"}` (flat)

## Recommended Action Plan

1. **Immediate**: Use Solution 3 (flatten field names) as it's the most reliable
2. **Short-term**: Test Solution 1 (bracket notation) if you want to keep ECS compatibility
3. **Long-term**: Work with Elastic Support to understand the expected behavior in 8.19.4

## Expected Behavior vs. Actual

**Expected:**
- Filebeat should handle dotted field names consistently with `fields_under_root: true`
- Conditional routing should work regardless of field name format
- Documentation should clearly specify how to reference fields in conditions

**Actual:**
- Dotted field names with `fields_under_root: true` may create nested structures
- `when.contains` may not handle nested fields correctly
- Workaround of using identical index patterns "fixes" the symptom but not the cause

## Additional Considerations

### Filebeat 8.x Changes
Elastic has made changes to how fields are handled in Filebeat 8.x, particularly around ECS compliance. This might be a regression or an undocumented behavior change.

### Alternative: Use Index Template Routing
If conditional routing continues to be problematic, consider using index template aliases or ingest pipeline routing as an alternative approach.

---

**Bottom Line:** The most likely issue is that `when.contains` doesn't properly match dotted field names when they're stored as nested objects. Test the solutions in order, starting with flattening your field names.
