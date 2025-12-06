1- check ebpf فعال هست ؟

```
if cilium
cilium status

Datapath:   eBPF
BPF Maps: OK
BPF Kernel: OK



if calico
root@k8s01:~# kubectl get felixconfiguration default -o yaml | grep -i BPF
    operator.tigera.io/bpfEnabled: "false"
  bpfConnectTimeLoadBalancing: TCP
  bpfEnabled: false
  bpfHostNetworkedNATWithoutCTLB: Enabled
  bpfLogLevel: ""
```

2- install gvisor انجام شده است

kubectl get runtimeclass

NAME        HANDLER
gvisor      runsc

runsc       runsc
```

3- USE RUNTIME DEFAULT IN SECOMP

```
ps aux | grep kubelet | grep seccomp
--seccomp-default=true

kubectl get pod POD-NAME -n NAMESPACE -o yaml | grep -A3 seccomp

kubectl get pod POD-NAME -n NAMESPACE -o jsonpath='{.metadata.annotations}'
seccomp.security.alpha.kubernetes.io/pod: runtime/default
```
4- use afiinity & node selector
```
kubectl get deploy -A -o yaml | grep -A5 nodeSelector
kubectl get pods -A -o yaml | grep -A5 nodeSelector

kubectl get deploy -A -o yaml | grep -A20 affinity
kubectl get pods -A -o yaml | grep -A20 affinity
```
5-check default deny on CNI
```
kubectl get networkpolicy --all-namespaces
اگر هیچ NetworkPolicy وجود ندارد → default-deny اعمال نشده است.
```



