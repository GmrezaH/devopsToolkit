# Adding custom host entry to Nodelocal DNS cache

1. Edit the nodelocaldns configmap

   ```bash
   kubectl -n kube-system edit configmaps nodelocaldns
   ```

1. Add `hosts` section to `.:53` section of nodelocaldns configmap

   ```
   Corefile: |
     ...
     .:53 {
         errors
         cache 30
         reload
         loop
         bind 169.254.25.10
         forward . 8.8.8.8 8.8.4.4
         prometheus :9253
         hosts {
           192.168.1.2 host.example.com
         }
     }
   ```

1. Restart nodelocaldns daemonset

   ```bash
   kubectl -n kube-system rollout restart daemonset nodelocaldns
   ```
