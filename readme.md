# kind cluster: cka-practice (1 control plane + 2 workers)

## Create

```bash
cat > kind-cka-practice.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cka-practice
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --config kind-cka-practice.yaml --wait 120s
```

## Validate

```bash
kind get clusters                                  # cka-practice
kubectl config current-context                     # kind-cka-practice
kubectl get nodes -o wide                          # 3 nodes, all Ready
kubectl wait --for=condition=Ready nodes --all --timeout=120s
kubectl get pods -n kube-system                    # all Running
kubectl get sc                                     # standard (default)
docker ps --filter name=cka-practice --format '{{.Names}}'
# cka-practice-control-plane, cka-practice-worker, cka-practice-worker2
```

## Delete

```bash
kind delete cluster --name cka-practice
```
