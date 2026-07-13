# Reusable Lab Configurations

These files preserve the working detection snippets from the lab. They are examples, not complete production configurations.

Merge each snippet into the existing configuration at its target location. Back up the current file first. Do not replace a full Wazuh or auditd configuration with one of these snippets.

## File placement

| Repository file | Host | Target and placement |
| --- | --- | --- |
| `wazuh/agent-fim.xml` | Wazuh agent | Merge the `<directories>` line inside the existing `<syscheck>` block in `/var/ossec/etc/ossec.conf`. |
| `wazuh/agent-audit-log.xml` | Wazuh agent | Merge the `<localfile>` block inside `<ossec_config>` in `/var/ossec/etc/ossec.conf`. |
| `wazuh/local_rules.xml` | Wazuh manager | Add the `<group>` block to `/var/ossec/etc/rules/local_rules.xml`. Preserve any rules already there. |
| `auditd/t1082-hostname.rules` | Monitored Linux host | Install as `/etc/audit/rules.d/t1082-hostname.rules`, then load the audit rules. |

## Lab-only caveats

Real-time FIM on `/tmp` can create high event volume on busy systems. Measure resource use and alert volume before adopting it outside a lab.

Rule 100100 alerts whenever decoded audit data shows `hostname` as the first argument. The command is often legitimate, so this lab correlation rule may need added context or tuning in production.

The audit rule targets the 64-bit syscall architecture and `/usr/bin/hostname`. Confirm both on the monitored host. Add an appropriate 32-bit rule if the host supports 32-bit programs.

Audit data can be sensitive and can consume disk space quickly. Apply access controls, log rotation, retention limits, and change management for production use.

## Apply and validate

> **⚠️ Not a blind paste.** The two agent fragments (`agent-fim.xml` and `agent-audit-log.xml`) must be merged **by hand** into the existing `<syscheck>` and `<ossec_config>` sections of `/var/ossec/etc/ossec.conf` — see the placement table above. Only the auditd rule file and the manager's rule block are true drop-ins. Back up each file before editing.

Install and start auditd on the monitored Ubuntu host if it is not present:

```bash
sudo apt install auditd audispd-plugins -y
sudo systemctl enable --now auditd
```

Confirm the executable path before installing the audit rule:

```bash
command -v hostname
uname -m
```

Install the drop-in audit rule as its own file. `cp` is idempotent — re-running it overwrites the file instead of appending duplicate rules the way `tee -a` on `audit.rules` would:

```bash
sudo cp configs/auditd/t1082-hostname.rules /etc/audit/rules.d/t1082-hostname.rules
```

Load the rule and verify that auditd accepted it:

```bash
sudo augenrules --load
sudo auditctl -l | grep t1082_recon
```

After merging both agent snippets, restart the Wazuh agent and verify that FIM and audit-log collection started:

```bash
sudo systemctl restart wazuh-agent
sudo grep -E "Directory set for real time monitoring: '/tmp'|Analyzing file: '/var/log/audit/audit.log'" /var/ossec/logs/ossec.log
```

On the Wazuh manager, test the merged ruleset before restarting the service:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
```

Run the command on the monitored host, then confirm that the kernel audit layer recorded it:

```bash
/usr/bin/hostname
sudo ausearch -k t1082_recon -i
```

Use `sudo /var/ossec/bin/wazuh-logtest` on the manager with a raw audit event to inspect decoding and matching. Rule 100100 should win after built-in rule 80700 groups the event.

For FIM, create and remove a harmless test file in `/tmp`, then check the endpoint FIM view or filter Wazuh alerts on the `syscheck` rule group.

```bash
touch /tmp/wazuh-fim-validation
rm /tmp/wazuh-fim-validation
```

Validate behavior in the lab before promoting any snippet. Keep screenshots or alert exports with the test time, host, rule ID, and expected result as evidence.
