# Troubleshooting Log

Real issues hit while building this lab, recorded as they happened.

| # | Phase | Issue | Layer |
|---|---|---|---|
| 1 | 0 | SSH server fails to start on Ubuntu automation host | Host / service |
| 2 | 1 | R1 WAN interface gets no DHCP address | Layer 2 / topology |
| 3 | 1 | SW1 can reach R1 but not the internet | Layer 3 |
| 4 | 1 | Automation host root disk almost full | Host / storage |
| 5 | 1 | SSH from Ubuntu 22.04 to Cisco devices fails algorithm negotiation | Management plane |
| 6 | Z | New Zabbix node cannot reach its gateway | Layer 2 |
| 7 | Z | Zabbix refuses two templates on the same host | Monitoring config |
| 8 | Z | Zabbix server fails to start | Service / config |
| 9 | Z | Zabbix agent rejects the server | Monitoring / access control |

---

### Issue #1 - SSH server fails to start on Ubuntu automation host

- **Phase:** 0 (environment preparation)
- **Symptom:** `ssh.service` in failed state; ExecStartPre `/usr/sbin/sshd -t` exited with status 1.
- **Investigation:** Ran the config test manually: `sshd -t` returned "no hostkeys available -- exiting."
- **Root cause:** The prebuilt Ubuntu image had its SSH host keys removed (common practice when distributing VM templates, so clones don't share the same server identity).
- **Fix:** `ssh-keygen -A` to generate the missing host keys, then `systemctl restart ssh`; service is active (running).
- **Lesson learned:** When a systemd service fails, find the failing step in `systemctl status` and run that command manually to get the real error message.

---

### Issue #2 - R1 WAN interface gets no DHCP address

- **Phase:** 1 (on-prem setup)
- **Symptom:** Gi0/0 stayed "unassigned"; `renew dhcp GigabitEthernet0/0` returned "Not in Bound state".
- **Investigation:** `debug dhcp detail` showed three DHCPDISCOVER broadcasts and no DHCPOFFER. A static test address (192.168.134.10/24) could not ping the VMware NAT gateway (.2) or the EVE host (.128), so the problem was below DHCP - no Layer 2 path at all.
- **Root cause:** The "INTERNET" network object in EVE-NG was type `bridge` (an isolated, lab-internal bridge) instead of `Management(Cloud0)`.
- **Fix:** Changed the network type to Management(Cloud0). After restart R1 received 192.168.134.130 via DHCP and `ping 8.8.8.8` succeeded.
- **Lesson learned:** Separate the layers. A static IP plus ping test showed the fault was in Layer 2 connectivity, which pointed to the topology rather than the DHCP configuration.

---

### Issue #3 - SW1 can reach R1 but not the internet

- **Phase:** 1 (on-prem setup)
- **Symptom:** From SW1, `ping 192.168.10.1` succeeded but `ping 8.8.8.8` failed (0/5), even though `ip default-gateway 192.168.10.1` was configured.
- **Investigation:** `show ip route` on SW1 printed a full routing table with "Gateway of last resort is not set" instead of "Default gateway is 192.168.10.1" - the switch was routing.
- **Root cause:** The IOL L2 image has `ip routing` enabled by default. A device that routes ignores `ip default-gateway`, so SW1 had no route toward the internet.
- **Fix:** `no ip routing` - SW1 is an access switch and should not route; R1 is the only Layer 3 device in the LAN. `show ip route` then showed "Default gateway is 192.168.10.1" and `ping 8.8.8.8` succeeded (5/5).
- **Lesson learned:** `ip default-gateway` only applies when IP routing is disabled. `show ip route` tells you immediately which mode a switch is in.

---

### Issue #4 - Automation host root disk almost full

- **Phase:** 1 (on-prem setup)
- **Symptom:** Login banner on AUTO-HOST warned "/ is using 85.7% of 1.96GB"; `df -h /` showed only 258 MB free - not enough for Python tooling, backups and logs.
- **Investigation:** `lsblk` showed the whole virtual disk (`vda`) was only 2.2 GB, with the root partition `vda1` at 2.1 GB. The prebuilt Ubuntu cloud image is intentionally minimal.
- **Root cause:** Undersized base image disk, not a misconfiguration.
- **Fix:** Stopped the node; on the EVE-NG host located the node's overlay disk (lab ID from the `.unl` file, node ID 1) and ran `qemu-img resize virtioa.qcow2 10G`. The base image (backing file) stayed untouched. Inside the VM: `growpart /dev/vda 1` to extend the partition, then `resize2fs /dev/vda1` to extend the ext4 filesystem online. Result: 9.6 GB total, 7.9 GB free.
- **Lesson learned:** Growing a VM disk is three layers - virtual disk, partition, filesystem - and each needs its own step. Resizing only the overlay keeps the shared base image clean for other labs.

---

### Issue #5 - SSH from Ubuntu 22.04 to Cisco devices fails algorithm negotiation

- **Phase:** 1 (on-prem setup)
- **Symptom:** `ssh admin@192.168.10.1` (R1, IOSv 15.9): "no matching key exchange method found. Their offer: diffie-hellman-group-exchange-sha1, diffie-hellman-group14-sha1". After fixing that, `ssh admin@192.168.10.2` (SW1, IOL 15.1): "no matching cipher found. Their offer: aes128-cbc, 3des-cbc, aes192-cbc, aes256-cbc".
- **Root cause:** OpenSSH on Ubuntu 22.04 disables SHA1-based key exchange, `ssh-rsa` host keys and CBC ciphers by default. The older Cisco images only offer those. The two devices failed at different stages because the newer IOSv supports modern ciphers and the older IOL does not.
- **Fix:** A client-side `~/.ssh/config` block scoped to `Host 192.168.10.*` that re-enables `diffie-hellman-group14-sha1` (the stronger of the offered KEX methods), `ssh-rsa` host keys and `aes256-cbc` (the strongest offered cipher). Legacy algorithms are allowed only for the lab subnet; every other host keeps OpenSSH's strict defaults.
- **Lesson learned:** Read the "Their offer" list - it tells you exactly which side lacks what. The `+` syntax adds an algorithm without removing the modern ones, so SSH still picks the best common option per device.

---

### Issue #6 - New Zabbix node cannot reach its gateway

- **Phase:** Z (monitoring)
- **Symptom:** From the new ZABBIX node, `ping 192.168.10.1` returned "Destination Host Unreachable"; DNS resolution also failed.
- **Root cause:** Two things at once: SW1 Et0/3 was still administratively down (it had been shut as an unused port in Phase 1), and R1 itself was not running in EVE-NG.
- **Fix:** Configured Et0/3 as a host port with `no shutdown`, and started R1.
- **Lesson learned:** "Destination Host Unreachable" means no ARP reply - a Layer 2 problem: a port that is down, a cable, or the device on the other side not running. Check the obvious first.

---

### Issue #7 - Zabbix refuses two templates on the same host

- **Phase:** Z
- **Symptom:** Adding R1 with the templates "Cisco IOS by SNMP" and "ICMP Ping" failed: "Cannot inherit items with key icmpping of both templates, because the key must be unique on host R1."
- **Root cause:** The Cisco IOS template already includes ICMP ping items, so the ICMP Ping template would duplicate the same item keys.
- **Fix:** Used only "Cisco IOS by SNMP" on R1 and SW1; the ICMP Ping template stays only on PC1.
- **Lesson learned:** Item keys must be unique per host - check what a template already contains before stacking templates.

---

### Issue #8 - Zabbix server fails to start

- **Phase:** Z
- **Symptom:** `zabbix-server` stuck in "activating (auto-restart)" with exit code 1; no log file was created; the web interface showed "Zabbix server is not running".
- **Investigation:** `zabbix_server -c /etc/zabbix/zabbix_server.conf -T` (config test) reported: unknown parameter "Password" at line 129.
- **Root cause:** A malformed line (`Password=...` instead of `DBPassword=...`) left over from editing the database password, which also left the server without a valid DB password.
- **Fix:** Removed the bad line, added a correct `DBPassword=` line, aligned the database user's password with `ALTER USER`, validated with `-T` and restarted the service.
- **Lesson learned:** When a service dies before it can write its own log, test its configuration directly - `-T` pointed to the exact line.

---

### Issue #9 - Zabbix agent rejects the server

- **Phase:** Z
- **Symptom:** AUTO-HOST agent availability went from "Connection refused" (agent not running yet) to "Connection reset by peer"; Zabbix opened "Linux: Zabbix agent is not available".
- **Investigation:** `/var/log/zabbix/zabbix_agent2.log`: connection from "192.168.10.30" rejected, allowed hosts: "127.0.0.1".
- **Root cause:** The agent configuration edits had first been run on the wrong node (the Zabbix server itself), so AUTO-HOST still had the default `Server=127.0.0.1`. The edits were reverted on the server and applied on AUTO-HOST.
- **Fix:** Set `Server` and `ServerActive` to 192.168.10.30 and `Hostname=AUTO-HOST` on AUTO-HOST and restarted the agent; the Zabbix problem resolved automatically.
- **Lesson learned:** "Connection refused" and "connection reset" mean different things - nothing is listening vs. the listener is rejecting you. The agent log names the reason.
