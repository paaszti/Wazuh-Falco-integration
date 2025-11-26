# Wazuh falco integration & DVWA  on MicroK8s
This project documents the automated deployment of a security monitoring and testing environment on a MicroK8s cluster.

Stack:
1. **Falco** version: 0.41.3 (x86_64), engine: ebpf
2. **Wazuh** 4.8.1
3. **DVWA** (Damn Vulnerable Web Application) pod to perform simulated attacks

## Infrastructure & Prerequisites
**Virtualization Evironment**
* Hypervisor: Proxmox VE 9
* Cluster topology: 2-Node Microk8s Cluster

**VM Specifications**
* OS: Ubuntu 24.04.3 LTS
* CPU: 4vCPUs
* RAM: 15GB
* Storage: min. 50GB
* Kubernetes platform: microk8s

**Microk8s required addons**
* DNS
* helm3
* metallb

## Installing microk8s
```sh
sudo snap install microk8s --classic --channel=1.33
microk8s status --wait-ready
alias kubectl='microk8s kubectl'
```
### Enable addons `dns, helm3, metallb`
```sh
microk8s enable dns helm3
microk8s enable metallb:[example range: 192.168.1.200-192.168.1.220]
```

### Creating cluster
To create a cluster out of two or more already-running MicroK8s instances, use the `microk8s add-node` command. The MicroK8s instance on which this command is run will be the master of the cluster and will host the Kubernetes control plane:
```sh
microk8s add-node
```
This will return some joining instructions which should be executed on the MicroK8s instance that you wish to join to the cluster
```sh
From the node you wish to join to this cluster, run the following:
microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05

Use the '--worker' flag to join a node as a worker not running the control plane, eg:
microk8s join 192.168.1.230:25000/92b2db237428470dc4fcfc4ebbd9dc81/2c0cb3284b05 --worker
```
For more instructions go to: `https://microk8s.io/docs/clustering`


## Cloning repo
```sh
git clone https://github.com/paaszti/Wazuh-Falco-integration.git
cd Wazuh-Falco-integration
```

## Installing falco & custom rules
Falco is installed using helm3 with specific configuration to enable custom rules.

### Installation command
Run the following command in the directory where custom-rules.yaml is located.
```sh
helm3 repo add falcosecurity https://falcosecurity.github.io/charts
helm3 repo update

helm3 install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set falco.json_output=true \
  --set falco.syslog_output.enabled=false \
  --set tty=true \
  --set collectors.kubernetes.enabled=true \
  --set falcoctl.artifact.install.enabled=true \
  --set falcoctl.artifact.follow.enabled=false \
  --set 'falco.rules_files={/etc/falco/rules.d}' \
  --set-json 'falco.append_output=[{"match":{"source":"syscall"},"extra_output":"pod_uid=%k8smeta.pod.uid, pod_name=%k8smeta.pod.name, namespace_name=%k8smeta.ns.name","extra_fields":[{"wazuh_integration":"falco"}]}]' \
  -f custom-rules.yaml
```

### Custom rules `(custom-rules.yaml)`
This file contains detection rules to detect: 
* reconnaissance `ps aux, ps -ef, df -h, df -a, hostname -I, hostname -i`
* open port
* mount operation
* su usage
* shell in container

These rules are loaded during installation. Post-installation, they are stored in a Kubernetes ConfigMap. You can edit the ConfigMap with this command:
```sh
kubectl edit configmap falco-rules -n falco
```

### Veryfing
```sh
kubectl get daemonset -n falco
kubectl get pods -n falco -o wide
```


## Wazuh installation with fine tuned manifests
### Default installation installation:
```
git clone https://github.com/wazuh/wazuh-kubernetes.git -b v4.8.2 --depth=1
cd wazuh-kubernetes

#Generating self-signed SSL certs
wazuh/certs/indexer_cluster/generate_certs.sh
wazuh/certs/dashboard_http/generate_certs.sh
```
There I changed Wazuh default manifests, so you can just simple copy them:
```sh
cp wazuh-worker-sts.yaml ./wazuh-kubernetes/wazuh/wazuh_managers/
cp wazuh-master-sts.yaml ./wazuh-kubernetes/wazuh/wazuh_managers/
cp indexer-resources.yaml ./wazuh-kubernetes/envs/local-env/
```

Then you can apply all manifests using kustomize
```sh
kubectl apply -k envs/local-env/
```

### Verifying the deployment
Namespace
```sh
kubectl get ns | grep wazuh
```

Services
```sh
kubectl get svc -n wazuh
```

Deployments & Statefulset
```sh
kubectl get deployments -n wazuh
kubectl get statefulsets -n wazuh
```

Pods
```sh
kubectl get pods -n wazuh
```

### Accessing Wazuh dashboard
First locate the IP address of dashboard
```sh
kubectl get svc -o wide | grep dashboard
```
Then you access dashboard: `https://<IP>:8443`\
Credentials: `admin:SecretPassword`

### Rules & Decoders
You have to apply rules `falco-rules.xml` and decoder `falco-decoder.xml` to the Wazuh Manager.\
You can do this with Wazuh Dashboard.
* Server management -> Decoders -> Add new decoders file -> Paste content from `falco-decoder.xml` -> Save
* Server management -> Rules -> Add new rules file -> Paste content from `falco-rules.xml` -> Save

After applying rules and decoders, Wazuh Manager restart is required.

## DVWA
Vulnerable pod to execute attacks and trigger alerts in Falco and see logs in Wazuh Dashboard.
### Building image
```sh
docker build -t mydvwa:v1
#Save to the .tar file
docker save mydvwa:v1 > dvwa.tar

#Save to the ctr
microk8s images import dvwa.tar
```
Then apply deployment
```sh
kubectl apply -f dvwa.yaml
```

### Veryfing
```sh
kubectl get pods -n dvwa
kubectl get svc -n dvwa
```

## Deploying Wazuh agent
Deploying an agent: [How to deploy an agent](https://documentation.wazuh.com/4.8/installation-guide/wazuh-agent/index.html)\
You have to add these lines to the end of the file: `/var/ossec/etc/ossec.conf`
```sh
 <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/pods/[falco-pod-name]/falco/*.log</location>
  </localfile>
```
Then restart Wazuh agent
```sh
systemctl restart wazuh-agent
```

## TODO:
* CI/CD
* Terraform
* Ansible
* Connector to MS Sentinel
* Rebuild image 
