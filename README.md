# lutris.house kubernetes setup scripts

## config

### `group_vars/all.yaml`

- `extra_packages` - any additional packages to load on each machine durin init

### `group_vars/k8s_nodes.yaml`

this file contains cluster-level configuration.  cidr, cluster name, etc.

### `host_vars/*`

any host that would be configured before dns is set up should have a file here with
a field `ip_address` containing the ip of that host.  additionally `ansible_host`
should have a conditional to set it to either the ip or host name based on the `use_ip`
variable.

### `inventory/hosts.yaml`

main ansible inventory

## adding ssh keys to servers

the `add_ssh_key.yaml` will upload your public key to all the hosts in the inventory.
run with: 

```ansible-playbook -i inventory -k add_ssh_key.yaml -e use_ip=true```

omit the `-e use_ip=true` if dns is setup.  

## setting up dns servers

this requires all dns servers to be in a group `dns_servers` and the primary dns
server is the only host in the group `dns_primary`.  run with:

```ansible-playbook -i inventory -K dns_servers.yaml -e use_ip=true```

## setting up the cluster

this one's simple, just run the site.yaml playbook

```ansible-playbook -i inventory -K site.yaml```
