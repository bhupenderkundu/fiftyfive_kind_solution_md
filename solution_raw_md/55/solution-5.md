Challenge 5: TLS Certificate Debugging
In namespace t5, there is a deployment secure-app running nginx configured for HTTPS on port 443, and a pod tls-client with curl installed.

A CA bundle is mounted at /etc/ssl/custom/ca.crt inside the tls-client pod.

Goal: Make this command succeed without using --insecure or -k:

kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt https://secure-app.t5.svc.cluster.local
It must return: TLS Challenge Complete!

Hint: There are multiple certificate-related issues. The server may not even start initially.

What symptoms you observed
1. Secure app is not up and running to provide the result

 kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt https://secure-app.t5.svc.cluster.local
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
  0      0   0      0   0      0      0      0                              0
curl: (7) Failed to connect to secure-app.t5.svc.cluster.local port 443 after 128 ms: Could not connect to server
command terminated with exit code 7
root@Bhupender:~/devops-hiring-assignment/terraform# kubectl get pod -n t5
NAME                         READY   STATUS             RESTARTS         AGE
secure-app-d5d56dc59-5vzmh   0/1     CrashLoopBackOff   17 (3m27s ago)   68m
tls-client                   1/1     Running            0                68m

What tools you used to investigate

kubectl log to get the issue
 kubectl log secure-app-d5d56dc59-5vzmh -n t5
error: unknown command "log" for "kubectl"

Did you mean this?
        top
        logs
root@Bhupender:~/devops-hiring-assignment/terraform# kubectl logs secure-app-d5d56dc59-5vzmh -n t5
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: can not modify /etc/nginx/conf.d/default.conf (read-only file system?)
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/04/28 11:36:45 [emerg] 1#1: cannot load certificate "/etc/nginx/ssl/tls.crt": PEM_read_bio_X509_AUX() failed (SSL: error:0909006C:PEM routines:get_name:no start line:Expecting: TRUSTED CERTIFICATE)
nginx: [emerg] cannot load certificate "/etc/nginx/ssl/tls.crt": PEM_read_bio_X509_AUX() failed (SSL: error:0909006C:PEM routines:get_name:no start line:Expecting: TRUSTED CERTIFICATE)

What the root cause was and how you confirmed it
 kubectl get secret tls-secret -n t5 -o jsonpath='{.data.tls\.crt}' | base64 -d
-----BEGIN PRIVATE KEY-----
What you did to fix it
crt and keye are swapped now re-swapped to create the right secret
also updated the configmap ca bundle with original ca crt text file
How you verified the fix
kubectl exec tls-client -n t5 -- curl --cacert /etc/ssl/custom/ca.crt https://secure-app.t5.svc.cluster.local
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
  0      0   0      0   0      0      0      0                              0TLS Challenge Complete!
100     24 100     24   0      0    667      0