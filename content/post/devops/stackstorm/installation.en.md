---
title: "StackStorm"
date: 2021-02-17T11:25:44+01:00
draft: false

tags:
  - "helm"
  - "kubernetes"

categories:
  - "devops"


---

## Installation

Using Helm:
```bash
helm repo add stackstorm https://helm.stackstorm.com/
helm install st2 stackstorm/stackstorm-ha
```

Create a service in `st2-svc.yaml` with LoadBalancer type:
```yaml
apiVersion: v1
kind: Service
metadata:  
  name: st2-service  
  namespace: default
spec:
  type: LoadBalancer  
  ports:    
    - port: 80      
      targetPort: 80      
      protocol: TCP  
  selector:    
    app: st2web
```

Get EXTERNAL-IP with:
```
kubectl get svc st2-service
```

## St2 CLI

- Following the steps in [st2client documentation](https://docs.stackstorm.com/install/k8s_ha.html#st2client):

```bash
# obtain st2client pod name
ST2CLIENT=$(kubectl get pod -l app=st2client -o jsonpath="{.items[0].metadata.name}")

# run a single st2 client command
kubectl exec -it ${ST2CLIENT} -- st2 --version

# switch into a container shell and use st2 CLI
kubectl exec -it ${ST2CLIENT} /bin/bash

# authenticate to StackStorm using st2 CLI and save the default password using the -w flag
kubectl exec -it ${ST2CLIENT}  -- st2 login st2admin -p 'Ch@ngeMe' -w
```

## Work with actions

Create a sample rule `sample_rule_with_webhook.yaml` :

```yaml
---
    name: "sample_rule_with_webhook"
    pack: "examples"
    description: "Sample rule dumping webhook payload to a file."
    enabled: true

    trigger:
        type: "core.st2.webhook"
        parameters:
            url: "sample"

    criteria:
        trigger.body.name:
            pattern: "st2"
            type: "equals"

    action:
        ref: "core.local"
        parameters:
            cmd: "echo \"{{trigger.body}}\" >> ~/st2.webhook_sample.out ; sync"
```

Deploy the sample rule:
```
kubectl exec -it ${ST2CLIENT} -n stackstorm -- st2 rule createfirst_rule.yaml
```
Get X-Auth-Token: 

```
kubectl exec -it st2-st2client-79884d9544-fp965  -- st2 auth st2admin -p 'Ch@ngeMe'
```

Execute the POST request:
```
curl -k http://35.204.170.36/api/v1/webhooks/sample -d '{"foo": "bar", "name": "st2"}' -H 'Content-Type: application/json' -H 'X-Auth-Token: 0ceb7bfbff0c467f880d9d117096bad8'
```

## Change password

```
sudo htpasswd /etc/st2/htpasswd st2admin
sudo st2ctl restart
```


---
## Draft 

Get internal IP:
```
export ST2WEB_IP=$(minikube ip 2>/dev/null || kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
export ST2WEB_PORT="$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services st2-st2web)"
echo http://${ST2WEB_IP}:${ST2WEB_PORT}/
```

--- 

## References

- [Kubernetes a DevOps Cookbook](https://www.packtpub.com/product/kubernetes-a-complete-devops-cookbook/9781838828042)

- [StackStorm Documentation, Define a Rule](https://docs.stackstorm.com/start.html#define-a-rule)