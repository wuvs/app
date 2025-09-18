# Karmada/ArgoCD

## Prerequisite
- increase fs.inotify.max_user_watches
- increase fs.inotify.max_user_instances
- increase fs.file-max
- apply `file.patch` to karmada repo for 4 clusters

## CLI Tools
- kind
- kubectl
- helm
- cloud-provider-kind
- argocd
- argo-rollouts

## Create karmada clusters
```bash
hack/local-up-karmada.sh
```

## Install argocd on karmada-host context

```bash
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-host create namespace argocd
kubectl --kubeconfig ~/.kube/karmada.config apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl --kubeconfig ~/.kube/karmada.config patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```


## Install argo-rollouts crd on karmada-apiserver

```bash
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-apiserver apply -k https://github.com/argoproj/argo-rollouts/manifests/crds?ref=v1.8.3

```


## Install argo-rollouts on all members context

```bash

for i in {1..4}; do
    kubectl --kubeconfig ~/.kube/members.config --context member$i create namespace argo-rollouts
    kubectl --kubeconfig ~/.kube/members.config --context member$i apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
done
```

## Create load-balancer

```bash
kubectl --kubeconfig ~/.kube/karmada.config config use-context karmada-host
cloud-provider-kind
```

## Register karamda to argocd

```bash
kubectl --kubeconfig ~/.kube/karmada.config config use-context karmada-host
argocd admin initial-password -n argocd
argocd login <LB_IP>
argocd cluster add karmada-apiserver
```


## Create argocd app

```bash
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-host apply -f app.yaml
```


## Watching/promoting deployments

```bash
kubectl argo rollouts get rollout app -n app --context member1 --watch
kubectl argo rollouts get rollout app -n app --context member2 --watch

kubectl argo rollouts promote app -napp --context member1
kubectl argo rollouts promote app -napp --context member2
```

## Manually continue deployment member3/member4

**Demo/POC purpose only**

*This step will mark the propagation policy out of sync within argo.*

*Additional logic into helm chart to be able to update values*

```yaml
karmada:
  activePhases:
    - "wave-1"

  phasedRollout:
    - phaseName: "wave-1"
      clusters:
        - member1
        - member2
    - phaseName: "wave-2"
      clusters:
        - member3
        - member4
```


```bash
kubectl --kubeconfig ~/.kube/karmada.config --context karmada-apiserver patch propagationpolicy app-phased -n app \
  --type='json' \
  -p='[{"op": "remove", "path": "/spec/suspension"}]'
```

## Watching deployments

```bash
kubectl argo rollouts get rollout app -n app --context member3 --watch
kubectl argo rollouts get rollout app -n app --context member4 --watch

kubectl argo rollouts promote app -napp --context member3
kubectl argo rollouts promote app -napp --context member4
```
