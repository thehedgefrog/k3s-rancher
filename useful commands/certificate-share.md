```sh
kubectl get secret **certificate name** -n **origin namespace** --export -o yaml | \
kubectl apply -n **destination namespace** -f -
```