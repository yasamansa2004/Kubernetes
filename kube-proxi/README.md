if your cluster run kube-proxy with a DaemonSet, you can see the proxies by
running:
   ```
   $ kubectl get daemonSets --namespace=kube-system kube-proxy

      NAME    DESIRED    CURRENT    READY    UP-TO-DATE    AVAILABLE    NODE    SELECTOR
   kube-proxy    5          5          5          5             5        ...       45d
```
