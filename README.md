# flux-test

## Baseline

```sh
minikube start

flux install --namespace=test-system

kubectl apply -f clusters/baseline/flux-config.yaml
```

KS indicates success:

```sh
$ flux get ks -n test-system
NAME	REVISION          	SUSPENDED	READY	MESSAGE
test	main@sha1:402cd347	False    	True 	Applied revision: main@sha1:402cd347
```

## Reproduction

```sh
minikube start

kubectl create ns test-system
kubectl create -n test-system sa flux-reconciler
kubectl create -n test-system rolebinding flux-reconciler-to-cr-admin --clusterrole admin --serviceaccount test-system:flux-reconciler

flux install --namespace=test-system

kubectl apply -f clusters/reproduction/flux-config.yaml
```

KS indicates failure to reconcile:

```sh
$ flux get ks -n test-system
NAME	REVISION	SUSPENDED	READY	MESSAGE                                                                                                           
test	        	False    	False	RoleBinding/test-system/test-role-binding not found: rolebindings.rbac.authorization.k8s.io "test-role" not found	
    	        	         	     	                                                                                                                 	
```


# Reproduce native K8s "bug" (but is actually a feature)

## Setup

```sh
minikube start

kubectl create ns test-system
kubectl create -n test-system sa flux-reconciler
kubectl create -n test-system rolebinding flux-reconciler-to-cr-admin --clusterrole admin --serviceaccount test-system:flux-reconciler
```

## Baseline

_This diffs, and creates the role, and rolebinding via the default user to minikube, which is a cluster-admin. Everything succeeds there._

```sh
# Diff works for any variations

$ kubectl diff -f role.yaml
<output removed for brevity>

$ kubectl diff -f rolebinding.yaml
<output removed for brevity>

$ kubectl diff -f role-and-binding.yaml
<output removed for brevity>

# Apply works for both combined
$ kubectl apply -f role-and-binding.yaml
role.rbac.authorization.k8s.io/test-role created
rolebinding.rbac.authorization.k8s.io/test-role-binding created

$ kubectl delete -f role-and-binding.yaml
role.rbac.authorization.k8s.io "test-role" deleted from test-system namespace
rolebinding.rbac.authorization.k8s.io "test-role-binding" deleted from test-system namespace

# Apply also works for the binding without the role
$ kubectl apply -f rolebinding.yaml
rolebinding.rbac.authorization.k8s.io/test-role-binding created

$ kubectl delete -f rolebinding.yaml
rolebinding.rbac.authorization.k8s.io "test-role-binding" deleted from test-system namespace
```

## Reproduction

_This diffs, and creates the role, and rolebinding via the serviceaccount user  which is an admin. The diff for both fails, but the apply does work in the combined situation._

```sh
# Diff works the role, but not the other 2 variations

$ kubectl --as system:serviceaccount:test-system:flux-reconciler diff -f role.yaml
<succeeds, output removed for brevity>

$ kubectl --as system:serviceaccount:test-system:flux-reconciler diff -f rolebinding.yaml
Error from server (NotFound): rolebindings.rbac.authorization.k8s.io "test-role" not found

$ kubectl --as system:serviceaccount:test-system:flux-reconciler diff -f role-and-binding.yaml
Error from server (NotFound): rolebindings.rbac.authorization.k8s.io "test-role" not found

# Apply works for all, except the rolebinding to non-existing role, which is an escalation, and hence not allowed.
$ kubectl --as system:serviceaccount:test-system:flux-reconciler apply -f role-and-binding.yaml
role.rbac.authorization.k8s.io/test-role created
rolebinding.rbac.authorization.k8s.io/test-role-binding created

$ kubectl delete -f role-and-binding.yaml
role.rbac.authorization.k8s.io "test-role" deleted from test-system namespace
rolebinding.rbac.authorization.k8s.io "test-role-binding" deleted from test-system namespace

$ kubectl --as system:serviceaccount:test-system:flux-reconciler apply -f rolebinding.yaml
Error from server (NotFound): error when creating "rolebinding.yaml": rolebindings.rbac.authorization.k8s.io "test-role" not found
```
