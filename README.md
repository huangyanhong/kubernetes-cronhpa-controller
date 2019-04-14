# kubernetes-cronhpa-controller 
[![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)
## Overview 
`kubernetes-cronhpa-controller` is a kubernetes cron horizontal pod autoscaler controller using `crontab` like scheme. You can use `CronHorizontalPodAutoscaler` with any kind object defined in kubernetes which support `scale` subresource(such as `Deployment` and `StatefulSet`). 


## Installation 
1. install CRD 
```$xslt
kubectl apply -f config/crds/autoscaling_v1beta1_cronhorizontalpodautoscaler.yaml
```
2. install RBAC settings 
```$xslt
# create ClusterRole 
kubectl apply -f config/rbac/rbac_role.yaml 
# create ClusterRolebinding and ServiceAccount 
kubectl apply -f config/rbac/rbac_role_binding.yaml 
```
3. deploy kubernetes-cronhpa-controller 
```$xslt
kubectl apply -f config/deploy/deploy.yaml 
```
4. verify installation
```$xslt
kubectl get cronhpa 

➜  kubernetes-cronhpa-controller git:(master) ✗ kubectl get cronhpa
NAME             AGE
cronhpa-sample   42s

kubectl get deploy kubernetes-cronhpa-controller -n kube-system -o wide 

➜  kubernetes-cronhpa-controller git:(master) ✗ kubectl get deploy kubernetes-cronhpa-controller -n kube-system
NAME                            DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-cronhpa-controller   1         1         1            1           49s
```
## Example 
Please try out the examples in the <a href="https://github.com/AliyunContainerService/kubernetes-cronhpa-controller/blob/master/examples">examples folder</a>.   

1. Deploy sample workload and cronhpa  
```$xslt
kubectl apply -f examples/deployment_hpa.yaml 
```

2. Check deployment replicas  
```$xslt
kubectl get deploy nginx-deployment-basic 

➜  kubernetes-cronhpa-controller git:(master) ✗ kubectl get deploy nginx-deployment-basic
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment-basic   2         2         2            2           9s
```

3. Describe cronhpa status 
```$xslt
kubectl describe cronhpa cronhpa-sample 

Name:         cronhpa-sample
Namespace:    default
Labels:       controller-tools.k8s.io=1.0
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"autoscaling.alibabacloud.com/v1beta1","kind":"CronHorizontalPodAutoscaler","metadata":{"annotations":{},"labels":{"controll...
API Version:  autoscaling.alibabacloud.com/v1beta1
Kind:         CronHorizontalPodAutoscaler
Metadata:
  Creation Timestamp:  2019-04-14T10:42:38Z
  Generation:          1
  Resource Version:    4017247
  Self Link:           /apis/autoscaling.alibabacloud.com/v1beta1/namespaces/default/cronhorizontalpodautoscalers/cronhpa-sample
  UID:                 05e41c95-5ea2-11e9-8ce6-00163e12e274
Spec:
  Jobs:
    Name:         scale-down
    Schedule:     30 */1 * * * *
    Target Size:  1
    Name:         scale-up
    Schedule:     0 */1 * * * *
    Target Size:  3
  Scale Target Ref:
    API Version:  apps/v1beta2
    Kind:         Deployment
    Name:         nginx-deployment-basic
Status:
  Conditions:
    Job Id:           38e79271-9a42-4131-9acd-1f5bfab38802
    Last Probe Time:  2019-04-14T10:43:02Z
    Message:
    Name:             scale-down
    Schedule:         30 */1 * * * *
    State:            Submitted
    Job Id:           a7db95b6-396a-4753-91d5-23c2e73819ac
    Last Probe Time:  2019-04-14T10:43:02Z
    Message:
    Name:             scale-up
    Schedule:         0 */1 * * * *
    State:            Submitted
Events:               <none>
```

if the `State` of cronhpa job is `Succeed` means the last execution is successful. `Submitted` means the cronhpa job is submitted to the cron engine but haven't be executed so far.

## Implementation Details
The following is an example of a `CronHorizontalPodAutoscaler`. 
```$xslt
apiVersion: autoscaling.alibabacloud.com/v1beta1
kind: CronHorizontalPodAutoscaler
metadata:
  labels:
    controller-tools.k8s.io: "1.0"
  name: cronhpa-sample
  namespace: default 
spec:
   scaleTargetRef:
      apiVersion: apps/v1beta2
      kind: Deployment
      name: nginx-deployment-basic
   jobs:
   - name: "scale-down"
     schedule: "30 */1 * * * *"
     targetSize: 1
   - name: "scale-up"
     schedule: "0 */1 * * * *"
     targetSize: 3
``` 
The `scaleTargetRef` is the field to specify workload to scale. If the workload supports `scale` subresource(such as `Deployment` and `StatefulSet`), `CronHorizontalPodAutoscaler` should work well. `CronHorizontalPodAutoscaler` support multi cronhpa job in one spec. 

The cronhpa job spec need three fields:
* name    
  `name` should be unique in one cronhpa spec. You can distinguish different job execution status by job name.
* schedule     
  The scheme of `schedule` is similar with `crontab`. `kubernetes-cronhpa-controller` use an enhanced cron golang lib （<a target="_blank" href="https://github.com/ringtail/go-cron">go-cron</a>） which support more expressive rules. 
  
  The cron expression format is as described below: 
  ```$xslt

    Field name   | Mandatory? | Allowed values  | Allowed special characters
    ----------   | ---------- | --------------  | --------------------------
    Seconds      | Yes        | 0-59            | * / , -
    Minutes      | Yes        | 0-59            | * / , -
    Hours        | Yes        | 0-23            | * / , -
    Day of month | Yes        | 1-31            | * / , - ?
    Month        | Yes        | 1-12 or JAN-DEC | * / , -
    Day of week  | Yes        | 0-6 or SUN-SAT  | * / , - ?    
  ```
  #### Asterisk ( * )    
  The asterisk indicates that the cron expression will match for all values of the field; e.g., using an asterisk in the 5th field (month) would indicate every month.
  #### Slash ( / )    
  Slashes are used to describe increments of ranges. For example 3-59/15 in the 1st field (minutes) would indicate the 3rd minute of the hour and every 15 minutes thereafter. The form "*\/..." is equivalent to the form "first-last/...", that is, an increment over the largest possible range of the field. The form "N/..." is accepted as meaning "N-MAX/...", that is, starting at N, use the increment until the end of that specific range. It does not wrap around.    
  #### Comma ( , )      
  Commas are used to separate items of a list. For example, using "MON,WED,FRI" in the 5th field (day of week) would mean Mondays, Wednesdays and Fridays.  
  #### Hyphen ( - )     
  Hyphens are used to define ranges. For example, 9-17 would indicate every hour between 9am and 5pm inclusive.   
  #### Question mark ( ? )      
  Question mark may be used instead of '*' for leaving either day-of-month or day-of-week blank.
  
  more schedule scheme please check this <a target="_blank" href="https://godoc.org/github.com/robfig/cron">doc</a>.
                                 
* targetSize
  `TargetSize` is the size you desired to scale when the scheduled time arrive. 


## Common Question  
* Cloud `kubernetes-cronhpa-controller` and HPA work together?       
Yes and no is the answer. `kubernetes-cronhpa-controller` can work together with hpa. But if the desired replicas is independent. So when the HPA min replicas reached `kubernetes-cronhpa-controller` will ignore the replicas and scale down and later the HPA controller will scale it up.

## Contributing
Please check <a href="https://github.com/AliyunContainerService/kubernetes-cronhpa-controller/blob/master/CONTRIBUTING.md">CONTRIBUTING.md</a>

## License
This software is released under the Apache 2.0 license.