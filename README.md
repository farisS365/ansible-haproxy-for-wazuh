# HAProxy & Keepalived Ansible Deployment Guide

## 📋 Overview

This Ansible playbook automates the deployment of HAProxy and Keepalived for high-availability load balancing of Wazuh servers.

## 🎯 Features

- ✅ Automated installation of HAProxy and Keepalived
- ✅ Configurable via single inventory file
- ✅ Automatic failover between MASTER and BACKUP
- ✅ Health checks for backend servers
- ✅ Firewall configuration (firewalld)
- ✅ Idempotent - safe to run multiple times

## 📦 Prerequisites

1. **Ansible installed** on control node:
```bash
   pip install ansible
   # or
   dnf install ansible
```

2. **SSH access** to both load balancer hosts with sudo privileges

3. **Two Linux hosts** (Rocky Linux, RHEL, or Debian-based)

## 🚀 Quick Start

### Step 1: Create Directory Structure
```bash
mkdir -p haproxy-keepalived-ansible/templates
cd haproxy-keepalived-ansible
```

### Step 2: Create All Files

Copy all 6 files from this package into the appropriate locations.

### Step 3: Update Configuration

Edit `inventory.ini` and update:

**Load Balancer Hosts:**
```ini
[loadbalancers]
haproxy1 ansible_host=192.168.237.168 keepalived_state=MASTER keepalived_priority=101 vrrp_interface=enp0s3
haproxy2 ansible_host=192.168.237.169 keepalived_state=BACKUP keepalived_priority=100 vrrp_interface=enp0s3
```

**Virtual IP Settings:**
```ini
virtual_ip=192.168.237.170          # Change to your VIP
virtual_ip_cidr=24
vrrp_interface=ens192                # Change to your interface name
vrrp_auth_pass=qwer1234             # Change this password!
```

**Wazuh Backend Servers:**
```ini
wazuh_master_ip=192.168.237.100
wazuh_worker1_ip=192.168.237.101
wazuh_dashboard1_ip=192.168.237.110
wazuh_dashboard2_ip=192.168.237.111
```

### Step 4: Test Connectivity
```bash
ansible loadbalancers -m ping
```

Expected output:
haproxy1 | SUCCESS => {"ping": "pong"}
haproxy2 | SUCCESS => {"ping": "pong"}
### Step 5: Deploy
```bash
# Dry run first (check mode)
ansible-playbook deploy_haproxy_keepalived.yml --check

# Deploy for real
ansible-playbook deploy_haproxy_keepalived.yml
```

## ✅ Verification

### Check Service Status
```bash
ansible loadbalancers -m shell -a "systemctl status haproxy keepalived"
```

### Check Keepalived Roles
```bash
ansible loadbalancers -m shell -a "journalctl -u keepalived -n 20 --no-pager | grep 'STATE'"
```

Expected output - one MASTER, one BACKUP:
haproxy1: (VI_1) Entering MASTER STATE
haproxy2: (VI_1) Entering BACKUP STATE

### Verify Virtual IP Assignment
```bash
ansible loadbalancers -m shell -a "ip addr show | grep {{ virtual_ip }}"
```

The VIP should only appear on the MASTER host.

### Test Load Balancing
```bash
# Test Wazuh Dashboard
curl -k https://<virtual_ip>:443

# Check HAProxy logs
ansible loadbalancers -m shell -a "journalctl -u haproxy -n 10 --no-pager"
```

## 🧪 Testing Failover

### Test 1: HAProxy Failure
```bash
# Stop HAProxy on MASTER
ansible haproxy1 -m systemd -a "name=haproxy state=stopped"

# Watch BACKUP become MASTER
ansible haproxy2 -m shell -a "journalctl -u keepalived -n 10 --no-pager"

# You should see: "(VI_1) Entering MASTER STATE"

# Verify VIP moved to new MASTER
ansible loadbalancers -m shell -a "ip addr show | grep {{ virtual_ip }}"

# Restart HAProxy
ansible haproxy1 -m systemd -a "name=haproxy state=started"
```

### Test 2: Keepalived Failure
```bash
# Stop Keepalived on MASTER
ansible haproxy1 -m systemd -a "name=keepalived state=stopped"

# Watch BACKUP become MASTER
ansible haproxy2 -m shell -a "journalctl -u keepalived -n 10 --no-pager"

# Restart Keepalived
ansible haproxy1 -m systemd -a "name=keepalived state=started"
```

### Test 3: Complete Node Failure
```bash
# Simulate complete failure by shutting down MASTER
ansible haproxy1 -m shell -a "systemctl stop haproxy keepalived"

# Verify BACKUP takes over immediately
ansible haproxy2 -m shell -a "systemctl status keepalived"

# Test that services still work via VIP
curl -k https://<virtual_ip>:443
```

## 🔧 Advanced Configuration

### Disable Preemption

To prevent automatic failback when original MASTER returns:
```ini
# In inventory.ini
[loadbalancers:vars]
keepalived_nopreempt=true
```

### Add More Wazuh Workers

Edit `templates/haproxy.cfg.j2`:
```jinja2
backend wazuh_agents
    mode tcp
    balance roundrobin
    option tcp-check
    server wazuh-master {{ wazuh_master_ip }}:{{ wazuh_agents_port }} check
    server wazuh-worker1 {{ wazuh_worker1_ip }}:{{ wazuh_agents_port }} check
    server wazuh-worker2 {{ wazuh_worker2_ip }}:{{ wazuh_agents_port }} check
```

Add variable to `inventory.ini`:
```ini
wazuh_worker2_ip=192.168.237.102
```

### Multiple Environments

Create separate inventory files:
```bash
# Production
ansible-playbook deploy_haproxy_keepalived.yml -i inventory-prod.ini

# Development
ansible-playbook deploy_haproxy_keepalived.yml -i inventory-dev.ini
```

## 🐛 Troubleshooting

### Issue: "BACKUP never becomes MASTER"

**Check:**
```bash
# 1. Verify priority settings
ansible loadbalancers -m shell -a "grep priority /etc/keepalived/keepalived.conf"

# 2. Check if MASTER is advertising
ansible haproxy1 -m shell -a "tcpdump -i {{ vrrp_interface }} vrrp -c 5"

# 3. Check firewall
ansible loadbalancers -m shell -a "firewall-cmd --list-all"
```

**Solution:** Ensure VRRP protocol is allowed in firewall.

### Issue: "Virtual IP not assigned"

**Check:**
```bash
# 1. Verify interface name
ansible loadbalancers -m shell -a "ip link show"

# 2. Check Keepalived logs
ansible loadbalancers -m shell -a "journalctl -u keepalived -n 50 --no-pager"

# 3. Verify kernel parameter
ansible loadbalancers -m shell -a "sysctl net.ipv4.ip_nonlocal_bind"
```

**Solution:** Update `vrrp_interface` in inventory.ini to match actual interface name.

### Issue: "HAProxy fails to start"

**Check:**
```bash
# 1. Test configuration
ansible loadbalancers -m shell -a "haproxy -c -f /etc/haproxy/haproxy.cfg"

# 2. Check logs
ansible loadbalancers -m shell -a "journalctl -u haproxy -n 50 --no-pager"

# 3. Verify backend servers are reachable
ansible haproxy1 -m shell -a "nc -zv {{ wazuh_master_ip }} {{ wazuh_agents_port }}"
```

**Solution:** Verify backend server IPs and ports are correct.

### Issue: "Split-brain - both nodes are MASTER"

**Check:**
```bash
# Check if both have VIP
ansible loadbalancers -m shell -a "ip addr show | grep {{ virtual_ip }}"

# Check network connectivity between nodes
ansible haproxy1 -m shell -a "ping -c 3 {{ hostvars['haproxy2'].ansible_host }}"
```

**Solution:** Ensure network connectivity between nodes and unique `vrrp_router_id`.

## 📊 Monitoring

### View Real-time Logs
```bash
# HAProxy logs
ansible loadbalancers -m shell -a "tail -f /var/log/haproxy.log"

# Keepalived logs
ansible loadbalancers -m shell -a "journalctl -u keepalived -f"
```

### Check Backend Server Status
```bash
ansible loadbalancers -m shell -a "echo 'show stat' | socat stdio /run/haproxy/admin.sock"
```

### Monitor Virtual IP
```bash
# Continuous monitoring
watch -n 1 'ansible loadbalancers -m shell -a "ip addr show | grep {{ virtual_ip }}" 2>/dev/null'
```

## 🔄 Maintenance Tasks

### Update Configuration Only
```bash
# Make changes to inventory.ini or templates, then:
ansible-playbook deploy_haproxy_keepalived.yml
```

### Restart Services
```bash
# Restart both services
ansible loadbalancers -m systemd -a "name=haproxy state=restarted"
ansible loadbalancers -m systemd -a "name=keepalived state=restarted"

# Restart one host at a time (zero downtime)
ansible haproxy2 -m systemd -a "name=haproxy state=restarted"
sleep 10
ansible haproxy1 -m systemd -a "name=haproxy state=restarted"
```

### Backup Configurations
```bash
ansible loadbalancers -m fetch -a "src=/etc/haproxy/haproxy.cfg dest=./backups/{{ inventory_hostname }}-haproxy.cfg flat=yes"
ansible loadbalancers -m fetch -a "src=/etc/keepalived/keepalived.conf dest=./backups/{{ inventory_hostname }}-keepalived.conf flat=yes"
```

### Rolling Updates
```bash
# Update BACKUP first
ansible-playbook deploy_haproxy_keepalived.yml --limit haproxy2

# Verify it works
sleep 30

# Update MASTER
ansible-playbook deploy_haproxy_keepalived.yml --limit haproxy1
```

## 🔐 Security Best Practices

1. **Change default passwords:**
   - Update `vrrp_auth_pass` in inventory.ini

2. **Use SSH keys:**
   - Configure `ansible_ssh_private_key_file` instead of passwords

3. **Restrict firewall rules:**
   - Only allow traffic from known sources
   - Limit management access

4. **Regular updates:**
```bash
   ansible loadbalancers -m package -a "name=* state=latest"
```

5. **Enable SELinux/AppArmor:**
```bash
   ansible loadbalancers -m shell -a "getenforce"
```

## 📚 Useful Commands
```bash
# Check Ansible version
ansible --version

# List all hosts
ansible loadbalancers --list-hosts

# Gather facts
ansible loadbalancers -m setup

# Run ad-hoc commands
ansible loadbalancers -a "uptime"

# Verbose output
ansible-playbook deploy_haproxy_keepalived.yml -vvv

# Syntax check
ansible-playbook deploy_haproxy_keepalived.yml --syntax-check

# Limit to specific host
ansible-playbook deploy_haproxy_keepalived.yml --limit haproxy1
```

## 📞 Support

### Log Locations

- HAProxy: `journalctl -u haproxy`
- Keepalived: `journalctl -u keepalived`
- System: `/var/log/messages` or `/var/log/syslog`

### Useful Resources

- HAProxy Documentation: https://www.haproxy.org/documentation.html
- Keepalived Documentation: https://www.keepalived.org/manpage.html
- Ansible Documentation: https://docs.ansible.com/

## 📝 Notes

- The playbook is **idempotent** - safe to run multiple times
- Configuration validation is performed before applying changes
- Services are automatically restarted when configuration changes
- Firewall rules are only applied on RedHat-based systems
- Original configurations are backed up before modification

## 🎓 Understanding the Setup

### How It Works

1. **HAProxy** load balances traffic across Wazuh servers
2. **Keepalived** manages the Virtual IP (VIP)
3. **MASTER** node holds the VIP and handles traffic
4. **BACKUP** node monitors MASTER and takes over if it fails
5. **Health checks** ensure only healthy backends receive traffic

### Priority Logic

- Base priority: Set in inventory (MASTER=101, BACKUP=100)
- When HAProxy is running: +2 weight
- MASTER effective priority: 101 + 2 = 103
- BACKUP effective priority: 100 + 2 = 102
- If MASTER HAProxy fails: 103 - 2 = 101 (lower than BACKUP's 102)
- BACKUP automatically becomes MASTER

### Preemption

- **Enabled by default**: Higher priority node becomes MASTER when available
- **Disabled with nopreempt**: MASTER stays MASTER until it fails

## ✨ What This Playbook Does

1. ✅ Installs HAProxy and Keepalived packages
2. ✅ Configures kernel parameters for IP binding
3. ✅ Deploys HAProxy configuration from template
4. ✅ Deploys Keepalived configuration from template
5. ✅ Validates HAProxy config before applying
6. ✅ Enables and starts both services
7. ✅ Configures firewall rules (RHEL/CentOS)
8. ✅ Creates backups of original configs
9. ✅ Handles service restarts via handlers

---

## 🎉 You're Ready!

Your high-availability load balancing setup is now automated and ready to deploy. Just update the IPs in `inventory.ini` and run the playbook!
```bash
ansible-playbook deploy_haproxy_keepalived.yml
```

Good luck! 🚀
