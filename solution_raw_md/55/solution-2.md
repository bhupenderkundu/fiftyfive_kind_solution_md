# Challenge 2: Fix the Broken Deployment
In namespace t2, the deployment task-2 wants 3 healthy replicas but all pods are failing.

Goal: Get all 3 replicas of task-2 running and ready.

Hint: There are multiple issues. The first fix won't be the last.


# What symptoms you observed
1. Initially 2 pods are in pending state.
2. Pod have wrong tag for image " ImagePullBackOff= Failed to pull image "nginx:1.19-alpne": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:1.19-alpne": failed to resolve reference "docker.io/library/nginx:1.19-alpne": docker.io/library/nginx:1.19-alpne: not found"
3. All 3 replicas not available as mentioned in deployment replicas

# What tools you used to investigate
1. Kubectl describe pod/$pod_name
2. Kubectl describe pod/$pod_name events showing reason with error
3. kubectl get deploy -n t2
   NAME     READY   UP-TO-DATE   AVAILABLE   AGE
   task-2   0/3     2            0           24m
# What the root cause was and how you confirmed it
1. Pod have key with node selector which is not available in existing node selector
  nodeSelector:
    disk: ssd

=======
 kubectl get node --show-labels
NAME                             STATUS                        ROLES           AGE     VERSION   LABELS
sanjay-challenge-control-plane   Ready                         control-plane   9m6s    v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress-ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=sanjay-challenge-control-plane,kubernetes.io/os=linux,node-role.kubernetes.io/control-plane=,node.kubernetes.io/exclude-from-external-load-balancers=
sanjay-challenge-worker          Ready,SchedulingDisabled      <none>          8m53s   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=sanjay-challenge-worker,kubernetes.io/os=linux
sanjay-challenge-worker2         NotReady,SchedulingDisabled   <none>          8m53s   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=sanjay-challenge-worker2,kubernetes.io/os=linux
sanjay-challenge-worker3         Ready                         <none>          8m54s   v1.32.2   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=sanjay-challenge-worker3,kubernetes.io/os=Linux
===================================================================================
2.  docker pull nginx:1.19-alpne 
Error response from daemon: failed to resolve reference "docker.io/library/nginx:1.19-alpne": docker.io/library/nginx:1.19-alpne: not found
====================================================
3. Events:
  Type     Reason            Age                   From                   Message
  ----     ------            ----                  ----                   -------
  Warning  FailedCreate      3m41s                 replicaset-controller  Error creating: pods "task-2-5b9fd4c864-ldgkb" is forbidden: exceeded quota: tight-quota, requested: pods=1,requests.memory=64Mi, used: pods=2,requests.memory=128Mi, limited: pods=2,requests.memory=150Mi
=> kubectl get quota -n t2
NAME          AGE   REQUEST                                   LIMIT
tight-quota   26m   pods: 2/2, requests.memory: 128Mi/150Mi
# What you did to fix it
1. we can update the deployment with node selector or we can add the label in node level. As of now add label to node.
   kubectl label node sanjay-challenge-worker disk=ssd 
2. Incorrect name of tag it should be docker.io/library/nginx:1.19-alpine
3. Updated the quote hard pod and memory size
# How you verified the fix
1. all pods are up and running
 kubectl get pod -n t2
NAME                      READY   STATUS    RESTARTS   AGE
task-2-5b9fd4c864-2rbtg   0/1     Running   0          2s
task-2-5b9fd4c864-q2s6m   0/1     Running   0          3s
task-2-5b9fd4c864-rdw99   0/1     Running   0          2s
