# flux-test

## Baseline

```sh
flux install --namespace=test-system

kubectl apply -f clusters/baseline/flux-config.yaml
```

## Reproduction

```sh
kubectl create ns test-system
kubectl create sa flux-reconciler
kubectl create rolebinding flux-reconciler-to-cr-admin --clusterrole admin --serviceaccount test-system:flux-reconciler

flux install --namespace=test-system

kubectl apply -f clusters/reproduction/flux-config.yaml
```
