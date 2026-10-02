# Case Study: n8n / PostgreSQL recovery after PBS restore reproduced the same faulty state

## Summary

A virtual machine hosting n8n and PostgreSQL was restored from Proxmox Backup Server.

The restore completed successfully and the VM booted, but the original application problems returned.

The key finding was that the backup system had worked correctly: the selected restore point already contained the unhealthy application state.

Recovery therefore required separating persistent application data from disposable runtime components and rebuilding the container stack.

## Impact

- n8n was not operating normally after restore
- PostgreSQL health and connectivity were inconsistent during recovery
- some workflow dependencies were unavailable
- restoring the VM alone did not resolve the incident

Persistent application data had to be preserved while the runtime layer was rebuilt.

## Environment

Relevant components:

- Proxmox VE
- Proxmox Backup Server
- Linux virtual machine
- Docker
- Docker Compose
- n8n
- PostgreSQL
- persistent application data
- external workflow dependencies
- LAN and Tailscale administration paths

Infrastructure-specific addresses, credentials and workflow identifiers have been removed.

## Symptoms

After restoring the VM from PBS:

- the virtual machine booted,
- PostgreSQL did not immediately reach a stable healthy state,
- database connection timeout messages appeared,
- n8n still showed application-level problems,
- some optional components reported errors,
- an external SSH dependency used by a workflow was unreachable.

The infrastructure restore itself had completed successfully.

## Investigation

The incident was separated into distinct failure domains:

- hypervisor and VM,
- operating system,
- Docker runtime,
- PostgreSQL,
- n8n,
- external workflow dependencies.

Because the VM booted correctly after restore, the investigation moved away from the Proxmox and PBS layer.

PostgreSQL readiness was checked independently from n8n.

The external SSH failure was also treated separately because an unavailable workflow target does not prove that the n8n platform itself is unhealthy.

One localhost test was also identified as misleading because it had been executed from a different host. The test was repeated against the actual n8n service address.

## Root Cause

The PBS restore was functioning correctly.

The selected backup contained the same problematic application state that existed before the restore.

Restoring the VM reproduced that state exactly.

The effective recovery therefore required rebuilding the Docker runtime while preserving the persistent n8n and PostgreSQL data.

## Data Preservation

Before rebuilding the stack, the recovery process preserved:

- PostgreSQL data,
- n8n application data,
- encryption-related configuration required by n8n,
- workflow definitions,
- recovery-relevant configuration.

Disposable container runtime components could then be recreated without discarding persistent application data.

## Resolution

The container stack was rebuilt instead of repeatedly restoring the same VM state.

Recovery sequence:

1. preserve PostgreSQL data,
2. preserve n8n application data,
3. preserve encryption-related configuration,
4. recreate Docker containers,
5. recreate Docker networking,
6. simplify the Compose stack for recovery,
7. start PostgreSQL first,
8. verify database readiness,
9. start n8n,
10. test the web interface,
11. test external workflow dependencies separately.

## Validation

Validation was performed at several layers.

PostgreSQL was checked independently to confirm that it could accept connections.

The n8n web interface was then tested through its actual management address.

Successful HTTP access confirmed that:

- the VM networking was operational,
- Docker was running,
- PostgreSQL was available,
- n8n had started,
- the application frontend was reachable.

External workflow dependencies were validated separately from the n8n platform.

## Troubleshooting Flow

    n8n / PostgreSQL malfunction
              |
              v
    Restore VM from PBS
              |
              v
    Same symptoms return
              |
              v
    Verify VM and PBS restore
              |
              v
    Separate application from infrastructure
              |
              v
    Preserve persistent data
              |
              v
    Rebuild Docker / Compose runtime
              |
              v
    Verify PostgreSQL
              |
              v
    Start and verify n8n
              |
              v
    Test external dependencies separately

## Preventive Actions

- maintain multiple restore points,
- identify known-good restore points using application-level health checks,
- periodically test restores,
- monitor PostgreSQL independently from n8n,
- keep persistent data separate from disposable containers,
- protect recovery-critical encryption configuration,
- validate external workflow dependencies independently,
- perform functional application testing after every restore.

## Lessons Learned

### Successful restore does not guarantee healthy application state

A backup system can restore a VM perfectly while reproducing an application problem that was already present at backup time.

### Backup integrity and application health are separate concerns

VM boot success is not sufficient recovery validation.

The database, application and important workflows must also be tested.

### Preserve state, rebuild disposable runtime components

Persistent PostgreSQL and n8n data required protection.

Containers and Docker networking could be recreated safely.

### Separate platform failures from dependency failures

An unreachable SSH target used by a workflow is a different failure domain from the n8n platform itself.

### Test from the correct network context

localhost refers to the machine executing the command.

Remote validation must target the real application endpoint.

## Result

The n8n environment was recovered by preserving persistent application data and rebuilding the container runtime rather than repeatedly restoring the same faulty VM state.

The incident led to a stronger recovery principle:

a restore point is only considered known-good after both infrastructure and application-level validation succeed.
