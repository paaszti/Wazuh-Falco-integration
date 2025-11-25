# Wazuh-Falco-integration
Simple Wazuh falco integration to extend visibility in container environment

1. Falco configuration with helm
```sh
helm install falco falcosecurity/falco \
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
2. Wazuh Kubernetes configuration
