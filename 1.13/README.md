## Demo 1 - generate foreach
1. prepare 4 namespaces with default deny all networkpolicy
```
$ k get ns -l test=demo                                    
NAME    STATUS   AGE
alice   Active   6h5m
bob     Active   6h5m
carol   Active   6h5m
dave    Active   5h48m

$ k get netpol -A
NAMESPACE   NAME               POD-SELECTOR   AGE
alice       default-deny-all   <none>         6h9m
bob         default-deny-all   <none>         6h9m
carol       default-deny-all   <none>         6h9m
dave        default-deny-all   <none>         5h53m
```
```yaml
$ k get netpol -n alice default-deny-all -o yaml | k neat
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: alice
spec:
  policyTypes:
  - Ingress
  - Egress
```
2. create the generate policy
```
$ k create -f 1-1-add-allowed-networkpolicy.yaml 
clusterpolicy.kyverno.io/add-allowed-networkpolicy created

$ k get cpol
NAME                        ADMISSION   BACKGROUND   READY   AGE   MESSAGE
add-allowed-networkpolicy   true        true         True    30s   Ready
```
3. create the trigger namespace
```
$ k create -f 1-2-namespace.yaml 
namespace/shuting created
```
4. verify the new networkpolicies are created in namespace `alice`, `bob` and `carol`, but not `dave`
```
$ k get netpol -A               
NAMESPACE   NAME                              POD-SELECTOR   AGE
alice       allow-all-between-shuting-alice   <none>         20s
alice       default-deny-all                  <none>         6h7m
bob         allow-all-between-shuting-bob     <none>         20s
bob         default-deny-all                  <none>         6h7m
carol       allow-all-between-shuting-carol   <none>         20s
carol       default-deny-all                  <none>         6h7m
dave        default-deny-all                  <none>         5h51m
```
## Demo 2 - warnings for Policy Violations and Mutations 
1. create the policy
```
$ k create -f 2-1-policy-validate.yaml                
clusterpolicy.kyverno.io/enforce-pod-security created
```
2. check the policy violation is returned as the warning when creating a new pod
```
$ k run nginx --image=nginx --dry-run=server   
Warning: policy enforce-pod-security.validate-pod-security: Validation rule 'validate-pod-security' failed. It violates PodSecurity "restricted:latest": (Forbidden reason: allowPrivilegeEscalation != false, field error list: [spec.containers[0].securityContext.allowPrivilegeEscalation: Required value])(Forbidden reason: unrestricted capabilities, field error list: [spec.containers[0].securityContext.capabilities.drop: Required value])(Forbidden reason: runAsNonRoot != true, field error list: [spec.containers[0].securityContext.runAsNonRoot: Required value])(Forbidden reason: seccompProfile, field error list: [spec.containers[0].securityContext.seccompProfile.type: Required value])
pod/nginx created (server dry run)
```
3. same thing for a mutate policy
```
$ k create -f 3-1-policy-mutate.yaml        
clusterpolicy.kyverno.io/add-owner-label created
```
4. check the warning
```
$ k create -f 3-2-namespace.yaml    
Warning: policy add-owner-label.apply-accurate-template: mutated Namespace/shuting-test-mutation
namespace/shuting-test-mutation created
```
5. check the policy report for the mutation
```
$ k get cpolr
NAME                                   KIND        NAME                    PASS   FAIL   WARN   ERROR   SKIP   AGE
3a0559e1-c17f-41ff-83f3-29ca4c11d762   Namespace   shuting-test-mutation   1      0      0      0       0      12s
```

