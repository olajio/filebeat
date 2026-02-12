# Registry Corruption Analysis: Filebeat/Metricbeat Deployment Issues

## TL;DR - Most Likely Causes

The registry corruption during Ansible deployments is most likely caused by:

1. **Race condition during service restart** - Service stop → registry flush → config update → service start cycle doesn't allow enough time for clean shutdown
2. **File locking from incomplete shutdown** - Windows service doesn't fully release file handles before Ansible attempts restart
3. **Concurrent writes during upgrade** - Old Beat version still writing to registry while new version initializes
4. **Insufficient permissions during state transition** - Service account permissions during upgrade/restart process

**Bottom line**: Windows services, especially those with file-based state like registry files, require proper shutdown grace periods that Ansible's rapid deployment cycle may not provide.

---

## Detailed Root Cause Analysis

### 1. The Registry's Critical Role

Filebeat and Metricbeat use the registry to maintain critical state:

**For Filebeat:**
- **File offsets**: Which byte position in each log file has been read
- **File identity**: Inode information to track file rotation
- **State metadata**: File modification times, sizes, harvester state
- **Meta.json**: Beat instance UUID, version, first start time

**For Metricbeat:**
- **Metric state**: Last collection timestamps
- **Counter values**: For metrics requiring delta calculations
- **Meta.json**: Beat instance UUID, version

**Why corruption is catastrophic:**
- Beat can't determine what data has already been sent
- Service startup fails because it can't read its own state
- No fallback mechanism - corrupted registry = failed start

### 2. Windows Service Lifecycle and File Locking

#### Normal Service Lifecycle
```
1. Service receives STOP signal
2. Service begins graceful shutdown:
   - Stops accepting new inputs
   - Flushes in-memory buffers
   - Writes registry state to disk  ← Critical point
   - Closes file handles
   - Releases locks
3. Service reports "STOPPED" status
4. Windows marks service as fully stopped
```

#### What Ansible Likely Does
```
Ansible Task 1: Stop-Service Filebeat
  └─> Sends STOP signal
  └─> Waits briefly (default: a few seconds)
  └─> Proceeds to next task
      
Ansible Task 2: Copy new filebeat.yml
  └─> Overwrites configuration
      
Ansible Task 3: Start-Service Filebeat
  └─> Sends START signal
  
PROBLEM: Step 1 might not fully complete before Step 3 begins
```

**The Race Condition:**
```
Time    Old Service              Registry Files           New Service
T0      Running (v8.19.4)        Clean state             Not started
T1      STOP signal received     Still being written     Not started  
T2      Shutting down...         Partial write/flush     Not started
T3      95% shut down            File locks still held   Not started
T4      [Ansible proceeds]       LOCKS STILL ACTIVE!     Not started
T5      Fully stopped            Lock released           START signal received
T6      Stopped                  Partial/corrupted       Tries to read registry
T7      Stopped                  Corrupted state         STARTUP FAILS! 💥
```

### 3. Specific Scenarios That Cause Corruption

#### Scenario A: Premature Service Start
```yaml
# Ansible playbook (problematic)
- name: Stop Filebeat
  win_service:
    name: filebeat
    state: stopped
  # Returns immediately after service reports "stopping"
  # Doesn't wait for file handles to be released

- name: Update configuration
  win_copy:
    src: filebeat.yml
    dest: 'C:\Program Files\Filebeat\filebeat.yml'

- name: Start Filebeat
  win_service:
    name: filebeat
    state: started
  # Service starts while old instance still has registry locks
```

**Result**: New service instance can't read/write registry → startup fails

#### Scenario B: Interrupted Registry Write
```
1. Old service receives STOP
2. Service begins writing final registry state
3. File: C:\Program Files\Filebeat\data\registry\filebeat\log.json
4. Write is 60% complete
5. Ansible timeout expires or process is killed
6. Partial JSON written: {"files":[{"source":"C:\logs\app.log","offset":12 [TRUNCATED]
7. New service reads corrupted JSON → parsing fails → startup fails
```

#### Scenario C: Meta.json UUID Mismatch
```
Old Instance:
  UUID: a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6
  Writes to registry with this UUID

Ansible Upgrade:
  Removes old version
  Installs new version
  Meta.json gets regenerated with NEW UUID: z9y8x7w6-v5u4-t3s2-r1q0-p9o8n7m6l5k4

New Instance:
  Reads meta.json (new UUID)
  Reads registry files (old UUID)
  UUID MISMATCH → Treats as corrupted → startup fails
```

#### Scenario D: NTFS File Locking on Windows
Windows file locking is more aggressive than Linux:

```
# Linux behavior:
Service stops → File handles released → File immediately available

# Windows behavior:
Service stops → File handles marked for closure → 
  → Background flush to disk (async) → 
  → File locks held during flush → 
  → Lock released after flush completes (could be seconds)
```

**If Ansible starts new service during Windows background flush**: Lock conflict → corruption

### 4. Why Manual Deletion Works

When you manually delete the registry and meta.json:

```powershell
Remove-Item -Path "C:\Program Files\Filebeat\data\registry" -Recurse -Force
Remove-Item -Path "C:\Program Files\Filebeat\data\meta.json" -Force
Start-Service Filebeat
```

**What happens:**
1. **Clean slate**: No existing state to corrupt
2. **Fresh meta.json**: New UUID generated on first start
3. **New registry**: Built from scratch as service reads files
4. **No locks**: No file locking conflicts
5. **No corruption**: No partial/invalid state to read

**Trade-off:**
- ✅ Service starts successfully
- ❌ Loses file position tracking (may re-read files or skip data)
- ❌ Loses metric state (counters reset)

### 5. Why It's Intermittent

The issue doesn't happen every time because it depends on timing variables:

**Variables that affect occurrence:**
- **Server load**: High CPU/disk usage → slower shutdown
- **File size**: Large registry files take longer to flush
- **Number of inputs**: More inputs = more registry state = longer write
- **Windows I/O scheduler**: Non-deterministic flush timing
- **Ansible execution speed**: Network latency, task execution timing varies
- **Service shutdown grace period**: Windows internal timeout varies

**Example timeline variation:**
```
Fast shutdown (lucky case):
T0: STOP signal → T1: Registry flush → T2: Locks released → T3: START signal ✅

Slow shutdown (corruption case):
T0: STOP signal → T1: Registry flush begins → T2: START signal (too early!) → T3: Lock conflict → T4: Corruption 💥
```

### 6. Technical Deep Dive: Registry File Structure

#### Log.json Structure (Filebeat)
```json
{
  "version": "1",
  "cursor": {
    "files": [
      {
        "source": "C:\\logs\\application.log",
        "offset": 2048576,
        "timestamp": "2024-02-11T10:30:00.000Z",
        "identifier_name": "native",
        "inode": 12345678,
        "device_id": 9876
      }
    ]
  }
}
```

**Corruption types:**
1. **Truncated JSON**: File write interrupted mid-write
2. **Invalid offset**: Offset larger than actual file size
3. **Missing fields**: Required fields not present
4. **Lock marker**: Windows lock file (.lock) left behind

#### Meta.json Structure
```json
{
  "uuid": "a1b2c3d4-e5f6-g7h8-i9j0-k1l2m3n4o5p6",
  "first_start": "2024-01-15T08:00:00.000Z",
  "version": "8.19.4"
}
```

**Corruption impact:**
- **UUID change**: Registry state becomes orphaned
- **Version mismatch**: Upgrade detection fails
- **Invalid JSON**: Parse error on startup

### 7. Windows-Specific Considerations

#### Windows Service Manager Behavior
```
sc.exe stop filebeat
  → Sends SERVICE_CONTROL_STOP
  → Waits for SERVICE_STOPPED status
  → Default timeout: 30 seconds
  → If timeout: Force terminate (dirty shutdown)
```

**Ansible's win_service might not wait long enough** for:
- Large registry flush
- Network I/O to Elasticsearch (pending acknowledgments)
- Graceful harvester shutdown

#### NTFS Journaling and Caching
Windows NTFS write caching can complicate things:
```
Application writes to file → 
  → Write cached in memory → 
  → Background flush to disk (lazy write) → 
  → Journal commit → 
  → File available
```

If service is killed during "background flush", registry is partially written.

### 8. Why Upgrades Are More Problematic

During a version upgrade:
```
1. Stop old version (8.19.3)
2. Uninstall old version
   - May leave registry files if "preserve data" option used
   - May partially clean registry
3. Install new version (8.19.4)
4. Start new version
   - Tries to read old registry with new code
   - Registry format may have changed between versions
   - Incompatibility → Corruption detected → Startup fails
```

## Prevention Strategies

### Solution 1: Add Proper Wait Times in Ansible

```yaml
- name: Stop Filebeat
  win_service:
    name: filebeat
    state: stopped

- name: Wait for service to fully stop and release locks
  win_shell: |
    $timeout = 30
    $elapsed = 0
    while ((Get-Process -Name filebeat -ErrorAction SilentlyContinue) -and ($elapsed -lt $timeout)) {
      Start-Sleep -Seconds 1
      $elapsed++
    }
    # Extra grace period for file lock release
    Start-Sleep -Seconds 5
  
- name: Verify registry is not locked
  win_shell: |
    $registryPath = "C:\Program Files\Filebeat\data\registry"
    $lockFile = Get-ChildItem -Path $registryPath -Filter "*.lock" -Recurse -ErrorAction SilentlyContinue
    if ($lockFile) {
      throw "Registry still locked!"
    }

- name: Update configuration
  win_copy:
    src: filebeat.yml
    dest: 'C:\Program Files\Filebeat\filebeat.yml'

- name: Start Filebeat
  win_service:
    name: filebeat
    state: started
```

### Solution 2: Pre-emptive Registry Backup and Restore

```yaml
- name: Backup registry before upgrade
  win_copy:
    src: 'C:\Program Files\Filebeat\data\registry'
    dest: 'C:\Program Files\Filebeat\data\registry.backup'
    remote_src: yes

- name: Attempt normal deployment
  block:
    - name: Stop service
      win_service:
        name: filebeat
        state: stopped
    
    - name: Wait for shutdown
      win_shell: Start-Sleep -Seconds 10
    
    - name: Update config
      win_copy:
        src: filebeat.yml
        dest: 'C:\Program Files\Filebeat\filebeat.yml'
    
    - name: Start service
      win_service:
        name: filebeat
        state: started
    
    - name: Verify service started
      win_service_info:
        name: filebeat
      register: service_status
      failed_when: service_status.services[0].state != 'started'
  
  rescue:
    - name: Service failed to start - restore registry
      win_copy:
        src: 'C:\Program Files\Filebeat\data\registry.backup'
        dest: 'C:\Program Files\Filebeat\data\registry'
        remote_src: yes
    
    - name: Retry start
      win_service:
        name: filebeat
        state: started
```

### Solution 3: Clean Slate Deployment Strategy

```yaml
# Accept that we'll lose state and start fresh
- name: Stop Filebeat
  win_service:
    name: filebeat
    state: stopped

- name: Clean registry (controlled)
  win_file:
    path: "{{ item }}"
    state: absent
  loop:
    - 'C:\Program Files\Filebeat\data\registry'
    - 'C:\Program Files\Filebeat\data\meta.json'

- name: Update configuration
  win_copy:
    src: filebeat.yml
    dest: 'C:\Program Files\Filebeat\filebeat.yml'

- name: Start Filebeat (fresh state)
  win_service:
    name: filebeat
    state: started
```

**Trade-offs:**
- ✅ Guaranteed clean start
- ❌ Potential data re-ingestion or gaps
- ❌ Loss of file position tracking

### Solution 4: Use Filebeat's Built-in Shutdown Endpoint

```yaml
- name: Gracefully stop Filebeat via API
  win_uri:
    url: http://localhost:5066/
    method: DELETE
    timeout: 30
  ignore_errors: yes

- name: Wait for graceful shutdown
  win_shell: Start-Sleep -Seconds 10

- name: Force stop if still running
  win_service:
    name: filebeat
    state: stopped
```

### Solution 5: Increase Service Shutdown Timeout

```yaml
- name: Increase service shutdown timeout
  win_regedit:
    path: HKLM:\SYSTEM\CurrentControlSet\Services\filebeat
    name: ServiceTimeout
    data: 60000  # 60 seconds in milliseconds
    type: dword

- name: Stop service with extended timeout
  win_service:
    name: filebeat
    state: stopped
```

## Recommended Ansible Playbook Structure

```yaml
---
- name: Deploy Filebeat Safely
  hosts: windows_servers
  tasks:
    
    - name: Pre-flight - Check current service status
      win_service_info:
        name: filebeat
      register: filebeat_status
    
    - name: Stop Filebeat gracefully
      win_service:
        name: filebeat
        state: stopped
      when: filebeat_status.services[0].state == 'started'
    
    - name: Wait for full shutdown (critical!)
      win_shell: |
        # Wait for process to exit
        $timeout = 30
        $elapsed = 0
        while ((Get-Process -Name filebeat -ErrorAction SilentlyContinue) -and ($elapsed -lt $timeout)) {
          Start-Sleep -Seconds 1
          $elapsed++
        }
        # Additional grace period for file system operations
        Start-Sleep -Seconds 5
      when: filebeat_status.services[0].state == 'started'
    
    - name: Verify no file locks on registry
      win_shell: |
        $registryPath = "C:\Program Files\Filebeat\data\registry"
        if (Test-Path $registryPath) {
          try {
            [System.IO.File]::OpenWrite("$registryPath\test.lock").Close()
            Remove-Item "$registryPath\test.lock" -Force
          } catch {
            throw "Registry directory is locked"
          }
        }
    
    - name: Backup existing registry
      win_copy:
        src: 'C:\Program Files\Filebeat\data'
        dest: 'C:\Program Files\Filebeat\data.backup'
        remote_src: yes
      ignore_errors: yes
    
    - name: Deploy new configuration
      win_copy:
        src: filebeat.yml
        dest: 'C:\Program Files\Filebeat\filebeat.yml'
    
    - name: Validate configuration syntax
      win_shell: |
        & "C:\Program Files\Filebeat\filebeat.exe" test config -c "C:\Program Files\Filebeat\filebeat.yml"
      register: config_validation
      failed_when: config_validation.rc != 0
    
    - name: Start Filebeat
      win_service:
        name: filebeat
        state: started
      register: start_result
    
    - name: Verify service started successfully
      win_service_info:
        name: filebeat
      register: verify_status
      retries: 3
      delay: 5
      until: verify_status.services[0].state == 'started'
    
    - name: Check Filebeat is shipping data
      win_shell: |
        # Check if Filebeat is connecting to Elasticsearch
        Start-Sleep -Seconds 10
        Get-Content "C:\Program Files\Filebeat\logs\filebeat" -Tail 50
      register: filebeat_logs
    
    - name: Handle startup failure
      block:
        - name: Clean corrupted registry
          win_file:
            path: "{{ item }}"
            state: absent
          loop:
            - 'C:\Program Files\Filebeat\data\registry'
            - 'C:\Program Files\Filebeat\data\meta.json'
        
        - name: Restart with clean state
          win_service:
            name: filebeat
            state: restarted
      when: verify_status.services[0].state != 'started'
```

## Long-term Monitoring Recommendations

### Add Pre-deployment Health Checks
```yaml
- name: Check for registry corruption before deployment
  win_shell: |
    $metaJson = "C:\Program Files\Filebeat\data\meta.json"
    if (Test-Path $metaJson) {
      try {
        $content = Get-Content $metaJson -Raw | ConvertFrom-Json
        if (-not $content.uuid) {
          throw "Invalid meta.json structure"
        }
      } catch {
        throw "meta.json is corrupted"
      }
    }
```

### Log Registry State Before/After
```yaml
- name: Capture registry state before deployment
  win_shell: |
    Get-ChildItem "C:\Program Files\Filebeat\data\registry" -Recurse | 
      Select-Object FullName, Length, LastWriteTime | 
      ConvertTo-Json
  register: registry_before

- name: Capture registry state after deployment
  win_shell: |
    Get-ChildItem "C:\Program Files\Filebeat\data\registry" -Recurse | 
      Select-Object FullName, Length, LastWriteTime | 
      ConvertTo-Json
  register: registry_after
```

## Conclusion

The registry corruption is almost certainly caused by **insufficient grace period between service shutdown and restart** during Ansible automation. Windows file locking, async disk I/O, and registry flush operations require more time than Ansible's default wait periods provide.

**Immediate fix**: Add 10-15 second delays after service stop  
**Better fix**: Implement process monitoring and file lock verification  
**Best fix**: Use clean slate deployment or implement proper registry backup/restore logic

The intermittent nature is explained by variable timing - sometimes the service shuts down fast enough, sometimes it doesn't.
