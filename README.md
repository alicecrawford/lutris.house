# lutris.house kubernetes setup scripts

## overview

these are a set of ansible playbooks written to setup a kubernetes cluster.
it's intended to setup a cluster with a minimum of three nodes, but can
setup more nodes.  it will setup a dns server outside of the cluster for
name services for accessing services.

the following services are setup in the base cluster:

- cri: cri-o
- cni: cilium
- gateway api and mesh: istio
- csi: longhorn
- load balancing: metallb
- cert-manager for certificate management
- external-dns to setup dns entries for services
- cnpg and maria operator for databases (mariadb operator still todo)

other services

- prometheus operator for monitoring
- argocd
- forgejo
- pihole

nodes can be set up either to have control nodes run workloads or not.
if you run workloads on your control nodes they will need at least 6GiB
of RAM and a minimum of 3 nodes.  if you run exclusive control nodes
the control nodes need 4GiB of RAM, minimum of 3 control nodes and
at least one (ideally three) worker nodes.  my setup is having control
nodes run on virtual machines and the workloads are run on the bare metal
across three machines.

nodes are assumed to be recent fedora nodes.  that's what i use, so it's
what the script uses.  it should be easy enough to change to any other
systemd linux by using a different pacakge manager and adjusting their
package definitions (in `group_vars/all.yaml` and `roles/k8s_prereqs/vars/main.yaml`)
to fit your distro.  if you're not using systemd idk you'll probably have
to change a lot more, but i would suspect you're used to that.

## configuration

all nodes go in `inventory/hosts.yaml` and need to be assigned to one or
more of the following groups:

- `dns_servers`: these hosts run the dns server outside the cluster.
- `dns_primary`: only one host must be in this group for being the primary dns server.  it must also be in the `dns_servers` group.
- `k8s_nodes`: all kubernetes nodes must be in this group.
- `k8s_control_nodes`: these nodes are designated control plane nodes.  the must also be in the `k8s_nodes` group.
- `k8s_init_node`: only one host must be in this group for being the first control plane node to be set up.  it must be in both `k8s_nodes` and `k8s_control_nodes`.

each node also must have a file listing its ip address in `host_vars`.

all configuration options are set in `group_vars/all.yaml`.  see that file
for more info on its options.  any `roles/*/vars/*` files will be variables
used for that specific role but aren't really meant for cluster config

## operation

the plays are enumerated in their order to be run.  the first five playbooks
must be run in order to properly set up the cluster.  from there it's highly
recommended to install monitoring next, then the rest in their recommended
order.  the extra parameter `use_ip` will have the script use specified
ip addresses (see configuration for more info) for connection instead of
hostname; this is used in the pre-dns environment. (TODO: add another
extra parameter for username)

the arguments `-k` (ask for auth password) and `-K` (ask for become password)
are used in steps that need them.

### step 0: copy ssh keys

command:

`ansible-playbook -i inventory -k 00-add_ssh_key.yaml -e use_ip=true`

this will copy your ssh key to the host.  this needs to be run first and
any time your ssh key either changes or new hosts are added.  if run
after dns is set up exclude the `-e use_ip=true` parameter to use
hostnames instead of ip addresses

### step 1: setup dns servers

command (needs become):

`ansible-playbook -i inventory -K 01-dns_servers.yaml -e use_ip=true`

this will set up dns servers on every host in the group `dns_servers`.
one host must also be in the group `dns_primary`.  the `dns_primary`
host will be set up as... wait for it... the primary dns server for
your subdomain and configures it for an rfc2136 dynamic dns server.
all the other dns servers will be set up as secondaries.  all dns
hosts will run dnsdist for dns loadbalancing and keepalived to
make a vrrp virtual ip address to access the loadbalancer.

note: the dns servers can be any arbirtary servers in your inventory,
they do not need to be control plane nodes or anything like that.

another feature of this step is it will add every host's ipv4 address
to the zone file for the primary dns server.  eventually i do plan
on writing some sort of rfc2136 script to update each server's ip
addresses in full with the dns server but priorities.

theoretically this step can be optional if you have another dns server
handling the domain that can be updated with cert-manager and external-dns.
if that's the case you will need to edit those roles for update methods.

### step 2: prepare the k8s nodes

command (needs become):

`ansible-playbook -i inventory -K 02-prepare_nodes.yaml`

here we do the prerequesite things for setting up each node to run k8s.
installing packages, setting various system settings, enabling services,
etc.  yes i disable selinux and the firewall here.  i may eventually
figure out all the vairous steps of firewall rules and selinux roles to
get everything working with them enabled but i got bigger fish to fry.

this step needs to be repeated to join other nodes into the cluster

### step 3: initialize cluster nodes

command (needs become):

`ansible-playbook -i inventory -K 03-init_cluster.yaml`

this step does all the heavy lifting to initalize the cluster and join
all the nodes to it.  it first checks against the init node to see if
that node has been set up.  with that info it'll look to see what other
nodes have been set up, and it will only run the init steps for those
nodes.  the steps are as follows:

- init the entire cluster on `k8s_init_node`
- join all other control plane nodes to the cluster
- join all worker nodes to the cluster
- optionally remove control plane noschedule taint
- fix metric endpoints for system components
- copy the admin kubeconfig to the user homedir
- optionally remove the lb exclusion label

this step protects itself from trying to run the init or join commands
on any node that's already joined.  it can be used to add more nodes
to the cluster once it's up and running.

### step 4: install cluster services

command (needs become):

`ansible-playbook -i inventory -K 04-cluster_base.yaml`

this step sets up all the other base services for the cluster in this order:

- cilium
- metrics-server
- istio
- longhorn
- metallb
- cert-manager
- external-dns
- cnpg
- mariadb operator

### steps 5 onward

these steps install services that aren't really a base part of the cluster.
the only one that might need to be installed first is monitoring since some
services will rely on the prometheus operator to be installed first. right
now none of these need to become root, so they can all be run without the
`-K` argument.
