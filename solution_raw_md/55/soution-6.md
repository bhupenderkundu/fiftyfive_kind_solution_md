Challenge 6: Performance Triage Under Load
In namespace t6, the deployment api-server is experiencing intermittent failures. A load-generator pod is sending continuous traffic to the service.

Rules:

Do NOT stop or delete the load-generator pod
Do NOT reduce the load generator's request rate
Goal: Make the api-server handle all requests successfully with response times under 2 seconds. An HPA is configured but isn't working.

Hint: There are resource constraints at multiple levels. Check what's limiting scaling.

What symptoms you observed
API-service is working fine there is no restart in pod, Check the HPA events. getting metrics issue
What tools you used to investigate
Kubectl descripe HPA
Deployment pods:                                       1 current / 0 desired
Conditions:
  Type           Status  Reason                   Message
  ----           ------  ------                   -------
  AbleToScale    True    SucceededGetScale        the HPA controller was able to get the target's current scale
  ScalingActive  False   FailedGetResourceMetric  the HPA was unable to compute the replica count: failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)
Events:
  Type     Reason                   Age                     From                       Message
  ----     ------                   ----                    ----                       -------
  Warning  FailedGetResourceMetric  4m26s (x381 over 102m)  horizontal-pod-autoscaler  failed to get cpu utilization: unable to get metrics for resource cpu: unable to fetch metrics from resource metrics API: the server could not find the requested resource (get pods.metrics.k8s.io)
What the root cause was and how you confirmed it

metrics server not installed in the kind server , kubectl get pods -n kube-system | grep metrics-server
What you did to fix it
installing the metrivcs servcer as first step.

How you verified the fix
kubectl get hpa -n t6 -w
NAME      REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
api-hpa   Deployment/api-server   cpu: 2%/50%   1         5         1          106m

