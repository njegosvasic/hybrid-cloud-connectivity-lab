# Monitoring Configuration (Phase Z - Zabbix)

Configuration added on top of Phase 1 to bring the on-prem network under Zabbix monitoring.

> Secrets are replaced with `<REDACTED-FOR-PUBLIC-REPO>`. `configs/R1.cfg` and `configs/SW1.cfg`
> still show the Phase 1 state; full running-configs including these changes will replace them
> once the monitoring phase is finished.

## New node: ZABBIX (Ubuntu 22.04)

| Item | Value |
|---|---|
| Address | 192.168.10.30/24, gateway 192.168.10.1 (netplan, same layout as AUTO-HOST) |
| Switch port | SW1 Et0/3 |
| Resources | 2 vCPU, 4 GB RAM, disk grown to 20 GB |
| Software | Zabbix 7.0 LTS server + frontend (Apache, PHP), MariaDB, Zabbix agent 2, fping, net-snmp tools |

### Installation summary

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-5+ubuntu22.04_all.deb
dpkg -i zabbix-release_7.0-5+ubuntu22.04_all.deb
apt update
apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf \
               zabbix-sql-scripts zabbix-agent2 mariadb-server fping snmp
```

```sql
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by '<REDACTED-FOR-PUBLIC-REPO>';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
```

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
mysql -uroot -e "set global log_bin_trust_function_creators = 0;"
# /etc/zabbix/zabbix_server.conf -> DBPassword=<REDACTED-FOR-PUBLIC-REPO>
zabbix_server -c /etc/zabbix/zabbix_server.conf -T      # validate config before starting
systemctl enable --now zabbix-server zabbix-agent2 apache2 mariadb
```

The default frontend account password was changed right after installation.

## SW1 - port for the Zabbix server

```
interface Ethernet0/3
 description ## Zabbix server ##
 switchport mode access
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown
```

## R1 and SW1 - SNMPv3 (read-only, restricted to the Zabbix server)

```
ip access-list standard ACL-SNMP
 remark Allow SNMP polling from Zabbix server
 permit host 192.168.10.30
!
snmp-server view ZBX-VIEW iso included
snmp-server group ZBX-GROUP v3 priv read ZBX-VIEW access ACL-SNMP
snmp-server user zabbix ZBX-GROUP v3 auth sha <REDACTED-FOR-PUBLIC-REPO> priv aes 128 <REDACTED-FOR-PUBLIC-REPO>
snmp-server location EVE-NG lab - on-prem site
snmp-server contact Njegos Vasic
snmp-server ifindex persist
```

The SNMPv3 user does not appear in `show running-config` by design; it is verified with
`show snmp user` and `show snmp group`.

Verified from the Zabbix server before adding hosts:

```
$ snmpget -v3 -l authPriv -u zabbix -a SHA -A <...> -x AES -X <...> 192.168.10.1 1.3.6.1.2.1.1.5.0
iso.3.6.1.2.1.1.5.0 = STRING: "R1.lab.local"
$ snmpget -v3 -l authPriv -u zabbix -a SHA -A <...> -x AES -X <...> 192.168.10.2 1.3.6.1.2.1.1.5.0
iso.3.6.1.2.1.1.5.0 = STRING: "SW1.lab.local"
```

## R1 - access to the Zabbix web interface (static PAT)

```
ip nat inside source static tcp 192.168.10.30 80 interface GigabitEthernet0/0 8080
```

The lab host sits in the VMware NAT network and only sees R1's WAN address, so the frontend is
reached at `http://<R1 WAN address>:8080/zabbix`.

## AUTO-HOST - Zabbix agent 2

`/etc/zabbix/zabbix_agent2.conf`:

```
Server=192.168.10.30
ServerActive=192.168.10.30
Hostname=AUTO-HOST
```

## Hosts in Zabbix

| Host | Interface | Template(s) |
|---|---|---|
| R1 | SNMPv3 authPriv (SHA1 / AES128), 192.168.10.1 | Cisco IOS by SNMP |
| SW1 | SNMPv3 authPriv (SHA1 / AES128), 192.168.10.2 | Cisco IOS by SNMP |
| AUTO-HOST | Agent, 192.168.10.20:10050 | Linux by Zabbix agent |
| Zabbix server | Agent, 127.0.0.1 | Default server templates |
| PC1 | 192.168.10.11 (ICMP only) | ICMP Ping |

"Cisco IOS by SNMP" already contains ICMP ping items, so the separate ICMP Ping template is used
only for PC1 (see Issue #7).
