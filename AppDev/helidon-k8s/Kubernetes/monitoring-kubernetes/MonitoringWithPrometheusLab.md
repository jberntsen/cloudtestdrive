[Go to Overview Page](../Kubernetes-labs.md)

![](../../../../common/images/customer.logo2.png)

# Migration of Monolith to Cloud Native

## C. Deploying to Kubernetes
## 2. Monitoring with Prometheus

### **Introduction**

Monitoring a service in Kuberneties involves three components

#### Generating the monitoring data.
This is mostly done by the service itself, the metrics capability we created when building the Helidon labs are an example of this.

Core Kubernetes services may also provide the ability to generate data, for example the Kubernetes DNS service can report on how many lookups it's performed.

#### Capturing the data
Just because the data is available something needs to extract it from the services, and store it. 

#### Processing and Visualizing the data
Once you have the data you need to be able to process it to visualize it and also report any alerts of problems.



### Monitoring and visualization software
We are going to use a very simple monitoring, we will use the metrics in our microservcies and the standard capabilities in the Kubernetes core services to generate data, then use the Prometheus to extract the data and Grafana to display it.

These tools are of course not the only ones, but they are very widely used, and are available as Open Source projects.

#### Namespace for the monitoring and visualization software
So separate the monitoring services from the  other services we're going to put them into a new namespace. Type the following to create it.

```
$ kubectl create namespace monitoring
namespace/monitoring created
```



### Prometheus

Fortunately for us installing Prometheus is simple, we just use helm. Helm will install the most recent version of the chart, however the most recent version of Prometheus does not support Kubernetes 1.13, so if your provider does not support Kubernetes 1.14 or later then you need to specify a version using --version 9.1.0.

if you are using Kubernetes 1.14 then helm charts up to 9.7.1 at least (the current chart version at the time of writing) are supported, so let's just install that. (Irritatingly I can't find an easy way to determine what chart versions are supported on what Kubernetes version)

```
$ helm3 install prometheus stable/prometheus --namespace monitoring
NAME: prometheus
LAST DEPLOYED: Mon Dec 30 13:25:23 2019
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.monitoring.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.monitoring.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace monitoring port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

Note the name given to the Prometheus server within the cluster, in this case `prometheus-server.monitoring.svc.cluster.local`  and also the alert manager's asigned name, in this case `prometheus-alertmanager.monitoring.svc.cluster.local`

The Helm chart will automatically create a couple of small persistent volumes to hold the data it captures. If you want to see more on the volume in the dashboard (namespace monitoring) look at the Config and storage section / Persistent volume claims section, chose the prometheus-server link to get more details, then to locate the volume in the storage click on the Volume link in the details section) Alternatively in the Workloads / Pods section click on the prometheus server pod and scroll down to see the persistent volumes assigned to it.

One key point is that Prometheus itself does not provide any login or authentication mechanism to access the UI. Because of this we do not want to expose it without security measures to the public internet using an Ingress or load balancer (neither of which apply authentication rules, but delegate that to the underlying services.)  

If you are running on a private cluster and (e.g. docker desktop on a laptop) and don't mind Prometheus (and Gafana) being openly accessible in the helidon-kubernetes/monitoring-kubernetes folder there ia a yaml file and scripts to create services behind a load balancer, the ports will be the same as discussed below. ***DO NOT*** use these it you are using a Kubernetes cluster that's connected to the public internet.

It is of course perfectly possible to place a reverse proxy in front of the Prometheus installation and have that do the security

Unless you've decided to setup a load balancer for the purposes of this lab we're just going to setup a port-forwarding using kubectl

Open up a ***new*** terminal and run the following commands

```
$  export POD_NAME=$(kubectl get pods --namespace monitoring -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
$  kubectl --namespace monitoring port-forward $POD_NAME 9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090

```

Now check it's all working by going to `http://localhost:9090/graph`in a web browser. You should see the Prometheus graphs page.

![Prometheus empty graphs page](images/prometheus-empty-graphs.png)

Let's check that Prometheus is scraping some data. Click the "Insert Metric At Cursor" button, you will see a *lot* of possible choices exploring the various servcies built into Kubernetes (Including apiserver, Core DNS, Container stats, the number of various kubernetes objects like secrets, pods, configmaps and so on.)

In the dropdown select `http_requests_total` you may have to chose the `promtetheus_http_requests_total` option if you can't find the plain http version (this depends on what the platform offers), then click the Execute button. 

Alternatively rather than selecting frm the list you can just start to type `http_requests_total` into the Expression box, as you type a drop down will appear showing the possible metrics that match your typing so far, once the list of choices is small enough to see it chose `http_requests_total` from the list (or just finish typing the entire name and press return to select it) 

You will be presented with either console text data

![prometheus-http-requests-total-console](images/prometheus-http-requests-total-console.png)

or a graph
![prometheus-http-requests-total-graph](images/prometheus-http-requests-total-graph.png)

Click the "Graph" or "Console" buttons to switch between them

Note that the precise details shown will of course vary, especially if you've only recently started Prometheus.
and non stacked modes. Click the + and - buttons next to the duration (default is 1 hour) to expand or shrink the time window of the data displayed, use the << and >> buttons to move the time window around within the overall data set (of course these may not be much use if you haven't got much data, but have a play if you like)

### Specifying services to scrape
The problem we have is that (currently) Prometheus is not collecting (scraping) any data from our services. Of course we may find info on the clusters behavior interesting, but out own services would be more interesting !

We can see what services Prometheus is currently scraping by clicking on the Status menu (top of the screen) 

![Choosing service discovery](images/prometheus-chosing-service-discovery.png)

Then selecting Service Discovery

![The initial service discovery screen](images/prometheus-service-discovery-initial.png)

Click on the "Show Me" button next to the Kubernetes-pods line (this is the 2nd reference to kubernetes pods, the 1st is just a link that takes you to the 2nd one it it's not on the screen)

You will be presented with a list of all of the pods in the system and the information that Prometheus has gathered about them (it does this by making api calls to the api server in the same way kubectl does)

![prometheus-pods-lists-initial](images/prometheus-pods-lists-initial.png)

If you scroll down (there may be a lot of scrolling) you'll find the entries for the storefront and stockmanager pods. If the pod is exposing multiple ports there may be multiple entries for the same pod as Prometheus is checking every port it can to see if there are metrics available. (The image below shows the entries for the storefront on ports 8080 and 9080)

![prometheus-pods-storefront-initial](images/prometheus-pods-storefront-initial.png)

We know that Prometheus can see our pods, the question is how do we get it to scrape data from them ?

The answer is actually very simple. In the same way that we used annotations on th Ingress rules to let the Ingress controller know to scan them we use annotations on the pod descriptions to let Prometheus know to to scrape them (and how to do so)

We can set annotations on a pod using the yaml descriptions, but first let's look at how we might use kubectl. (When we do it for real we'll update the deployment yaml files) One point about this approach though, using kubectl is making changes to a *running* pod, that's not a problem but it does mean that if the pod exits that the changes will be lost. Under normal circumstances you'd want these to be persistent and would make the annotation changes to the pod definition in the deployment yaml file, it's only if you need to make changes on a temporary basis that you'd make changes with the kubectl annotation mechanism.

We can use kubectl to get the pod id, using a label selector to get pods with a label app and a value storefront, then we're using jsonpath to get the specific details from the JSON output here
don't actually do this, this is just to show how you could modify a live pod with kubectl

```
$ kubectl get pods -l "app=storefront" -o jsonpath="{.items[0].metadata.name}"
storefront-588b4d69db-w244b
```

Now we could use kubectl to add a few annotations to the pod (this is applied immediately, and Prometheus will pick up on it in a few seconds)

```
$ kubectl annotate pod storefront-588b4d69db-w244b prometheus.io/scrape=true --overwrite
pod/storefront-588b4d69db-w244b annotated
$ kubectl annotate pod storefront-588b4d69db-w244b prometheus.io/path=/metrics --overwrite
pod/storefront-588b4d69db-w244b annotated
$ kubectl annotate pod storefront-588b4d69db-w244b prometheus.io/port=9080 --overwrite
pod/storefront-588b4d69db-w244b annotated
```

We can see what annotations there would be on the pod with kubectl (this works even if you setup the annotations using the deployment yaml file.)

```
$  kubectl get pod storefront-588b4d69db-w244b -o jsonpath="{.metadata..annotations}"
map[prometheus.io/path:/metrics prometheus.io/port:9080 prometheus.io/scrape:true]
```

If you want to see how to script temporary changes then in the helidon-kubernetes/monitoring-kubernetes folder look at the script setupAnnotations.sh

***However***
In most cases we don't want these to be a temporary change, we want the Prometheus to monitor our pods if they restart (or we re-deploy)

In the helidon-kubernetes folder edit the storefront-deployment.yaml file, look for the pod annotations (part of the spec / template / metadata section) which are currently commented out

```
   metadata:
      labels:
        app: storefront
#      annotations:
#        prometheus.io/path: /metrics
#        prometheus.io/port: "9080"
#        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: storefront
```

Remove the comment symbol (the #) in front of the annotations so the section now looks like

```
   metadata:
      labels:
        app: storefront
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9080"
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: storefront
```
Be ***very*** careful to only remove the # character and no other whitespace.

Make the same modifications to the stockmanager-deployment.yaml file so it now has a pod annotations section that looks like 

```
 template:
    metadata:
      labels:
        app: stockmanager
      annotations:
        prometheus.io/path: /metrics
        prometheus.io/port: "9081"
        prometheus.io/scrape: "true"
    spec:
      containers:
      - name: stockmanager
```

In the helidon-kubernetes folder run the undeploy.sh script to remove the deployments

``` 
$ ./undeploy.sh 
Deleting storefront deployment
deployment.extensions "storefront" deleted
Deleting stockmanager deployment
deployment.extensions "stockmanager" deleted
Deleting zipkin deployment
deployment.extensions "zipkin" deleted
Kubenetes config is
NAME                               READY   STATUS    RESTARTS   AGE
pod/stockmanager-bd44bbbb7-qk6g9   1/1     Running   0          8h
pod/storefront-564dd99db9-fzf8s    1/1     Running   0          8h
pod/zipkin-88c48d8b9-tjvcg         1/1     Running   0          8h

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.156.12    <none>        8081/TCP,9081/TCP   8h
service/storefront     ClusterIP   10.110.88.187    <none>        8080/TCP,9080/TCP   8h
service/zipkin         ClusterIP   10.101.129.223   <none>        9411/TCP            8h

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-bd44bbbb7   1         1         1       8h
replicaset.apps/storefront-564dd99db9    1         1         1       8h
replicaset.apps/zipkin-88c48d8b9         1         1         1       8h
```
This script just does a kubectl delete -f on each of the deployments. 

Then in the same helidon-kubernetes folder run the deploy.sh script to re-create them

```
$ ./deploy.sh 
Creating zipkin deployment
deployment.extensions/zipkin created
Creating stockmanager deployment
deployment.extensions/stockmanager created
Creating storefront deployment
deployment.extensions/storefront created
Kubenetes config is
NAME                               READY   STATUS              RESTARTS   AGE
pod/stockmanager-d6cc5c9b7-f9dnf   0/1     ContainerCreating   0          1s
pod/storefront-588b4d69db-vnxgg    0/1     ContainerCreating   0          1s
pod/zipkin-88c48d8b9-x8b6t         0/1     ContainerCreating   0          1s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.156.12    <none>        8081/TCP,9081/TCP   8h
service/storefront     ClusterIP   10.110.88.187    <none>        8080/TCP,9080/TCP   8h
service/zipkin         ClusterIP   10.101.129.223   <none>        9411/TCP            8h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   0/1     1            0           1s
deployment.apps/storefront     0/1     1            0           1s
deployment.apps/zipkin         0/1     1            0           1s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-d6cc5c9b7   1         1         0       1s
replicaset.apps/storefront-588b4d69db    1         1         0       1s
replicaset.apps/zipkin-88c48d8b9         1         1         0       1s
```

This script just does a kubectl apply -f on each of the deployments. 

If we use kubectl to get the status after a short while we'll see everything is working as expected and the pods are Running

```
$ kubectl get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/stockmanager-d6cc5c9b7-f9dnf   1/1     Running   0          79s
pod/storefront-588b4d69db-vnxgg    1/1     Running   0          79s
pod/zipkin-88c48d8b9-x8b6t         1/1     Running   0          79s

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.156.12    <none>        8081/TCP,9081/TCP   8h
service/storefront     ClusterIP   10.110.88.187    <none>        8080/TCP,9080/TCP   8h
service/zipkin         ClusterIP   10.101.129.223   <none>        9411/TCP            8h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           79s
deployment.apps/storefront     1/1     1            1           79s
deployment.apps/zipkin         1/1     1            1           79s

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-d6cc5c9b7   1         1         1       79s
replicaset.apps/storefront-588b4d69db    1         1         1       79s
replicaset.apps/zipkin-88c48d8b9         1         1         1       79s
```

(We are doing an undeploy and deploy to acoud confusion with different replica sets. is we'd just re-done the deploy Kuberneties would have acted like it was an upgrade of the deployments and we'd have seen pods terminating, while new ones were being created and additional replica sets. We'll look into how this actually works in detail in a later lab)

If you return to the browser and reload the Prometheus page we'll see that there are 2 pods showing as being discovered, previously it was 0

![prometheus-pods-list-updated](images/prometheus-pods-list-updated.png)

If we now click on the "show more" button net to the kubernetes-pods label we'll see that the storefront pod (port 9080) and stockmanager (port 9081) pods are no longer being dropped and there is now data in the target labels column. The actual data services for storefont (port 8080) and stockmanager (port 8081) are however still dropped.

![prometheus-pods-storefront-updated](images/prometheus-pods-storefront-updated/png)

### Let's look at our captured data
Now we have configured Prometheus to scrape the data from our services we can look at the data it's been capturing.

Firstly return to the Graph page in the Prometheus web page, just click on the Graph button at the top of the page.

In the Expression box use the auto complete function by typing the start of `application:list_all_stock_meter_total` or alternatively select it from the list under the "Insert metric at cursor" button.

Once it's entered click the Execute button.

If you're on the graph screen you'll probabaly see a pretty boring graph

![prometheus-list-stock-empty](images/prometheus-list-stock-empty-graph.png)

The console view however provides us with a bit more information

![prometheus-list-stock-empty-console](images/prometheus-list-stock-empty-console.png)

If we look at the data we can see that the retrieved value is 0 (it may be another number, depends on how often you made the call to list the stock in previous labs) of course our graph looks boring, since we setup prometheus we haven't actually done anything)

Let's make a few calls to list the stock and see what we get

Execute the following a few times, **replacing *localhost* with your Ingress endpoint**

```
$ curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Mon, 30 Dec 2019 16:58:26 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

If we now go back to the Prometheus browser window and reload the page (or reselect the metric `application:list_all_stock_meter_total`) we see that our changes have been applied (Note it may take up to 60 seconds for Prometheus to get round to scraping the metrics form the service, it doesn't do it continuously as that may put a significant load on the services it's monitoring)

In the console we can see the number of requests we have made

![prometheus-list-stock-requested-console](images/prometheus-list-stock-requested-console.png)

And in the Graph we can see the data history

![prometheus-list-stock-requested-graph](images/prometheus-list-stock-requested-graph.png)


(Note in this case I made a set of 10 requests, then waited a couple of mins and made another 5 requests, I have also adjusted the graph so it's displaying data over a 15 min period, rather than the default of 1 hour which makes it easier to see the calls made)

When we did the Helidon labs we actually setup the metrics on the call to list stock to not just generate an absolute count of calls, but to also calculate how man calls were begin made over a time interval. If we change the metric from `application:list_all_stock_meter_total` to `application:list_all_stock_meter_one_min_rate_per_second` we can see the number of calls per second averaged over 1 min. This basically provides us with a view of the number of calls made to the system overtime, and we can use it to identify peak loads.

![prometheus-list-stock-requested-graph-rat-per-sec-one-min](images/prometheus-list-stock-requested-graph-rat-per-sec-one-min.png)

Prometheus can also produce multi value graphs. For example in addition to the counting metrics we also setup a timer on the listALlStock method to see how long calls to it took. If we now generate a graph on the timer we see in the Console view that instead of just seeing a single entry representing the latest data, that there are actually 6 entries representing different breakdowns of the data (0.5 being the most common data, 0.999 being the least common) Of course the data you see may vary depending on your situation and how much you've already been using the services.

![prometheus-list-stock-timer-quantile-console](images/prometheus-list-stock-timer-quantile-console.png)

Most of the time the requests are in the 0.165 to 0.172 seconds range, but there was one grouping which took over 11 seconds. This could be a cause for concern (in actually the first request always takes a lot longer because Helidon REST services and Database accesses are configured on the first request, and I had re-started the pods since the basic labs so one of my requests was a first access)

The graph is also a lot more interesting (you may have to scroll your Prometheus browser window to see all of the graph and legend information)

![prometheus-list-stock-timer-quantile-graph](images/prometheus-list-stock-timer-quantile-graph.png)

It's not possible to show in a static window but as you move your mouse over the legend the selected data is highlighted, and if you click on a line in the legend only that data is displayed.

Prometheus has a number of mathematical functions we can apply to the graphs it produces, these are perhaps not so much use if there's only a single pod servicing requests, but if there are multiple pods all generating the same statistics (perhaps because of a replica set providing multiple pods to a service for horizontal scaling) then when you gather information such as call rates (the  `application:list_all_stock_meter_one_min_rate_per_second` metric) instead of just generating and displaying the data per pod you could also generate data such as `msum(application:list_all_stock_meter_one_min_rate_per_second)` which would tell you the total number of requests across ***all*** the pods providing the service.

It's also possible to do things like separate out pods that are being used for testing (say they have a deployment type of test rather than production) or many other parameters. If you want more details there is a link to the Prometheus Query language description in the further-information document

### But it's not a very good visualization
Prometheus was not designed to be a high end graphing tool, the graphs cannot for instance be saved so you can get back to them later. For that we need to move on to the next lab and have a look at the capabilities of Grafana





---

You have reached the end of this lab !!

Use your **back** button to return to the **C. Deploying to Kubernetes** section