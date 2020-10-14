ssuming we three services:
  producer          - frontend
  queueprocessor    - rabbitmq
  consumer          - backend
If the frontend is producing more messages than what the backend can handle, those messages are buffered in the rabbitmq.
We have some admin panel to see RabbitMQ pending messages.At the same time we should also enable /metric  endpoint to expose the number of pending messages in the queue
Configuring kubernetes to scale your application automatically:
The autoscaler works by monitoring metrics server.
Only then it can increase or decrease the instances of your application.
So you could expose the length of the queue as a metric and ask the autoscaler to watch that value.
With the autoscaler enabled, the more pending messages in the queue, the more instances of your application Kubernetes will create.
The application has a /metrics endpoint to expose the number of messages in the queue.
The format is plain text which is understandable by prometheus
Installation steps:
We  needed to install the Custom Metrics API :https://github.com/kubernetes-sigs/custom-metrics-apiserver
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq . -> Will return the list of custom metrics.
We have an application exposing metrics and a metric server consuming them. to check that
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/messages" | jq .
HPA Configuration to read the custom metrics:
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: backend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metricName: messages
      targetAverageValue: 10
Kubernetes watches the deployment specified in scaleTargetRef. 
We messages metric to scale your Pods. Kubernetes will trigger the autoscaling when there're more than ten messages in the queue.
Also, every scale-up is re-evaluated every minute, whereas any scale down every two minutes.
Cluster autoscaler can also help us here if we  don't have enough capacity in the cluster to scale your Pods.
