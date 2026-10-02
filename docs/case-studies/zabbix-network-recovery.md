# Case Study: Zabbix unavailable after system restart

## Summary

After a restart of the Zabbix server, the monitoring frontend was no longer reachable through its expected management address.

The server itself remained operational and remote SSH access through Tailscale was still available.

The investigation showed that the problem was not caused by Zabbix. The primary Ethernet interface had returned with a DHCP-assigned address instead of the expected persistent management address.

## Impact

- Zabbix frontend unavailable at the expected LAN address
- monitoring access disrupted
- dependent systems could no longer reach the server at its normal management address
- remote administration remained available through Tailscale

No application rebuild or backup restore was required.

## Environment

Relevant components:

- Linux
- Zabbix Server
- Zabbix Agent
- Ethernet LAN
- private management network
- Tailscale
- monitored Linux and Windows hosts

Infrastructure-specific addresses and host identifiers have been removed.

## Symptoms

After reboot:

- the normal Zabbix frontend address stopped responding,
- SSH through Tailscale still worked,
- the operating system was running,
- there was no immediate evidence that Zabbix itself had failed.

## Investigation

The investigation started at the network layer before making application changes.

The active Ethernet interface had received an unexpected DHCP address.

The persistent interface configuration contained:

    BOOTPROTO=dhcp

This explained why the server was operational but no longer reachable at the address expected by administrators and dependent services.

Because Tailscale provided an independent management path, the host could still be diagnosed remotely.

## Root Cause

The Ethernet interface was persistently configured for DHCP even though the Zabbix server was expected to retain a fixed management address.

After reboot, DHCP assigned a different address.

The application stack remained operational, but the server was no longer available at the expected LAN address.

## Resolution

The expected management address was restored.

The persistent network configuration was corrected so that the required address would survive future reboots.

## Validation

After the correction, the following areas were verified:

- SSH connectivity
- network interface configuration
- HTTP / Zabbix frontend
- Zabbix Agent
- Zabbix Server communication

The frontend became reachable again without rebuilding Zabbix.

## Troubleshooting Flow

    Zabbix frontend unavailable
              |
              v
    Verify independent remote access
              |
              v
    SSH through Tailscale works
              |
              v
    Inspect network interface
              |
              v
    Unexpected DHCP address detected
              |
              v
    Check persistent configuration
              |
              v
    DHCP configuration confirmed
              |
              v
    Restore expected management address
              |
              v
    Validate Zabbix services

## Preventive Actions

- maintain persistent addressing for infrastructure servers,
- document expected management interface configuration,
- validate networking after reboot,
- retain an independent remote administration path,
- monitor management interface availability.

## Lessons Learned

### Start at the lowest relevant layer

An unavailable application frontend does not necessarily indicate an application failure.

Networking, addressing, routing and listening services should be verified before changing application configuration.

### Independent management access reduces recovery time

Tailscale prevented the network configuration issue from becoming a complete remote lockout.

### Persistent configuration must be tested after reboot

A working live configuration is not sufficient if the persistent configuration differs from the intended state.

## Result

The Zabbix server was restored to normal operation without reinstalling the application or restoring the system from backup.

The failure domain was correctly identified as network configuration rather than the Zabbix application stack.
