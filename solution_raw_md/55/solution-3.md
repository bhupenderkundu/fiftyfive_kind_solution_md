In namespace t3, there is a deployment task-3 running a standard nginx server with a service exposing port 80.

In the default namespace, a pod debug-client (with full networking tools) has been deployed.

Goal: From inside debug-client, successfully run:

curl http://task-3.t3.svc.cluster.local
It should return the nginx welcome page.

Hint: There are multiple layers blocking connectivity. The obvious one isn't the only one.\


What symptoms you observed
Pod not able to resolve the DNS
kubectl describe svc -n t3
Name:                     task-3
Namespace:                t3
Labels:                   app=task-3
Annotations:              <none>
Selector:                 app=task-3
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.108.14.27
IPs:                      10.108.14.27
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
Endpoints:                10.244.1.5:80 Pod_IP
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
root@Bhupender:~/devops-hiring-assignment/terraform# kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
debug-client   1/1     Running   0          33m
root@Bhupender:~/devops-hiring-assignment/terraform# kubectl exec -it debug-client -- sh
~ # curl http://task-3.t3.svc.cluster.local
curl: (6) Could not resolve host: task-3.t3.svc.cluster.local (Domain name not found)
What tools you used to investigate
execute into the pod and try to curl the command

What the root cause was and how you confirmed it
Issue with Kubernetes networking as checked it only not working for this specific
Need to check the coredns  pod no issue in it
Need to check the coredns  config
}
        ready
        rewrite name task-3.t3.svc.cluster.local task-3.t3.svc.cluster.invalid issue
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }

What you did to fix it
remove the blocking re-write line form configmap

How you verified the fix
kubectl exec -it debug-client -- sh
~ # curl http://task-3.t3.svc.cluster.local
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
~ #