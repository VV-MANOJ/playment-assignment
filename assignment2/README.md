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













































**Revision Notes**


I have tested entire flow in my kubernetes cluster, with the sample spring boot application. Reference https://github.com/learnk8s/spring-boot-k8s-hpa

Steps to setup:

```
git clone https://github.com/learnk8s/spring-boot-k8s-hpa
cd spring-boot-k8s-hpa
TO deploy metric-service: kubectl create -f monitoring/metrics-server
TO deploy namespace: kubectl create -f monitoring/namespaces.yaml
TO deploy prometheus: kubectl create -f monitoring/prometheus
TO deploy customapi metrics: kubectl create -f monitoring/custom-metrics-api
YOu can see the metrics here : kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .
Run the deployment :
 kubectl create -f kube/deployment
 This basically deploys a frontend, backend and queue.
 To see the metrics: kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/messages" | jq .

TO configure the HPA: kubectl create -f kube/hpa.yaml

```

we  have configured something like below in the code to expose `/metrics`

    @RequestMapping(value="/metrics", produces="text/plain")
    public String metrics() {
        int totalMessages = queueService.pendingJobs(queueName);
        return "# HELP messages Number of messages in the queueService\n"
                + "# TYPE messages gauge\n"
                + "messages " + totalMessages;
    }


The application has a /metrics endpoint to expose the number of messages in the queue.
curl http://endpoint/metrics -> Will give output like below
messages 30


To scale the pods we use below HPA in the backend service: 

We are using  the messages metric to scale your Pods. Kubernetes will trigger the autoscaling when there're more than ten messages in the queue.
Kubernetes watches the deployment specified in scaleTargetRef. In this case, it's the worker.


```
1. Does rabbitMQ provide /metrics endpoint out of the box or do we have to do any customization there ?

Yes rabbitmq provides the metrics in GET /api/overview. IT will return the json format. We can get all the below details from the json.
Cluster-name	cluster_name
Total-no-of-connections	object_totals.connections
Total-no-of-channels	object_totals.channels
Total-no-of-queues	    object_totals.queues
Total-no-of-consumers	object_totals.consumers
Total-no-of-messages  	queue_totals.messages
```
```
2. When it comes to a custom metrics server,  it looks like we have to build it , then deploy it and send the custom metrics via an api call to it.  Only then the autoscaler will be able to read the custom metrics.  How do we push the RabbitMQ queue size to custom metrics server ?

By default when you deploy cutom-metric-server you get this metricnames:
namespaces/messages
pods/messages

Below command is used to get metrics of messages from all the pods.
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/messages" | jq .
```


```
3. How to expose the length of  the queue as a metric for the autoscaler to watch ?
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: spring-boot-hpa
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

```

```
4. Any open source apps or tools that can be leveraged here?
We can leverage this by using kubernetes autoscaler using Sysdig’s custom metrics(Not tested).Blog: https://sysdig.com/blog/kubernetes-autoscaler/
```



