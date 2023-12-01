# awx
DCSR's AWX server



> :warning: **NOTE**: Starting in version 18.0, the AWX Operator is the preferred way to install AWX. 


## Hardware Specs

> TO COMPLETE


## Network Prerequisites

Ensure that your AWX host is authorized to communicate with managed nodes through the following network protocols :

`TCP/22 (SSH)`

`TCP/5986 (WinRM)`

Check with the network team if some firewall rules needs to be configured to allow the communications.


## Prepare the host

Create an "ansibleadmin" user which will be used to install AWX and connect to the remote ansible nodes

```bash
adduser ansibleadmin
usermod -aG wheel ansibleadmin
```

Connect as 'ansibleadmin'

```bash
su - ansibleadmin
```

Disable SElinux by editing the configuration file `/etc/selinux/config`

```bash
sudo sed -i 's/^SELINUX=.*$/SELINUX=permissive/' /etc/selinux/config
sudo setenforce 0
```

Disable firewalld

```bash
sudo systemctl stop firewalld && sudo systemctl disable firewalld
```

Install the required packages to deploy AWX Operator and AWX

```bash
sudo dnf install -y git
```


## Install Kubernetes (K3s) 

To deploy AWX Operator, a Kubernetes cluster is needed. 

```bash
curl -sfL https://get.k3s.io | sudo bash -
```

Make the config file `/etc/rancher/k3s/k3s.yaml` readable by a non-root user

```bash
sudo chmod 644 /etc/rancher/k3s/k3s.yaml
```


## Install AWX Operator 

Create a file called `kustomization.yaml` with the following content :

```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=<tag>

# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: <tag>

# Specify a custom namespace in which to install AWX
namespace: awx
```

Replace the `<tag>` tag with the last version of AWX operator. (currently 2.5.3 at the time of this writing)

Install the manifests by running this :

```bash
kubectl apply -k .
```

The output should look like :
```
namespace/awx created
customresourcedefinition.apiextensions.k8s.io/awxbackups.awx.ansible.com created
customresourcedefinition.apiextensions.k8s.io/awxrestores.awx.ansible.com created
customresourcedefinition.apiextensions.k8s.io/awxs.awx.ansible.com created
serviceaccount/awx-operator-controller-manager created
role.rbac.authorization.k8s.io/awx-operator-awx-manager-role created
role.rbac.authorization.k8s.io/awx-operator-leader-election-role created
clusterrole.rbac.authorization.k8s.io/awx-operator-metrics-reader created
clusterrole.rbac.authorization.k8s.io/awx-operator-proxy-role created
rolebinding.rbac.authorization.k8s.io/awx-operator-awx-manager-rolebinding created
rolebinding.rbac.authorization.k8s.io/awx-operator-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/awx-operator-proxy-rolebinding created
configmap/awx-operator-awx-manager-config created
service/awx-operator-controller-manager-metrics-service created
deployment.apps/awx-operator-controller-manager created
```

The `awx-operator` should be running :

```bash
kubectl get pods -n awx
```

The output should look like :
```
NAME                                               READY   STATUS    RESTARTS   AGE
awx-operator-controller-manager-66ccd8f997-rhd4z   2/2     Running   0          11s
```

Modify the current namespace for `kubectl` :

```bash
kubectl config set-context --current --namespace=awx
```

Create a file named `awx.yaml` with the following content :

```bash
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
spec:
  service_type: nodeport
  nodeport_port: 30000
```

Add the line `- awx.yaml` to the `kustomization.yaml` file as identical as below :
```bash
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  # Find the latest tag here: https://github.com/ansible/awx-operator/releases
  - github.com/ansible/awx-operator/config/default?ref=<tag>
  - awx.yaml
  
# Set the image tags to match the git version from above
images:
  - name: quay.io/ansible/awx-operator
    newTag: <tag>

# Specify a custom namespace in which to install AWX
namespace: awx
```

Create the AWX instance in your cluster :

```bash
kubectl apply -k .
```

The output should look like :

```
namespace/awx unchanged
customresourcedefinition.apiextensions.k8s.io/awxbackups.awx.ansible.com unchanged
customresourcedefinition.apiextensions.k8s.io/awxrestores.awx.ansible.com unchanged
customresourcedefinition.apiextensions.k8s.io/awxs.awx.ansible.com unchanged
serviceaccount/awx-operator-controller-manager unchanged
role.rbac.authorization.k8s.io/awx-operator-awx-manager-role configured
role.rbac.authorization.k8s.io/awx-operator-leader-election-role unchanged
clusterrole.rbac.authorization.k8s.io/awx-operator-metrics-reader unchanged
clusterrole.rbac.authorization.k8s.io/awx-operator-proxy-role unchanged
rolebinding.rbac.authorization.k8s.io/awx-operator-awx-manager-rolebinding unchanged
rolebinding.rbac.authorization.k8s.io/awx-operator-leader-election-rolebinding unchanged
clusterrolebinding.rbac.authorization.k8s.io/awx-operator-proxy-rolebinding unchanged
configmap/awx-operator-awx-manager-config unchanged
service/awx-operator-controller-manager-metrics-service unchanged
deployment.apps/awx-operator-controller-manager configured
awx.awx.ansible.com/awx created
```

You can check the operator pod logs to know where the installation process is at :

```bash
kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager
```

Once deployed you should have a similar output :

```bash
 TASK [Remove ownerReferences reference] ********************************
ok: [localhost] => (item=None) => {"censored": "the output has been hidden due to the fact that 'no_log: true' was specified for this result", "changed": false}

-------------------------------------------------------------------------------

--------------------------- Ansible Task StdOut -------------------------------

TASK [installer : Start installation if auto_upgrade is false and deployment is missing] ***
task path: /opt/ansible/roles/installer/tasks/main.yml:31

-------------------------------------------------------------------------------
{"level":"info","ts":"2023-08-23T15:19:59Z","logger":"logging_event_handler","msg":"[playbook task start]","name":"awx","namespace":"awx","gvk":"awx.ansible.com/v1beta1, Kind=AWX","event_type":"playbook_on_task_start","job":"8059824194470061451","EventData.Name":"installer : Start installation if auto_upgrade is false and deployment is missing"}
{"level":"info","ts":"2023-08-23T15:20:00Z","logger":"runner","msg":"Ansible-runner exited successfully","job":"8059824194470061451","name":"awx","namespace":"awx"}

----- Ansible Task Status Event StdOut (awx.ansible.com/v1beta1, Kind=AWX, awx/awx) -----


PLAY RECAP *********************************************************************
localhost                  : ok=81   changed=0    unreachable=0    failed=0    skipped=81   rescued=0    ignored=1
```

You can check the new created resources :

```bash
kubectl get pods -l "app.kubernetes.io/managed-by=awx-operator"
```

The output should look like :

```
NAME                        READY   STATUS    RESTARTS      AGE
awx-postgres-13-0           1/1     Running   0             3m12s
awx-web-548c4747d5-9chw6    3/3     Running   0             3m12s
awx-task-544767f4d5-vkj4c   4/4     Running   0             3m12s
```

and also :

```
kubectl get svc -l "app.kubernetes.io/managed-by=awx-operator"
```

The output should look like :
```
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
awx-postgres-13   ClusterIP   None           <none>        5432/TCP       4m32s
awx-service       NodePort    10.43.141.96   <none>        80:30000/TCP   4m32s
```

The AWX instance should be accessible to the URL `http://<host_ip_address>:30000`.

By default, the admin user is `admin` and the password can be retrieved by running this :

```bash
kubectl get secret awx-admin-password -o jsonpath="{.data.password}" | base64 --decode ; echo
G7arl7kJlXrxhWznyGUv6uZ0G3iU7Gui
```

## Create a SSL certificate

Request a certificate signed by your intermediate CA (Vault cluster) by issuing the following command :

```
vault write pki/issue/issuer common_name="awx.dcsr.unil.ch" alt_names="infra-awx.ad.unil.ch,10.128.1.50" ttl="2160h"
```

This command output will display the private key, the related certificate and the CA certificate.

Generate the different files as follow :
- awx-cert.pem
- awx-key.pem
- awx-ca.pem

### Create a TLS secret in AWX namespace

Using Key and Certificate create a tls secret in awx namespace.

```
kubectl -n awx create secret tls  awx-cert --key ./awx-key.pem --cert ./awx-cert.pem
```

Successful output:
```
secret/awx-cert created
```

Verify secret creation with the following commands:
```
kubectl get secrets -n awx awx-cert
```

Successful output example:
```
NAME       TYPE                DATA   AGE
awx-cert   kubernetes.io/tls   2      20s
```

## Configure traefic for HTTPS

We will expose Ansible AWX Service using Traefik Ingress.

### Install Traefik Ingress Controller

Traefik Ingress is already bundled with k3s Kubernetes distribution. 

You can check the service with the following commands:

```
kubectl get svc -n kube-system
```

Successful output example:
```
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
kube-dns         ClusterIP      10.43.0.10      <none>         53/UDP,53/TCP,9153/TCP       6h32m
metrics-server   ClusterIP      10.43.140.220   <none>         443/TCP                      6h32m
traefik          LoadBalancer   10.43.156.54    10.128.1.150   80:31799/TCP,443:31418/TCP   6h32m
```

### Configure DNS record or modify hosts file

In the DNS server, create a new A record with the wished domain name and its IP address.

```
10.128.1.50  awx.dcsr.unil.ch
```

### Configure Traefik Ingress for AWX

Create a new manifest file for AWX ingress.

```
vim awx-traefik-ingress.yaml
```

Paste and modify the configurations provided here to suit your use case :
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: awx
  name: awx-ingress
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    # HTTPS as entry point
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    # Enable TLS
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  tls:
  - hosts:
    - awx.dcsr.unil.ch
    secretName: awx-cert
  rules:
    - host: awx.dcsr.unil.ch
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: awx-service
                port:
                  number: 80
```

Apply the new configuration in your Kubernetes cluster :

```
kubectl apply -f awx-traefik-ingress.yaml
```

Successful output example:
```
ingress.networking.k8s.io/awx-ingress created
```

To confirm if the ingress was created, use the following commands:
```
kubectl get ingress -n awx
```

Successful output example:
```
NAME          CLASS     HOSTS                  ADDRESS        PORTS     AGE
awx-ingress   traefik   dev-awx.dcsr.unil.ch   10.128.1.150   80, 443   50m
```

### Configure HTTP to HTTPS redirect

We are going to configure a new router to allow to redirect http traffic to https

Create a new manifest file for AWX ingress.

```
vim awx-traefik-ingress-redirect.yaml
```

Paste and modify the configurations provided here to suit your use case :
```
---
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect
  namespace: awx

spec:
  redirectScheme:
    scheme: https
    permanent: true
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: awx
  name: awx-ingress-redirect
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: web
    traefik.ingress.kubernetes.io/router.middlewares: awx-redirect@kubernetescrd

spec:
  rules:
    - host: awx.dcsr.unil.ch
      http:
        paths:
          - backend:
              service:
                name: awx-service
                port:
                  number: 80
            path: /
            pathType: ImplementationSpecific
```

Apply the new configuration in your Kubernetes cluster :

```
kubectl apply -f awx-traefik-ingress-redirect.yaml
```

Successful output example:
```
middleware.traefik.containo.us/redirect created
ingress.networking.k8s.io/awx-ingress-redirect created
```

To confirm if the ingress-redirect was created, use the following commands:
```
kubectl get ingress -n awx
```

Successful output example:
```
NAME                   CLASS     HOSTS                  ADDRESS        PORTS     AGE
awx-ingress            traefik   awx.dcsr.unil.ch       10.128.1.50    80, 443   56m
awx-ingress-redirect   traefik   awx.dcsr.unil.ch       10.128.1.50    80        40s
```

## First connection

You can now connect to AWX through the URL https://awx.dcsr.unil.ch


...
> TO BE CONTINUE
...

# Reference links

- Last release for AWX Operator : https://github.com/ansible/awx-operator/releases
- Redirect http to https with traefik : https://aqibrahman.com/set-up-traefik-kubernetes-ingress-for-http-and-https-with-redirect-to-https
