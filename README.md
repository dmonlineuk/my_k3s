# my_k3s

Start with multiple VMs and scale out. It’s the better lab for learning the practices that matter at work—even though all nodes share one physical failure domain.

A strong first layout for your server:

| Role                     | Count | Suggested VM size                |
| ------------------------ | ----- | -------------------------------- |
| K3s control-plane + etcd | 3     | 2–4 vCPU, 8 GB RAM, SSD/NVMe     |
| Worker                   | 2     | 6–8 vCPU, 16–24 GB RAM, SSD/NVMe |
| HAProxy load balancers   | 2     | 1 vCPU, 1 GB RAM                 |

This is now as follows:

| Machine       | IP              | Setup                       |
| ------------- | --------------- | --------------------------- |
| k3s-cp-01     | 192.168.100.1   | 2 vCPU, 8 GB, 100 GB vDisk  |
| k3s-cp-02     | 192.168.100.2   | 2 vCPU, 8 GB, 100 GB vDisk  |
| k3s-cp-03     | 192.168.100.3   | 2 vCPU, 8 GB, 100 GB vDisk  |
| k3s-lb-01     | 192.168.100.98  | 1 vCPU, 2 GB, 20 GB vDisk   |
| k3s-lb-02     | 192.168.100.99  | 1 vCPU, 2 GB, 20 GB vDisk   |
| k3s-worker-01 | 192.168.100.101 | 6 vCPU, 16 GB, 100 GB vDisk |
| k3s-worker-02 | 192.168.100.102 | 6 vCPU, 16 GB, 100 GB vDisk |

The following update and hardening is done on each node once Ubuntu 24.04.4 LTS is installed:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install vim ufw -y
sudo ufw allow OpenSSH
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 8472/udp
sudo ufw allow 10250/tcp
sudo ufw enable
```

#### Restrict firewall rules by role

This applies the etcd and kubelet ports to every VM. Better practice will be to adjust these generalised settings to allow from specific IP addresses and to be set on specific nodes. As the next hardening pass, allow etcd only between control planes, API access only to appropriate internal sources, and retain Flannel/kubelet rules only where required.

For the load balancers k3s-lb-01 and k3s-lb-02 they will need to see each other so these will need adding:

```bash
# On k3s-lb-01
sudo ufw allow in on enp1s0 from 192.168.100.99 comment 'K3s LB VRRP peer'

# On k3s-lb-02
sudo ufw allow in on enp1s0 from 192.168.100.98 comment 'K3s LB VRRP peer'
```
### Installing K3s

The install is performed with this command: `curl -sfL https://get.k3s.io | sh -` but there are steps required to set up each box before installing.

#### Pin the K3s release

The stable installation channel changes over time, so a rebuild will not necessarily match this cluster. Record the exact release currently running and use INSTALL_K3S_VERSION for every server and agent installation: `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.36.2+k3s1" sh -`

It is important to understand this order of events:
1. initialise the first server; then
2. add a second server node; then
3. add a third server..

It is possible to continue adding more nodes to the control plane, but it is expected the total number always be an odd value. The reason is out of scope of this document, but just imagine that every odd numbered node adds quorum tolerance, but every even node adds additional cost and complexity.
## k3s-cp-01

Initialise this node first. The other control plane nodes will attach themselves to this to form a cluster.

1. Create a secret using `openssl rand -hex 32`
2. Create a directory using `sudo mkdir -p /etc/rancher/k3s/`
3. Create a file using `sudo vim /etc/rancher/k3s/config.yaml`:

```yaml
token: "PASTE_THE_GENERATED_TOKEN_HERE"
cluster-init: true

tls-san:
  - "192.168.100.100" # Future HAProxy/kube-vip address
  - "k3s-api" # Optional future DNS name

secrets-encryption: true

etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 14
```

4. Install k3s with `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.36.2+k3s1" sh -`
#### Back up snapshots off-host

These snapshots are stored locally on the control-plane VM and ultimately share the Unraid host failure domain. Add an off-host or remote-object-store backup destination and a documented restore test.
## k3s-cp-02, and k3s-cp-03

Attach additional control plane nodes in turn, following these steps.

1. Create a directory using `sudo mkdir -p /etc/rancher/k3s/`
2. Create a file using `sudo vim /etc/rancher/k3s/config.yaml`:

```yaml
token: "PASTE_THE_SAME_GENERATED_TOKEN_HERE"
server: "https://192.168.100.1:6443"

tls-san:
  - "192.168.100.100" # Future HAProxy/kube-vip address
  - "k3s-api" # Optional future DNS name

secrets-encryption: true

etcd-snapshot-schedule-cron: "0 */12 * * *"
etcd-snapshot-retention: 14
```

3. Install k3s with `curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.36.2+k3s1" sh -`
## k3s-lb-01, and k3s-lb-02

Although the two load balancers have IP addresses of 192.168.100.98 and 192.168.100.99, the virtual IP (VIP) will be set to 192.168.100.100. Note also that the subnet mask for the home network is 255.255.0.0 therefore the CIDR range is /16 rather than the usually expected value of /24.

1. Install HAProxy and KeepAliveD with `sudo apt install -y haproxy keepalived`
2. Update the configuration with `sudo vim /etc/haproxy/haproxy.cfg`:

```cfg
frontend k3s_api
    bind *:6443
    mode tcp
    option tcplog
    default_backend k3s_control_planes

backend k3s_control_planes
    mode tcp
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s
    server k3s-cp-01 192.168.100.1:6443 check
    server k3s-cp-02 192.168.100.2:6443 check
    server k3s-cp-03 192.168.100.3:6443 check
```

### k3s-lb-01

This is set as the MASTER, priority 200.

1. Create the config file using `sudo vim /etc/keepalived/keepalived.conf`:

```conf
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
    timeout 1
    fall 2
    rise 2
}

vrrp_instance K3S_API_VIP {
    state MASTER
    interface enp1s0
    virtual_router_id 51
    priority 200
    advert_int 1

    virtual_ipaddress {
        192.168.100.100/16
    }

    track_script {
        check_haproxy
    }
}
```

### k3s-lb-02

This is set as the BACKUP, priority 100.

1. Create the config file using `sudo vim /etc/keepalived/keepalived.conf`:

```conf
global_defs {
    enable_script_security
    script_user root
}

vrrp_script check_haproxy {
    script "/usr/bin/systemctl is-active --quiet haproxy"
    interval 2
    timeout 1
    fall 2
    rise 2
}

vrrp_instance K3S_API_VIP {
    state BACKUP
    interface enp1s0
    virtual_router_id 51
    priority 100
    advert_int 1

    virtual_ipaddress {
        192.168.100.100/16
    }

    track_script {
        check_haproxy
    }
}
```

### Start the service

On both load balancer machines, start the services with `sudo systemctl enable --now haproxy keepalived`

Test the service with this. Note it will return unauthorised, but this does prove the service is listening: `curl -k https://192.168.100.100:6443/readyz`

If there are issues:

1. Identify which load balancer machine has the virtual IP (VIP) address by running this:
		`ip -br address`
	- If both load balancers report that they have 192.168.100.100/16 then check the firewall rules allow these to see each other and if not, add the rules included previously
2. Check on each that the service is listening correctly using this:
		`sudo ss -ltnp | grep :6443`
	- If there is nothing returned, then the HAProxy configuration hasn't taken. Restart the service.

Use the following commands to check running services on the load balancers and restart as necessary:

```bash
sudo systemctl status keepalived
sudo systemctl status haproxy

sudo systemctl restart keepalived
sudo systemctl restart haproxy
```

## k3s-worker-01, and k3s-worker-02

Now join the worker nodes. This requires the authentication token. Get this by running the following on any of the 3 control plane machines we have set up: `sudo cat /var/lib/rancher/k3s/server/node-token`

Add the worker nodes one at a time by running this command on each in turn: 

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="v1.36.2+k3s1" \
  K3S_URL="https://192.168.100.100:6443" \
  K3S_TOKEN="PASTE_NODE_TOKEN_HERE" sh -
```

Label the nodes by running these commands on any control plane machine:

```bash
sudo k3s kubectl label node k3s-worker-01 node-role.kubernetes.io/worker=worker
sudo k3s kubectl label node k3s-worker-02 node-role.kubernetes.io/worker=worker
```

Check the shape of the setup with `sudo k3s kubectl get nodes` on any control plane machine. Here's an example output:

```stdout
NAME            STATUS   ROLES                AGE     VERSION
k3s-cp-01       Ready    control-plane,etcd   11h     v1.36.2+k3s1
k3s-cp-02       Ready    control-plane,etcd   11h     v1.36.2+k3s1
k3s-cp-03       Ready    control-plane,etcd   11h     v1.36.2+k3s1
k3s-worker-01   Ready    worker               23m     v1.36.2+k3s1
k3s-worker-02   Ready    worker               4m20s   v1.36.2+k3s1
```

## DHCP and Hostnames

These machines may be assigned static IP addresses, but this makes things murky for future troubleshooting, so the preference is static assignments to be made by our router's DHCP service. Also, using the Hosts feature to assign hostnames to the IP addresses helps with documenting the setup and giving us the ability to use memorable names for the nodes and the API VIP.

| Host IP address | Hostname      | Domain name | PTR |
| --------------- | ------------- | ----------- | --- |
| 192.168.100.1   | k3s-cp-01     | loc         | Yes |
| 192.168.100.2   | k3s-cp-02     | loc         | Yes |
| 192.168.100.3   | k3s-cp-03     | loc         | Yes |
| 192.168.100.98  | k3s-lb-01     | loc         | Yes |
| 192.168.100.99  | k3s-lb-02     | loc         | Yes |
| 192.168.100.100 | k3s-api       | loc         | Yes |
| 192.168.100.101 | k3s-worker-01 | loc         | Yes |
| 192.168.100.102 | k3s-worker-02 | loc         | Yes |
## Failover tests

Run these only after confirming all five Kubernetes nodes are `Ready`. Do not stop more than one control-plane node at a time.

### Load-balancer failover

First, identify the active load balancer. The VIP must be present on exactly one load balancer:

```bash
ip -br address show dev enp1s0
```

From a worker or another LAN machine, continuously test access through the VIP:

```bash
while true; do
  printf '%s ' "$(date -Is)"
  curl -sk --connect-timeout 1 --max-time 2 -o /dev/null -w '%{http_code}\n' \
    https://k3s-api:6443/readyz
  sleep 1
done
```

A `401` status is expected: it confirms that the protected Kubernetes API was reached. A connection failure appears as `000`.

On the currently active load balancer, stop HAProxy:

```bash
sudo systemctl stop haproxy
```

After a short interruption, confirm that `192.168.100.100/16` has moved to the other load balancer:

```bash
ip -br address show dev enp1s0
```

The repeating test should return to `401`. Restore HAProxy:

```bash
sudo systemctl start haproxy
```

Because `k3s-lb-01` has the higher priority, it should eventually reclaim the VIP. Confirm that only one load balancer owns it.

### Control-plane failover

From `k3s-cp-02` or `k3s-cp-03`, verify authenticated API access through the VIP:

```bash
sudo k3s kubectl --server=https://k3s-api:6443 get nodes
```

Stop K3s on one control-plane node only; for example, on `k3s-cp-01`:

```bash
sudo systemctl stop k3s
```

From another control-plane node, repeatedly run:

```bash
sudo k3s kubectl --server=https://k3s-api:6443 get nodes
```

The Kubernetes API should remain available because two control-plane/etcd members retain quorum. Restore the stopped node:

```bash
sudo systemctl start k3s
```

Wait until `k3s-cp-01` returns to `Ready` before performing any further failure test.

### Record

For each test, record:

- Which VM failed
    
- The time to regain API access
    
- Whether the VIP moved as expected
    
- Whether all nodes returned to `Ready`
    
- Any Keepalived, HAProxy, or K3s log messages