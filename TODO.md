## todo list

### implementations

- ~[p0] add cnpg configs~
- ~[p0] forgejo~
- [p1] mariadb operator
- [p2] immich
- [p3] romm
- [p3] nextcloud
- [p3] update dashboard/notifyer
- [p3] web based k8s dashboard
- [p4] add an oidc provider

### configurations

- [p1] monitoring for all services
- [p1] all relevant configs into gitops
  - deployments
  - statefulsets
  - configmaps
  - routes, gateways, certificates
  - clusterissuers
  - roles, bindings

### fixups

- ~[p0] make sure all cni traffic is encrypted~
- [p3] tighten up clusterrole for external dns
- [p3] make sure that keepalived and haproxy restarts only happen on change
