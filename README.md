<!--
title: 'web app architecture'
description: 'This AWS lambda function will terminate AWS EC2 instance daily when the last commit of specific github repository:branch is older than 3 days.'
layout: Doc
framework: v1
platform: AWS
language: nodeJS
authorName: 'Tony Lee'
-->
# web app architecture
This picture demostrate two different architecture solutions (serverless and k8s) for auto scalable web app.
![](https://raw.githubusercontent.com/dogzzdogzz/web-app-architecture/master/web-app-architecture.png)

## AWS serverless architecture
For the apps using AWS serverless architecture, we don't need to take care of too much about load balancing, session handling and so on since everything is managed by AWS. But still need to pay attention on some AWS limit like Lambda concurrency pool and API gateway throttling.

## Kuberetes architecture
With kubernetes, we also can manage an auto-scalable web app easily.

### Load Balancer
k8s exposes the service externally using a cloud provider’s load balancer. Traffic from the external load balancer will be directed at the backend Pods. the AWS Classic Load Balancers that receives the request selects a registered instance or pod using the round robin routing algorithm for TCP listeners and the least outstanding requests routing algorithm for HTTP and HTTPS listeners.

### Routing
K8s ingress controller can redirect the request for each API to specific k8s service/pod.
![](https://raw.githubusercontent.com/dogzzdogzz/web-app-architecture/master/ingress-controller.png)

By using Istio service mesh, we can have more flexibility for traffic routing management like canary deployment.
![](https://raw.githubusercontent.com/dogzzdogzz/web-app-architecture/master/istio-TrafficManagementOverview.jpg)

### Sticky Session Handling
You can guarantee session affinity with k8s services, so each customer, when gets back to hit your service, will be redirected to the same pod.

This would be the yaml file of the service:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  sessionAffinity: ClientIP
```

If you want to specify time, as well, this is what needs to be added:

```yaml
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 60
```

Note that the example above would work hitting ClusterIP type service directly (which is quite uncommon), or with Loadbalancer type service, but won't with an Ingress behind NodePort type service.
This is because with an Ingress, the requests come from many, randomly chosen source IP addresses.

With an ingress, we have to enable using cookies to achieve session affinity.
- [Nginx](https://kubernetes.github.io/ingress-nginx/examples/affinity/cookie/)
- [Traefik](https://docs.traefik.io/configuration/backends/kubernetes/)

### Auto scaling
Create several AWS auto scaling groups at the beginning:
- On-demand instance for stateful app (e.g. DB).
- Spot instance of appropriated EC2 type for stateless app (e.g. t2/t3 for burstable app, m5 for memory intensive app).

![](https://raw.githubusercontent.com/dogzzdogzz/web-app-architecture/master/aws-auto-scaling.png)

K8S can create/terminate pod replications with HPA (Horizontal Pod Autoscaler) and create/terminate ec2 instances with CA (Cluster Autoscaler) automatically by monitoring some custom metrics.

Some of metrics can help us to determine if the app need to be scaled out, for example:
- CPU usage
- Memory usage
- Thread usage
- Heap
- Disk IOPS
- Network bandwidth usage
- Application Service Quality (e.g. latency)
- Application Load (e.g. transactions-per-sec, requests-per-sec, network throughput)
- Error rate

### Health check
K8s provides the readinessProbe and livenessProbe function to do health check periodically, the pod will be killed and restart a new one if health check failed.

```yaml
readinessProbe:
    httpGet:
      path: /liveness-probe
      port: http
    initialDelaySeconds: 3
    periodSeconds: 15
livenessProbe:
    httpGet:
      path: /liveness-probe
      port: http
    initialDelaySeconds: 3
    periodSeconds: 15
```

### Monitoring
- Use prometheus stack
  - Various exporters for metrics collection + alert manager + Grafana for metrics monitoring
  - Prometheus: Data storage, data aggregation, etc.
  - Grafana: Data visualiztion
  - Alert Manager: Push alert to E-mail, SMS, Slack, etc when hit threshold
  
- Use ELFK stack or cloudwatch for logging
  - Filebeat: Log collection
  - Logstash: Data processing
  - Elasticsearch: log storage
  - Kibana: log visualization

### Security Settings
- EC2 instance security group:
  - 443, 1025-65535: Allowed for k8s master.
  - 22: Allowed for bastion instance to troubleshoot.
- k8s: only open needed port.
```yaml
kind: Deployment
ports:
- name: http
  containerPort: 80
--
kind: Service
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: http
  selector:
    app: webapp
```