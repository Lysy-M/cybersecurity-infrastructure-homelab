# Case Study: Zabbix unavailable after system restart

## Incident summary

After a restart of the Zabbix server, the monitoring frontend was no longer reachable through its expected management address.

The server itself remained operational and remote administrative access through Tailscale was still available.

The root cause was an incorrect persistent network configuration: the primary Ethernet interface used DHCP instead of the expected fixed management address.

## Environment

Relevant components:

- Zabbix Server
- Linux operating system
- Ethernet LAN interface
- private management network
- Tailscale
- monitored Linux and Windows hosts

All infrastructure-specific IP addresses and host identifiers have been removed from this public case study.

## Symptoms

After the restart:

- the normal Zabbix frontend address stopped responding,
- SSH access through Tailscale still worked,
- the operating system was running,
- the application itself was not initially confirmed as the source of the outage.

## Initial hypotheses

Possible causes included:

1. Zabbix service failure
2. web server failure
3. firewall filtering
4. routing problems
5. interface configuration change
6. loss of the expected management IP address

## Diagnostics

The investigation started with the network layer.

The active Ethernet interface had received a different address from DHCP.

The persistent interface configuration showed:

    BOOTPROTO=dhcp

This explained why the server was no longer reachable at the expected management address after reboot.

Because Tailscale used an independent management path, SSH access to the server remained available and troubleshooting could continue remotely.

## Root cause

The Ethernet interface was persistently configured for DHCP even though the infrastructure expected the Zabbix server to retain a fixed management address.

After reboot, DHCP assigned a different address.

The host and Zabbix services were operational, but administrators and dependent systems were still attempting to communicate with the previous management address.

## Resolution

The expected management address was restored and connectivity was verified.

The following areas were checked:

- SSH connectivity
- HTTP / Zabbix frontend
- Zabbix agent
- Zabbix server communication
- network interface state

The application stack itself was operational.

The actual failure domain was the persistent network configuration.

## Troubleshooting workflow

    Zabbix frontend unavailable
              |
              v
    Verify independent remote access
              |
              v
    SSH through Tailscale operational
              |
              v
    Inspect network interfaces
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
    Verify Zabbix services
              |
              v
    Service restored

## What this incident demonstrated

The incident required troubleshooting across several layers:

- Linux networking
- IP addressing
- remote administration
- service availability
- Zabbix communication
- dependency analysis

It also demonstrated the value of maintaining an independent management path.

Tailscale allowed the server to be diagnosed remotely even though its normal LAN management address had changed.

## Preventive actions

- use persistent addressing for infrastructure servers,
- verify interface configuration after system changes,
- maintain an independent remote administration path,
- monitor management interface availability,
- document expected interface configuration,
- perform network and service validation after reboot.

## Lessons learned

### Frontend unavailable does not necessarily mean application failure

The application can remain healthy while the host is unavailable at the address expected by administrators and dependent services.

### Start troubleshooting from the lowest relevant layer

Before changing Zabbix configuration, verify:

- interface state,
- IP addressing,
- routing,
- listening ports,
- service status.

### Independent remote access is valuable

Tailscale provided an alternative path to the server and prevented the network configuration problem from becoming a complete remote lockout.

### Reboot testing matters

A live configuration may work correctly while the persistent configuration contains an error that only becomes visible after restart.

## Result

The Zabbix server was restored without rebuilding the application or restoring the virtual machine from backup.

The investigation correctly identified the failure domain as network configuration rather than the Zabbix application stack.
