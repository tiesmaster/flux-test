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
kubectl create sa flux-reconciler
kubectl create rolebinding flux-reconciler-to-cr-admin --clusterrole admin --serviceaccount test-system:flux-reconciler

flux install --namespace=test-system

kubectl apply -f clusters/reproduction/flux-config.yaml
```

KS indicates failure to reconcile:

```sh
$ flux get ks -n test-system
NAME	REVISION	SUSPENDED	READY	MESSAGE                                                                                                                                                                                                                                                                         
test	        	False    	False	Role/test-system/test-role dry-run failed (Forbidden): roles.rbac.authorization.k8s.io "test-role" is forbidden: User "system:serviceaccount:test-system:flux-reconciler" cannot patch resource "roles" in API group "rbac.authorization.k8s.io" in the namespace "test-system"	
    	        	         	     	                                                                                                                                                                                                                                                                               	
```
