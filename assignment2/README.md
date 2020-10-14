***Overview***: 
1. Assuming we three services: 
      
        producer - frontend 
        
        queueprocessor - rabbitmq
        
        consumer - backend
2. If the frontend is producing more messages than what the backend can handle, those messages are buffered in the rabbitmq.
3. We have some admin panel to see RabbitMQ pending messages. At the same time we should also enable /metric endpoint to expose the number of pending messages in the queue

***Configuring kubernetes to scale your application automatically***:

1. The autoscaler works by monitoring metrics server.
2. Only then it can increase or decrease the instances of your application.
3. So you could expose the length of the queue as a metric and ask the autoscaler to watch that value.
4. With the autoscaler enabled, the more pending messages in the queue, the more instances of your application Kubernetes will create.
5. The application has a /metrics endpoint to expose the number of messages in the queue.
6. The format is plain text which is understandable by prometheus

***Installation steps for custom metrics***:
1. We needed to install the Custom Metrics API :<https://github.com/kubernetes-sigs/custom-metrics-apiserver>
2. kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq . -&gt; Will return the list of custom metrics.
3. We have an application exposing metrics and a metric server consuming them. to check that
4. kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/\*/messages" | jq .


***HPA Configuration to read the custom metrics***:
```
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
    -   type: Pods
        pods:
          metricName: messages
          targetAverageValue: 10
```

***Scaling results:***
1. Kubernetes watches the deployment specified in scaleTargetRef.
2. We messages metric to scale your Pods. Kubernetes will trigger the autoscaling when there're more than ten messages in the queue.
3. Also, every scale-up is re-evaluated every minute, whereas any scale down every two minutes.
4. Cluster autoscaler can also help us here if we don't have enough capacity in the cluster to scale your Pods.

