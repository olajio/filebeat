# Elastic Support Case: Filebeat/Metricbeat Service Fails to Start After Ansible Deployment

## Environment
- **Filebeat Version**: 8.19.4
- **Metricbeat Version**: 8.19.4
- **Elasticsearch Version**: 8.19.4
- **Operating System**: Windows Server
- **Deployment Method**: Ansible via AWS CodeBuild
- **Deployment Type**: Production
- **Cluster**: hselkprod.hedgeserv.net:9243

## Issue Summary
Filebeat and/or Metricbeat services intermittently fail to start after deployment via Ansible automation through AWS CodeBuild. The issue is resolved by manually deleting the registry folder and meta.json file, then restarting the service. This suggests registry corruption or lock issues during the automated installation process.

## Frequency
- **Occurrence**: Intermittent (not every deployment)
- **Pattern**: Unpredictable - sometimes Filebeat fails, sometimes Metricbeat fails, sometimes both start successfully
- **Impact**: Requires manual intervention, delays deployment completion, potential data loss during downtime

## Expected Behavior
After Ansible deployment via AWS CodeBuild:
1. Filebeat/Metricbeat should be installed successfully
2. Configuration files should be updated correctly
3. Service should start automatically or on first manual start attempt
4. Service should connect to Elasticsearch cluster and begin shipping data
5. Registry files should be created/updated without corruption

## Actual Behavior
After Ansible deployment via AWS CodeBuild:
1. Installation completes without errors in Ansible output
2. Configuration files are correctly deployed
3. **Service fails to start** (intermittently)
4. Service status shows error or failure state
5. No data is shipped to Elasticsearch

**Manual Resolution Required:**
1. Stop the Filebeat/Metricbeat service (if running)
2. Delete: `C:\Program Files\Filebeat\data\registry` (entire folder)
3. Delete: `C:\Program Files\Filebeat\data\meta.json` (file)
4. Start the Filebeat/Metricbeat service
5. Service starts successfully and begins shipping data

Same pattern occurs for Metricbeat:
- Delete: `C:\Program Files\Metricbeat\data\registry`
- Delete: `C:\Program Files\Metricbeat\data\meta.json`

## Deployment Details

### Infrastructure
- **Platform**: AWS EC2 Windows Server instances
- **Automation**: Ansible playbooks executed via AWS CodeBuild
- **Deployment Trigger**: Automated CI/CD pipeline
- **Target Servers**: Multiple Windows servers in production environment

### Ansible Deployment Process
Our Ansible playbook typically performs:
1. Download Filebeat/Metricbeat MSI installer (if needed)
2. Install or upgrade Filebeat/Metricbeat
3. Deploy configuration files (filebeat.yml / metricbeat.yml)
4. Set environment variables (ES_PWD for Elasticsearch credentials)
5. Stop existing service (if running)
6. Start/restart service
7. Verify service is running

### Potential Timing Issues
The deployment process may have race conditions:
- Service stop → configuration update → service start cycle
- Registry file writes occurring during service state transitions
- File locks from previous service instance not fully released
- Concurrent access to registry files during upgrade

## Registry File Details

### Registry Folder Structure
```
C:\Program Files\Filebeat\data\registry\
├── filebeat/
│   ├── log.json
│   ├── active.dat
│   └── [other registry files]
```

### Meta.json Content
The meta.json file contains Beat instance metadata:
```json
{
  "uuid": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "first_start": "2024-XX-XXT00:00:00.000Z",
  "version": "8.19.4"
}
```

### Symptoms When Corrupted
When the registry causes startup failure:
- Service fails to transition to "Running" state
- Windows Event Log may show service start failures
- Filebeat/Metricbeat logs show errors (if they exist before crash)
- Service appears to start then immediately stops

## Questions for Elastic Support

### 1. Root Cause Identification
- **Is there a known issue in Filebeat/Metricbeat 8.19.4** with registry corruption during service restarts or upgrades on Windows?
- **What specific conditions** cause the registry files to become corrupted or locked?
- **Are there any race conditions** between service shutdown and registry file writes that could cause this?

### 2. Registry File Locking
- **How does Filebeat/Metricbeat handle registry file locks** during service stops and starts?
- **What is the expected grace period** between stopping a service and starting it again?
- **Does the service properly release file locks** on the registry folder when stopping?
- **Are there any Windows-specific file locking issues** we should be aware of?

### 3. Upgrade and Deployment Best Practices
- **What is the recommended procedure** for upgrading Filebeat/Metricbeat via automation on Windows?
- **Should we implement a delay** between service stop and service start operations?
- **Should we always delete registry files** during upgrades to prevent corruption?
- **Is there a safe way to preserve the registry** across deployments while avoiding corruption?

### 4. Configuration Management
- **Does changing the configuration file** while the service is running cause registry issues?
- **Should the service be stopped before updating** filebeat.yml or metricbeat.yml?
- **Are there any configuration settings** that make the registry more resilient to corruption?

### 5. Diagnostic Information
- **What logs or debug output** would help diagnose why the registry becomes corrupted?
- **Are there specific log levels** we should enable to capture this issue when it occurs?
- **What information from the registry files** would help you diagnose the root cause?
- **Should we preserve corrupted registry files** for analysis when this occurs?

### 6. Prevention Strategies
- **Can we pre-emptively detect** when registry corruption might occur?
- **Are there registry validation commands** we can run as part of deployment?
- **Should we implement registry backups** before each deployment?
- **Is there a configuration option** to make the Beat more tolerant of registry issues?

## Impact Assessment

### Business Impact
- **Severity**: Medium-High
- **Affected Systems**: Multiple production Windows servers
- **Data Loss Risk**: Potential gap in monitoring data during manual intervention period
- **Operational Overhead**: Manual intervention required for each occurrence
- **Deployment Delays**: Automated deployments cannot complete without manual verification

### Frequency and Scale
- Occurs intermittently across multiple deployments
- Affects both Filebeat and Metricbeat
- Requires manual verification of service status after every deployment
- Unpredictable nature makes it difficult to plan for

## Workaround Currently Implemented

### Manual Remediation Steps
```powershell
# Stop the service
Stop-Service Filebeat

# Delete registry and meta files
Remove-Item -Path "C:\Program Files\Filebeat\data\registry" -Recurse -Force
Remove-Item -Path "C:\Program Files\Filebeat\data\meta.json" -Force

# Start the service
Start-Service Filebeat

# Verify
Get-Service Filebeat
```

### Limitations of Workaround
- Requires manual intervention (defeats automation purpose)
- Loses registry state (file position tracking, etc.)
- May cause data duplication or loss depending on inputs
- Not scalable across many servers
- Delays deployment completion

## Attempted Solutions (None Successful)

We have tried various approaches without consistent success:
1. Adding delays between service stop and start (still intermittent)
2. Ensuring service is fully stopped before configuration updates
3. Verifying file locks are released before deployment
4. Different Ansible task ordering
5. Checking Windows Event Logs for service errors

None of these have reliably prevented the issue.

## Request

We need Elastic's guidance on:
1. **Root cause** of registry corruption during automated deployments
2. **Best practices** for deploying Beats via automation on Windows
3. **Configuration changes** that might prevent this issue
4. **Whether this is a known bug** that has been addressed in newer versions
5. **Recommended deployment workflow** that avoids registry corruption

## Additional Information Available

We can provide:
- Complete Ansible playbook code
- Filebeat/Metricbeat configuration files
- Windows Event Logs from failed startup attempts
- Corrupted registry files (if we preserve them on next occurrence)
- Filebeat/Metricbeat logs showing startup failures
- Timeline of service state transitions during deployment

## Environment Details

### Windows Configuration
- Windows Server version: [Specify your version]
- Filebeat/Metricbeat running as: Local System / Service Account
- File system: NTFS
- Antivirus: [Specify if applicable]
- Security policies: [Any relevant Group Policies]

### Ansible Configuration
- Ansible version: [Specify]
- WinRM configuration: [Basic/NTLM/Kerberos]
- Execution policy: [Specify]
- Connection method: [Specify]

### Network and Security
- Firewall rules: Elasticsearch connection allowed
- SSL/TLS verification: Disabled (ssl.verification_mode: "none")
- Authentication: Username/password via environment variable

---

**Case Priority**: High  
**Preferred Contact Method**: Email  
**Environment Type**: Production  
**Request Type**: Bug Investigation + Best Practices Guidance

## Attachment Checklist
- [ ] Ansible playbook (sanitized)
- [ ] Filebeat configuration (filebeat.yml)
- [ ] Metricbeat configuration (metricbeat.yml)
- [ ] Windows Event Logs from failure
- [ ] Filebeat/Metricbeat logs
- [ ] Corrupted registry files (when next occurrence is captured)
