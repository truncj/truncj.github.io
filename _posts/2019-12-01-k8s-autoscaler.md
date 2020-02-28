---
layout: post
title: Building a K8s Autoscaler with Custom Metrics
feature-img: "assets/img/blog/k8s_autoscaler/k8s_autoscaler.jpg"
thumbnail: "assets/img/blog/k8s_autoscaler/k8s_autoscaler.jpg"
image: "assets/img/blog/k8s_autoscaler/k8s_autoscaler.jpg"
---

When we talk about the benefits of Kubernetes with clients, we would be remiss not to mention the ability to scale your application. Often customers will mention how migrating their applications to Kubernetes will solve many of their problems, including automatic scaling. Those features certainly exist as a part of the platform, but there are many implementation details and caveats. 
Let’s look at an example and assume the following about your application

1.	Your application is a stateless microservice named “ConsumerApp”
2.	Your application consumes messages off of a RabbitMQ queue
3.	Your application can process a fixed number of messages over an amount of time (eg. 100msg/min)

Our goal is to automatically scale the “ConsumerApp” based off of the average length of the RabbitMQ queue. This seems like it would be a relatively trivial task considering the advertised features of Kubernetes. In the OSS community, you can be inundated with different types of solutions to the same problem. You may find something that appears to fit the bill, but it’s important to spend some time evaluating solutions before intertwining them with your production workloads. With such a fast paced ecosystem it can be hard to tell what is best-practice. For that reason, I want to show you how you can achieve the desired outcome using a production recommended pattern instead of an all-in-one solution.
We want to implement a robust, production-ready and standardized autoscaling design for our applications. In order to achieve this, let’s start with Prometheus metrics as the source of data for our autoscaling logic. Since Kubernetes users are most likely already running Prometheus (whether for monitoring, as a part of Istio, etc..) we have an opportunity to reuse our tooling. Normally Prometheus would be able to scrape metrics directly from RabbitMQ. Unfortunately, RabbitMQ’s metrics endpoint does not contain “queue length” metrics. In order to gather that data, we employ the use of RabbitMQ Exporter. Once we get it connected to RabbitMQ, we’ll have a plethora of RabbitMQ metrics that we can use as a base for our scaling. Prometheus is able to scrape those metrics through the use of a ServiceMonitor and then stores them in its time-series database. 


<p align="center">
  <img src="/assets/img/blog/k8s_autoscaler/architecture.png" width="75%" title="architecture">
</p>
<p align="center">
  <i>Figure 1. High level autoscaler architecture </i>
</p>

Kubernetes supports the use of Horizontal Pod Autoscalers (HPA). These HPAs are Kubernetes native resources that are normally used to scale applications based on resource consumption (CPU, memory, or network). Since utilization based scaling isn’t what we need and HPAs are unable to query directly against prometheus for these metrics, there’s a need for the Prometheus Adapter. We have the choice to create either custom or external metrics using this metrics adapter. Since our metrics are not based on the Kubernetes native resources, we’ll create external metrics based on this definition.

```yaml
rules:
    default: false
    external:
      - seriesQuery: '{__name__=~"^rabbitmq_queue_.*"}'
        resources:
          template: <<.Resource>>
        metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[1m])'
```

Any metric from prometheus that begins with “rabbitmq_queue” will be available in the form of a rate over a 1 minute interval through this new external.metrics.k8s.io API.

<p align="center">
  <img src="/assets/img/blog/k8s_autoscaler/detailed_architecture.png" width="75%" title="detailed architecture">
</p>
<p align="center">
  <i>Figure 2. Detailed autoscaler architecture</i>
</p>

The adapter translates the HPA’s API requests into PromQL to gather the metrics needed. In order to verify if your HPA is able to get the appropriate Prometheus metrics, you can simulate an API request using the “–raw” query listed below.

Query:

```shell
kubectl get --raw "/apis/external.metrics.k8s.io/v1beta1/namespaces/apps/rabbitmq_queue_messages_ready?labelSelector=queue%3dabcqueue,service%3dprometheus-rabbitmq-exporter"
```

Result:

```yaml
"metricName": "",
"metricLabels": {
	"durable": "true",
	"endpoint": "publish",
	"instance": "1.2.3.4:9419",
	"job": "prometheus-rabbitmq-exporter",
	"namespace": "applications",
	"pod": "prometheus-rabbitmq-exporter-abcdef-ghijk",
	"queue": "abcqueue",
	"service": "prometheus-rabbitmq-exporter",
	"vhost": "/"
},
"value": "10m"
```

The value shown above indicates that there are an average of 10 messages over the past 1 minute in the “abcqueue” RabbitMQ queue. Now that we know our external metric is functional, we can create our own HPA resource for our application that references the values in our sample raw api request.

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: ConsumerApp-autoscaler
  namespace: applications
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    Name: ConsumerApp-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages_ready
          selector:
            matchLabels:
              service: prometheus-rabbitmq-exporter
              queue: abcqueue
        target:
          type: Value
          value: 100
```

With this HPA created, our application will now scale as follows:
1.	It will create at most 10 replicas in our ConsumerApp-deployment
2.	It will watch the queue: “abcqueue” from service: “prometheus-rabbitmq-exporter”
3.	Our external metric is called “rabbitmq_queue_messages_ready” and it is the rate of messages over a 1 minute interval
4.	For every 100 messages/min, our HPA will scale up our application to N+1 replicas

Though this HPA implementation may seem overly-complicated for the task at hand, it’s important to remember why we took this route. With the architecture defined above, we’ve created Kubernetes-native autoscaling that is both robust and reusable. This design can handle a large amount of deployments and is highly-available since it uses the Kubernetes controller-manager. We can define new autoscaling metrics for future applications with low effort since we’re able to reuse all of these components. Additionally, we have the ability to scale based on predictive metrics by writing more advanced queries for Prometheus. Share some of your experiences of doing a project “right” instead of just “fast”.
