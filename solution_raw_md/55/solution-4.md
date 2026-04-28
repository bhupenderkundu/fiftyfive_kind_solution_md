Challenge 4: Node Recovery
Node sanjay-challenge-worker2 has gone NotReady.

Goal: Bring the node back to Ready status and ensure it can schedule and run pods.

Hint: You'll need to get inside the node's container to debug. The node runs as a Docker container — use docker exec to access it. The issue is not a single problem.


What symptoms you observed
Node showing not ready in kubectl get node command
What tools you used to investigate
kubectl describe node
What the root cause was and how you confirmed it
1. kubelet not ready, service is down
2.  6804 bootstrap.go:241] "Unhandled Error" err="unable to read existing bootstrap client config from /etc/kubernetes/kubelet.conf: invalid configuration: [unable to read client-cert /var/lib/kubelet/pki/kubelet-client-current.pem for default-auth due to open /var/lib/kubelet/pki/kubelet-client-current.pem: no such file or directory, unable to read client-key /var/lib/kubelet/pki/kubelet-client-current.pem for default-auth due to open /var/lib/kubelet/pki/kubelet-client-current.pem: no such file or directory]" logger="UnhandledError"
Apr 28 11:35:30 sanjay-challenge-worker2 kubelet[6804]: E0428 11:35:30.233890    6804 run.go:72] "command failed" err="failed to run Kubelet: unable to load bootstrap kubeconfig: stat /etc/kubernetes/bootstrap-kubelet.conf: no such file or directory"
Apr 28 11:35:30 sanjay-challenge-worker2 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Apr 28 11:35:30 sanjay-challenge-worker2 systemd[1]: kubelet.service: Failed with result 'exit-code'.
root@sanjay-challenge-worker2:/# ls -la /var/lib/kubelet/pki/
ls -la /etc/kubernetes/
total 20
drwxr-xr-x 2 root root 4096 Apr 28 10:33 .
drwx------ 9 root root 4096 Apr 28 10:33 ..
-rw------- 1 root root 1135 Apr 28 10:31 kubelet-client-2026-04-28-10-31-11.pem
lrwxrwxrwx 1 root root   59 Apr 28 10:31 kubelet-client-current.pem.bak -> /var/lib/kubelet/pki/kubelet-client-2026-04-28-10-31-11.pem
-rw-r--r-- 1 root root 2375 Apr 28 10:31 kubelet.crt
-rw------- 1 root root 1675 Apr 28 10:31 kubelet.key
What you did to fix it
ln -s /var/lib/kubelet/pki/kubelet-client-2026-04-28-10-31-11.pem \
      /var/lib/kubelet/pki/kubelet-client-current.pem
root@sanjay-challenge-worker2:/# systemctl restart kubelet
How you verified the fix
kubectl get node
NAME                             STATUS                     ROLES           AGE   VERSION
sanjay-challenge-control-plane   Ready                      control-plane   67m   v1.32.2
sanjay-challenge-worker          Ready,SchedulingDisabled   <none>          67m   v1.32.2
sanjay-challenge-worker2         Ready,SchedulingDisabled   <none>          67m   v1.32.2
sanjay-challenge-worker3         Ready                      <none>          67m   v1.32.2