# OPA Gatekeeper
OPA Gatekeeper sample Rego Policies

## Disallowing Privileged Containers
It represents the broadest, most dangerous permission possible. <br/>
We will create an OPA Gatekeeper ```ConstraintTemplate``` and ```Constraint``` to enforce this.

#### ConstraintTemplate (The Rego Logic)
The ```ConstraintTemplate``` defines the reusable Rego logic. This policy iterates through all containers (main containers, init containers, and ephemeral containers) and reports a violation if any of them have ```privileged: true```.

```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/constrainttemplate.yaml
```

#### Constraint (The Enforcement)
The ```Constraint``` is an instance of the template that sets the specific parameters (like the violation message and where to apply the policy).
```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/constraint.yaml
```

#### How This Policy Works
1. **Intercepts:** <br/>
Gatekeeper Admission Controller intercepts all ```CREATE``` & ```UPDATE``` requests for ```Pods```, ```Deployments```, ```StatefulSets```, & ```DaemonSets```.
2. **Evaluation:** <br/>
The **Rego** code (from the ```ConstraintTemplate```) is executed against the incoming resource's YAML.
3. **Validation:** <br/>
The Rego logic checks the ```.spec.template.spec.containers``` (for Deployments, etc.) <br/>
or ```.spec.containers``` (for naked Pods) for a field named ```securityContext.privileged```.
4. **Enforcement:** <br/>
If ```privileged: true``` is found in a container spec, <br/>
and the object is not in ```excludedNamespace```, the ```violation``` rule is triggered.
5. **Rejection:** <br/>
Admission request is rejected with custom error ```message``` defined in the ```Constraint```, <br/>
preventing the insecure resource from ever being applied in-cluster.

#### Insecure Deployment Manifest
This YAML uses the highly secure Chainguard ```nginx``` image but overrides the security context at the Deployment level to introduce the security flaw your policy checks for.
```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/deployment.yaml
```
#### Secure Deployment Manifest (Policy Pass)
For comparison, here is the corrected, secure deployment that will pass the policy because it omits the insecure setting. <br/>
Since Chainguard images run as non-root by default, this template is secure without needing explicit ```runAsNonRoot``` or ```runAsUser``` settings.
```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/secure-deployment.yaml
```

<img width="1502" height="422" alt="Screenshot 2025-11-28 at 22 46 37" src="https://github.com/user-attachments/assets/bdd2cf33-5ba1-43c6-8e50-f8b6580c5798" />


#### Cleanup Exercise 1
```
alias kubectl="kubecolor"
kubectl delete -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/constrainttemplate.yaml -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/constraint.yaml -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/secure-deployment.yaml
```

<img width="1502" height="342" alt="Screenshot 2025-11-28 at 22 50 42" src="https://github.com/user-attachments/assets/9d75fe7d-46ee-465e-9968-0afac386a9d2" />


## Mandate NetworkPolicy
This policy requires every Namespace to have an active ```NetworkPolicy``` to ensure network segmentation is considered.

```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/constrainttemplate.yaml -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/constraint.yaml
```

Create the ```namespace``` and ```deployment``` resources:

```
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/secure-deployment.yaml -n policy-test-fail-ns
```

#### Cleanup Exercise 2
```
kubectl delete -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/constrainttemplate.yaml -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/constraint.yaml -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/privileges/secure-deployment.yaml -n policy-test-fail-ns -f https://raw.githubusercontent.com/ndouglas-cloudsmith/opa-gatekeeper/refs/heads/main/netpol/namespace.yaml
```
