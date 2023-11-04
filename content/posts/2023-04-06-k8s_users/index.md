---
title: Create an user in Kubernetes 
date: 2023-04-06T08:39:00+01:00
tags: 
    - kubernetes
    
---

# Kubernetes authentication - RBAC in 3 lines
Permissions are granted to `Roles` or `ClusterRoles`. A `Roles` always sets permissions within a particular namespace. 
Authentication is managed by multiple providers; the easiest one is x509 certificates.

## x509 - Create an "user"
An user is just a certificate with a CN or a group, in the Other attribute. 
Create and sign a certificate, from a master node
```shell
cd /etc/kubernetes/users
username="yye4916"
groupname="grp_yye4916"
openssl genrsa -out ${username}.key 2048
openssl req -new -key ${username}.key -out ${username}.csr -subj "/CN=${username} /O=${groupname}"
openssl x509 -req -in ${username}.csr -CA ../pki/ca.crt -CAkey ../pki/ca.key -CAcreateserial -out ${username}.crt -days 3540
```

## Create the user .kubeconfig
Copy the existing kubeconifg, then change the `.kubeconfig` as follows (note: this is meta-code, don't att bash commands in a yaml file - it won't work):
```yaml
users:
- name: ${username}
  user:
    client-certificate-data: $(cat ${username}.crt | base64)
    client-key-data: $(cat ${username}.key | base64)
```


## Bonus: assign a role to the user
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: test-cloudenabler-yye4916
  namespace: ce-infra
subjects:
  - kind: Group
    apiGroup: rbac.authorization.k8s.io
    name: grp_yye4916
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: team-role

```