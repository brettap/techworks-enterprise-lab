# SAM-017: Tailscale subnet access blocked by an access grant

| Field | Value |
| --- | --- |
| Document ID | SAM-017 |
| Title | Tailscale remote access to the lab network failed: access control, not routing |
| Owner / Author | Techworks |
| Status | Resolved for the Windows management workstation. Android client and external-network retests open |
| Last updated | 2026-11-25 |

**Search keywords:** `Tailscale`, `subnet router`, `subnet route approved`, `access grant`, `ACL`, `tag:admin`, `autogroup:member`, `tailscale ping`, `tailscale netcheck`, `DERP`, `Get-NetRoute`, `Test-NetConnection`, `100.100.100.100`

Placeholders: `<devops-server>` (the Ubuntu DevOps server, Tailscale subnet router), `<devops-tailscale-ip>`, `<mgmt-tailscale-ip>`, `<network>` (the lab subnet), `<dc01-address>`. Real values never go in this document or the repository.

## Environment

- Incident Date: November 23–24, 2026
- Operating System: Ubuntu Server and Windows 11
- Application/System: Tailscale
- Network Environment: Techworks Network
- Hypervisor: Proxmox VE
- Firewall: Netgate 2100 (pfSense)
- Tailscale Server: the Ubuntu DevOps server (`<devops-server>`), acting as subnet router
- Management Workstation: TWMGMT
- Tailscale Server IP: `<devops-tailscale-ip>`
- Management Workstation Tailscale IP: `<mgmt-tailscale-ip>`
- Internal Network: `<network>/24`
- Active Directory Domain: techworks.local
- Domain controller tested: DC-01 (`<dc01-address>`)

## Problem

Tailscale remote access to internal network resources
stopped functioning from the Windows management
workstation.

The management workstation had previously been able
to access internal resources remotely using Tailscale
subnet routing.

Connectivity failures occurred when attempting to
access the network through external Wi-Fi networks
and a mobile hotspot.

The objective was to identify the cause of the
connectivity failure and restore remote access
without unnecessarily modifying the existing
network infrastructure.

## Symptoms

- Previously functional Tailscale remote access failed.
- Internal network resources were inaccessible remotely.
- Intermittent Tailscale peer connectivity was observed.
- Standard ping tests experienced packet loss.
- Connectivity failed over external Wi-Fi networks.
- Connectivity failed over a mobile hotspot.
- Android Tailscale client appeared offline.
- Tailscale service remained active on the DevOps server.
- Subnet routes were approved but access was unsuccessful.

## Troubleshooting Steps

1. Verified the Tailscale daemon was running on
   the DevOps server.

2. Reviewed Tailscale device status and confirmed
   the server's assigned Tailscale IP address.

3. Tested peer-to-peer connectivity between the
   Windows management workstation and the DevOps server.

4. Executed Tailscale network diagnostics to
   investigate UDP connectivity, NAT traversal,
   and DERP relay availability.

5. Reviewed the Netgate 2100 firewall configuration
   for potential network connectivity issues.

6. Verified that the internal network subnet route
   was approved within the Tailscale administration
   console.

7. Reviewed the Tailscale access control policy
   and identified a restriction associated with
   the tag:admin source selector.

8. Modified the applicable access grant to use
   autogroup:member.

9. Verified that the Windows routing table contained
   the advertised internal network subnet.

10. Tested TCP connectivity to DC-01
    using port 53 (DNS).

11. Confirmed successful TCP connectivity through
    the Tailscale network interface.

### Commands Used

#### Verify Tailscale Service Status

```bash
sudo systemctl status tailscaled
```

Purpose:
Confirm that the Tailscale daemon is active and
running on the Ubuntu server.

#### Verify Tailscale Device Status

```bash
tailscale status
```

Purpose:
Review connected devices, assigned Tailscale IP
addresses, and peer connectivity status.

#### Verify Tailscale IP Address

```bash
tailscale ip -4
```

Purpose:
Identify the IPv4 address assigned to the
Tailscale network interface.

#### Test Tailscale Peer Connectivity

```powershell
tailscale ping <devops-tailscale-ip>
```

Purpose:
Determine whether the management workstation can
communicate with the Ubuntu Tailscale peer.

#### Perform Tailscale Network Diagnostics

```powershell
tailscale netcheck
```

Purpose:
Evaluate UDP connectivity, NAT traversal
capabilities, and DERP relay availability.

#### Verify Windows Subnet Route

```powershell
Get-NetRoute -DestinationPrefix "<network>/24"
```

Output:

```text
ifIndex           : 39
DestinationPrefix : <network>/24
NextHop           : 100.100.100.100
RouteMetric       : 0
```

Result:
The Windows management workstation contained a
route to the internal network through Tailscale.

#### Test Internal Resource Connectivity

```powershell
Test-NetConnection <dc01-address> -Port 53
```

Output:

```text
ComputerName     : <dc01-address>
RemoteAddress    : <dc01-address>
RemotePort       : 53
InterfaceAlias   : Tailscale
SourceAddress    : <mgmt-tailscale-ip>
TcpTestSucceeded : True
```

Result:
Successful TCP connectivity to DC-01
through the Tailscale interface.

## Root Cause

The Tailscale access control policy restricted
the applicable access grant to devices associated
with the tag:admin source selector.

The management workstation did not satisfy the
applicable source authorization requirement.

Although the subnet route was available, the
access control policy prevented the intended
communication with internal network resources.

The issue was associated with authorization
rather than the availability of the subnet route.

## Resolution

Modified the applicable Tailscale access grant
within the Tailscale administration console.

Previous source selector:

```json
"src": ["tag:admin"]
```

Updated source selector:

```json
"src": ["autogroup:member"]
```

This modification allowed eligible tailnet members
to match the applicable access grant.

The existing subnet route remained in place.

No additional pfSense firewall rule changes were
identified as necessary to resolve the confirmed
connectivity issue.

## Validation

| Validation Test | Result |
|---|---|
| Tailscale daemon status | Passed |
| Tailscale subnet route approval | Passed |
| Windows subnet route installation | Passed |
| TCP connectivity to DC01 | Passed |
| Traffic using Tailscale interface | Passed |
| DNS TCP port 53 connectivity | Passed |
| DNS query resolution | Not independently verified |
| External Wi-Fi retest | Not documented |
| Mobile hotspot retest | Not documented |
| Android client connectivity | Pending |

The successful TCP connectivity test confirmed
that the Windows management workstation could
reach DC-01 through
the Tailscale subnet route.

## Security Considerations

Changing the source selector from tag:admin to
autogroup:member potentially broadens access
to the resources covered by the grant.

Review the complete access grant, including
destination networks and permitted protocols
and ports, to ensure compliance with the
principle of least privilege.

Consider implementing a dedicated administrative
group if internal infrastructure access should
be restricted to authorized administrators.

## Lessons Learned

- Subnet route approval does not automatically
  grant authorization to access subnet resources.

- Routing and access control must be evaluated
  independently during network troubleshooting.

- A running VPN service does not guarantee
  end-to-end application connectivity.

- Successful Tailscale peer connectivity does
  not necessarily confirm subnet resource access.

- ICMP connectivity tests alone are insufficient
  for determining application availability.

- TCP port testing provides more specific
  evidence of service-level connectivity.

- Windows routing table inspection can confirm
  whether the advertised subnet route is installed.

- Access control policies should be reviewed
  before making unnecessary firewall changes.

- Troubleshooting should distinguish between
  control plane, data plane, routing, and
  authorization failures.

## Preventive Actions

- Document Tailscale access control policy changes.
- Review source and destination authorization.
- Maintain least-privilege access grants.
- Validate subnet routes after policy changes.
- Test application-specific TCP ports.
- Perform remote connectivity tests periodically.
- Validate connectivity from external networks.
- Complete Android client troubleshooting.

## Incident Status

Status: Resolved - Windows subnet connectivity

Root Cause Category: Access Control / Authorization

Affected Component: Tailscale Access Grants

Corrective Action: Access Control Policy Modification

Remaining Actions: External network validation
and Android client connectivity testing.

## Command Reference

| Task | Command | Where |
| --- | --- | --- |
| Is the Tailscale daemon running? | `sudo systemctl status tailscaled` | Subnet router (Linux) |
| List tailnet devices and their state | `tailscale status` | Any Tailscale device |
| Show this device's Tailscale IPv4 address | `tailscale ip -4` | Any Tailscale device |
| Test the Tailscale path to a peer | `tailscale ping <peer-tailscale-ip>` | Any Tailscale device |
| Check UDP, NAT traversal and DERP relays | `tailscale netcheck` | Any Tailscale device |
| Is the subnet route installed on Windows? | `Get-NetRoute -DestinationPrefix "<network>/24"` | Windows client |
| Test a TCP port on an internal host | `Test-NetConnection <address> -Port <port>` | Windows client |

`100.100.100.100` is Tailscale's own internal address (its built-in DNS resolver), not a lab address.

## Document History

| Date | Change | By |
| --- | --- | --- |
| 2026-11-24 | Incident written up | Techworks |
| 2026-11-25 | Numbered SAM-017; addresses and server name replaced with placeholders for the public repository; command reference added | Techworks |
