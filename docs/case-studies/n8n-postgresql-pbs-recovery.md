# Case Study: n8n / PostgreSQL recovery after PBS restore reproduced the same faulty state

## Incident summary

A Proxmox Backup Server restore was performed for the virtual machine hosting n8n and PostgreSQL.

The restore process itself completed, but after starting the recovered system the same application problems reappeared.

This was an important finding:

the backup was technically valid, but it contained an already unhealthy application state.

The incident demonstrated the difference between:

- a successful infrastructure-level restore,
- and an application that is actually healthy after recovery.

## Environment

Relevant components:

- Proxmox VE
- Proxmox Backup Server
- Linux virtual machine
- Docker / Docker Compose
- n8n
- PostgreSQL
- persistent PostgreSQL data
- n8n application data
- external workflow dependencies
- Tailscale and LAN administration paths

All infrastructure-specific IP addresses, credentials, workflow identifiers and user data have been removed from this public case study.

## Symptoms

After restoring the machine from PBS, the application stack still showed problems.

Observed symptoms included:

- PostgreSQL container health remaining in a starting state,
- intermittent database connection timeout messages,
- recovery messages showing that database connectivity returned temporarily,
- telemetry-related errors,
- runner-related failures,
- workflow dependencies that were not reachable,
- n8n not behaving as expected even though the virtual machine itself was running.

One workflow also attempted an SSH connection to an unavailable dependency and returned a host-unreachable error.

The important observation was that the infrastructure restore did not eliminate the original application fault.

## Initial hypotheses

Possible causes included:

1. damaged virtual machine filesystem,
2. corrupted Docker installation,
3. PostgreSQL data corruption,
4. application configuration problems,
5. broken Docker networking,
6. unavailable external dependencies,
7. an unhealthy state already present inside the restored backup.

## First recovery attempt

A PBS restore was used because it provided a known recovery point of the virtual machine.

The restored VM booted successfully.

However, the same symptoms returned after recovery.

This indicated that PBS itself was not the root cause.

The backup had faithfully restored the machine state, including the application condition that already existed when the backup was created.

## Diagnostic conclusion

The failure domain had to be separated into layers:

- Proxmox / virtual machine layer,
- operating system layer,
- Docker layer,
- PostgreSQL layer,
- n8n application layer,
- external workflow dependencies.

Because the VM could boot and the restore itself completed, attention shifted away from the hypervisor and backup system.

The main objective became preserving application data while rebuilding only the runtime components that could be safely recreated.

## Data preservation strategy

Before rebuilding the stack, the recovery plan focused on preserving:

- PostgreSQL data,
- n8n application data,
- the n8n encryption key,
- configuration required to decrypt stored credentials,
- workflow definitions,
- relevant SQL and file backups.

The goal was to avoid destroying useful application state while replacing the container runtime layer.

## Rebuild strategy

Instead of repeatedly restoring the same VM state, the container stack was rebuilt.

The recovery approach was:

1. preserve PostgreSQL data,
2. preserve n8n application data,
3. preserve encryption-related configuration,
4. recreate Docker containers,
5. recreate Docker networking,
6. use a minimal Compose configuration,
7. reduce optional components during recovery,
8. disable unnecessary runners during initial validation,
9. bring PostgreSQL up first,
10. verify database readiness,
11. start n8n only after PostgreSQL was healthy.

## PostgreSQL verification

Database readiness was checked independently from n8n.

The objective was to confirm that PostgreSQL could accept connections before troubleshooting the application layer.

A healthy result from PostgreSQL demonstrated that the database service itself was available and that further failures had to be investigated higher in the stack.

## n8n verification

After rebuilding the stack, the n8n web interface was tested through the normal management path.

The application returned its HTML interface successfully.

This confirmed that:

- the VM networking was operational,
- the Docker stack was running,
- PostgreSQL was available,
- n8n could start,
- the HTTP service was reachable.

The final verification was performed from another host in the LAB rather than relying only on localhost testing.

## Important diagnostic distinction

One failed localhost test was initially misleading.

The test had been executed from a different host, so localhost referred to the machine running the test rather than the n8n VM.

Testing the actual n8n management address returned the expected application interface.

This reinforced an important troubleshooting principle:

always verify which network namespace and host context a diagnostic command is running in.

## External dependency failure

One workflow still produced an SSH host-unreachable error.

This did not indicate that n8n itself was broken.

It showed that the workflow depended on an external system that was unavailable.

The application platform and the workflow dependency therefore had to be treated as separate failure domains.

## Root cause

The recovery process showed that the PBS restore was functioning correctly.

The real issue was that the selected restore point already contained the problematic application state.

Restoring the VM reproduced that state exactly.

The effective recovery required rebuilding the container runtime while preserving persistent application data.

## Troubleshooting workflow

    n8n / PostgreSQL malfunction
              |
              v
    Restore VM from PBS
              |
              v
    Same symptoms return
              |
              v
    PBS restore verified
              |
              v
    Separate infrastructure from application state
              |
              v
    Preserve PostgreSQL and n8n data
              |
              v
    Rebuild Docker / Compose stack
              |
              v
    Start PostgreSQL first
              |
              v
    Verify database readiness
              |
              v
    Start n8n
              |
              v
    Verify HTTP interface
              |
              v
    Test workflow dependencies separately

## What this incident demonstrated

The incident required troubleshooting across several layers:

- virtualization,
- backup and restore,
- Linux,
- Docker,
- Docker Compose,
- PostgreSQL,
- n8n,
- networking,
- external workflow dependencies.

It also demonstrated that a backup system can work perfectly while still restoring an unhealthy application state.

## Preventive actions

The following controls reduce the risk of similar incidents:

- maintain more than one restore point,
- verify application health before considering a backup a known-good recovery point,
- test restores periodically,
- monitor PostgreSQL health independently from n8n,
- keep persistent data separated from disposable container runtime components,
- document encryption keys and recovery-critical configuration securely,
- verify workflow dependencies separately from the automation platform,
- perform post-restore functional tests rather than relying only on VM boot success.

## Lessons learned

### A successful restore does not guarantee a healthy application

PBS can restore a machine exactly as it existed.

If the machine was already in a faulty state, the restore may reproduce the same fault.

### Backup integrity and application health are different things

A backup can be technically correct while still containing an application problem.

Recovery planning therefore needs both infrastructure validation and application-level validation.

### Preserve data, rebuild disposable components

Containers and Docker networks can often be recreated safely.

Persistent data and encryption-related configuration require more careful handling.

### Verify dependencies independently

A workflow failure caused by an unavailable SSH target should not automatically be interpreted as an n8n platform failure.

### Test from the correct host context

localhost always refers to the machine where the command is executed.

Remote application tests must use the actual address of the target service.

## Result

The n8n environment was recovered by preserving the important persistent application data and rebuilding the container runtime instead of repeatedly restoring the same faulty state.

PostgreSQL readiness and the n8n web interface were verified independently.

The incident improved the recovery procedure by introducing an important rule:

a restore point should only be treated as a known-good recovery point after the application itself has been validated.
