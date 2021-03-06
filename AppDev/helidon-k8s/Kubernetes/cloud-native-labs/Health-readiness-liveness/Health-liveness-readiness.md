[Go to Overview Page](../Kubernetes-labs.md)

![](../../../../common/images/customer.logo2.png)

# Migration of Monolith to Cloud Native

## C. Deploying to Kubernetes
## 4. Health, Readiness and Liveness

### **Introduction**

Kubernetes provides a service that monitors the pods to see if they meet the requirements in terms of running, being responsive, and being able to process requests. 

A core feature of Kubernetes is the assumption that eventually for some reason or another something will happen that means a service cannot be provided, and the designers of Kuberneties made this a core understanding of how it operates. Kubernetes doesn't just set things up they way you request, but it also continuously monitors the state of the entire deployments so that if the system does not meet what was specified Kubernetes steps in and automatically tries to adjust things so it does !

These labs look at how that is achieved.

### Is the container running ?

As we've seen a service in Kubernetes is delivered by programs running in containers. The way a container operates is that it runs a single program, once that program exists then the container exits, and the pod is no longer providing the service. 

So the first thing we need to protect against is the program providing our service existing, there are always bugs in code (with the possible exception of "Hello World" bug free programs just don't exist) and quite often the bugs can cause programs to completely crash.

Before we can see this in action, we need to get a bit of data on the current state of the system.

First let's make sure that the service is running. In a terminal type

```
$ curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 14:01:18 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

OK, all is running fine, in a terminal type 

```
$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-rl8wg   1/1     Running   0          92m
storefront-65bd698bb4-cq26l     1/1     Running   0          92m
zipkin-88c48d8b9-jkcvq          1/1     Running   0          92m
```

We can see the state of our pods, look at the RESTARTS column, all of the values are 0 (if there was a problem, for example Kubernetes could not download the container images you would see a message in the Status column, and possibly additional re-starts, but here everything is A-OK.)

We're going to simulate a crash in our program, this will cause the container to exit, and Kubernetes will idntify this and start a replacement container for us.

Using the name of the storefront pod above let's connect to the container in the pod using kubectl, and then use ps to see what processes are running in it

```
$ kubectl exec storefront-65bd698bb4-cq26l -ti -- /bin/bash
root@storefront-65bd698bb4-cq26l:/# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 12:23 ?        00:00:29 java -server -Djava.awt.headless=true -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -cp /app/resources:/app/classes:/app/libs/* com.oracle.labs.helidon.storefront.Main
root        47     0  0 14:04 pts/0    00:00:00 /bin/bash
root        53    47  0 14:04 pts/0    00:00:00 ps -ef
```

We can see the bash and ps process we just kicked off, but also the primary process which is running out service, the java process.

We're going to simulate the process having a major fault and exiting by just killing it. In the container type

```
root@storefront-65bd698bb4-cq26l:/# pkill java
root@storefront-65bd698bb4-cq26l:/# command terminated with exit code 137
```

Within a second or two of the process being killed the connection to the container in the pod is terminated as the container exits

If we now try getting the data again it still responds, admittedly with a short delay, how come ?

```
 curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 14:10:04 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

To find out let's look at the pod details again

```
$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-rl8wg   1/1     Running   0          107m
storefront-65bd698bb4-cq26l     1/1     Running   1          107m
zipkin-88c48d8b9-jkcvq          1/1     Running   0          107m
```

Now we can see that the pod names are still the same, but the storefront pod has had a restart.

Kubernetes has identified that the container exited and within the pod restarted a new container. Another indication of this is if we look at the logs we can see that previous activity is no longer displaying a the timestamps for starting the process are all recent.

```
$ kubectl logs storefront-65bd698bb4-cq26l 
2020.01.02 14:06:30 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Starting server
2020.01.02 14:06:32 INFO org.jboss.weld.Version Thread[main,5,main]: WELD-000900: 3.1.1 (Final)
2020.01.02 14:06:32 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-000020: Using jandex for bean discovery
2020.01.02 14:06:33 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] public org.glassfish.jersey.ext.cdi1x.internal.ProcessAllAnnotatedTypes.processAnnotatedType(@Observes ProcessAnnotatedType<?>, BeanManager) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 14:06:33 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] private io.helidon.microprofile.openapi.IndexBuilder.processAnnotatedType(@Observes ProcessAnnotatedType<X>) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 14:06:35 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-002003: Weld SE container 0228d5bd-f6ed-4c4e-9b09-492d44fad6cc initialized
2020.01.02 14:06:35 INFO io.helidon.tracing.zipkin.ZipkinTracerBuilder Thread[main,5,main]: Creating Zipkin Tracer for 'sf' configured with: http://zipkin:9411/api/v2/spans
2020.01.02 14:06:36 INFO io.smallrye.openapi.api.OpenApiDocument Thread[main,5,main]: OpenAPI document initialized: io.smallrye.openapi.api.models.OpenAPIImpl@4db77402
2020.01.02 14:06:38 INFO io.helidon.webserver.NettyWebServer Thread[main,5,main]: Version: 1.3.1
2020.01.02 14:06:38 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel 'admin' started: [id: 0x1e472e66, L:/0.0.0.0:9080]
2020.01.02 14:06:38 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-2,10,main]: Channel '@default' started: [id: 0x8f683428, L:/0.0.0.0:8080]
2020.01.02 14:06:38 INFO io.helidon.microprofile.server.ServerImpl Thread[nioEventLoopGroup-2-1,10,main]: Server started on http://localhost:8080 (and all other host addresses) in 219 milliseconds.
2020.01.02 14:06:38 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Running on http://localhost:8080/store
2020.01.02 14:10:02 WARNING com.netflix.config.sources.URLConfigurationSource Thread[helidon-1,5,server]: No URLs will be polled as dynamic configuration sources.
2020.01.02 14:10:02 INFO com.netflix.config.sources.URLConfigurationSource Thread[helidon-1,5,server]: To enable URLs as dynamic configuration sources, define System property archaius.configurationSource.additionalUrls or make config.properties available on classpath.
2020.01.02 14:10:02 INFO com.netflix.config.DynamicPropertyFactory Thread[helidon-1,5,server]: DynamicPropertyFactory is initialized with configuration sources: com.netflix.config.ConcurrentCompositeConfiguration@7f2faedd
2020.01.02 14:10:02 INFO io.helidon.microprofile.faulttolerance.CommandRetrier Thread[helidon-1,5,server]: About to execute command with key listAllStock-1066291322 on thread helidon-1
2020.01.02 14:10:02 INFO com.oracle.labs.helidon.storefront.resources.StorefrontResource Thread[hystrix-io.helidon.microprofile.faulttolerance-1,5,server]: Requesting listing of all stock
2020.01.02 14:10:04 INFO com.oracle.labs.helidon.storefront.resources.StorefrontResource Thread[hystrix-io.helidon.microprofile.faulttolerance-1,5,server]: Found 5 items
```

The reason it took a bit longer than usual when starting the service is that the code was doing the on-demand setup of the web services.

### Liveness
We now have mechanisms in place to restart a container if it fails, but it may be that the container does not actually fail, just that the program running in it ceases to behave properly, for example there is some kind of non fatal resource starvation such as a deadlock. In this case the pod cannot recognize the problem as the container is still running.

Fortunately Kubernetes provides a mechanism to handle this as well. This mechanism is called Liveness probes, if a pod fails a liveness probe then it will be automatically restarted.

You may recall in the Helidon labs (if you did them) we created a liveness probe, this is an example of Helidon is designed to work in cloud native environments.

In the helidon-kubernetes folder there is the storefront-deployment.yaml file. Let's open that file and search for the Liveness probes section. This is under the spec.template.spec.containers section. (the 

```
        resources:
          limits:
            # Set this to me a whole CPU for now
            cpu: "1000m"
#        # Use this to check if the pod is alive
#        livenessProbe:
#          #Simple check to see if the liveness call works
#          # If must return a 200 - 399 http status code
#          httpGet:
#             path: /health/live
#             port: health-port
#          # Give it a few seconds to make sure it's had a chance to start up
#          initialDelaySeconds: 60
#          # Let it have a 5 second timeout to wait for a response
#          timeoutSeconds: 5
#          # Check every 5 seconds (default is 1)
#          periodSeconds: 5
#          # Need to have 3 failures before we decide the pod is dead, not just slow
#          failureThreshold: 3
        # This checks if the pod is ready to process requests
#        readinessProbe:
```

The first thing to note here is the # at the beginning, it's been commented out. on each line remove the # (only the first one, and only the # character, be sure not to remove any whitespace.) The resulting section should look like this, assuming it does save the file. The stockmanager has similar entries in it, it's not required in the lab, but you can also update those if you like.

```
        resources:
          limits:
            # Set this to me a whole CPU for now
            cpu: "1000m"
        # Use this to check if the pod is alive
        livenessProbe:
          #Simple check to see if the liveness call works
          # If must return a 200 - 399 http status code
          httpGet:
             path: /health/live
             port: health-port
          # Give it a few seconds to make sure it's had a chance to start up
          initialDelaySeconds: 60
          # Let it have a 5 second timeout to wait for a response
          timeoutSeconds: 5
          # Check every 5 seconds (default is 1)
          periodSeconds: 5
          # Need to have 3 failures before we decide the pod is dead, not just slow
          failureThreshold: 3
        # This checks if the pod is ready to process requests
#        readinessProbe:
```

This is a pretty simple test to see if there is a running service, in this case we use the service to make an http get request (this is made by the framework and is done from outside the pod) on the 9080:/health/live url (we know it's on port 9080 as the port definition names it health-port.) There are other types of liveness probe than just get requests, you can run a command in the container itself, or just see if it's possible to open a tcp/ip connection to a port in the container. Of course this is a simple definition, it doesn't look at the many options that are available.

The first thing to say is that whatever steps your actual liveness test does it needs to be sufficient to detect service problems like deadlocks, but also to use as few resources as possible so the check itself doesn't become a major load factor.

Let's look at some of these values.

As it may take a while to start up the container, we specify and initialDelaySeconds of 60, Kubernetes won't start checking if the pod is live until that period is elapsed. If we made that to short then we may never start the container as Kuberneties would always determine it was not alive before the container had a chance to start up properly. Setting it to be to long however could mean non responding service at start up would not be detected in time.

The timeoutSeconds specifies that for the http request  to have failed it could not have responded in 5 seconds. As many http service implementations are initialized on first access we need to chose a value here that it long enough for the framework to do it's lazy initialization.

The periodSeconds is how often Kuberneties will check the container to see if it's alive and responding. This is a balance, especially if the liveness check involved significant resources (e.g. making a RDBMS call) You need to check often enough that a non responding container will be detected quickly, but not check so often that the checking process itself uses to many resources.

Finally the failureThreshold specifies how many consecutive failures are needed before it's deemed to have failed, in this case we need 3 failures to respond

Whatever your actual implementation you need to carefully consider the values above. Get them wrong and your service may never be allowed to start, or problems may not be detected.

Let's remove the deployment and start them again. In the helidon-kuberneties folder run the undeploy.sh script to stop the deployments

```
$ ./undeploy.sh 
Deleting storefront deployment
deployment.extensions "storefront" deleted
Deleting stockmanager deployment
deployment.extensions "stockmanager" deleted
Deleting zipkin deployment
deployment.extensions "zipkin" deleted
Kubenetes config is
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-j6vf9   1/1     Running   0          66m
pod/storefront-dcc76cccb-6ztsf      1/1     Running   1          66m
pod/zipkin-88c48d8b9-7rx6q          1/1     Running   0          66m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   3h52m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   3h52m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            3h52m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       66m
replicaset.apps/storefront-dcc76cccb      1         1         1       66m
replicaset.apps/zipkin-88c48d8b9          1         1         1       66m

```

(you may see some pods and services still showing as running if they haven't shutdown yet)

And now lts's deploy the updated versions. Again in the helidon-kubenetes folder

```
$ ./deploy.sh
Creating zipkin deployment
deployment.extensions/zipkin created
Creating stockmanager deployment
deployment.extensions/stockmanager created
Creating storefront deployment
deployment.extensions/storefront created
Kubenetes config is
NAME                                READY   STATUS              RESTARTS   AGE
pod/stockmanager-6456cfd8b6-29lmk   0/1     ContainerCreating   0          0s
pod/storefront-b44457b4d-29jr7      0/1     Pending             0          0s
pod/zipkin-88c48d8b9-bftvx          0/1     ContainerCreating   0          0s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   3h55m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   3h55m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            3h55m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   0/1     1            0           0s
deployment.apps/storefront     0/1     1            0           0s
deployment.apps/zipkin         0/1     1            0           0s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         0       0s
replicaset.apps/storefront-b44457b4d      1         1         0       0s
replicaset.apps/zipkin-88c48d8b9          1         1         0       0s

```


Let's see how our pod is doing.

Firstly if we use kubectl to get the pods status we will see that they are all running and ready. A liveness probe does not stop the pod from coming up, even if the probe has not yet started being called.

```
$ kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-29lmk   1/1     Running   0          24s
storefront-b44457b4d-29jr7      1/1     Running   0          24s
zipkin-88c48d8b9-bftvx          1/1     Running   0          24s
```

Note that as we have undeployed and then deployed again there are new pods and so the RESTSRTS has been reset.

If we look at the logs for the storefront **before** the liveness probe has started (so before the 60 seconds from container creation) we see that it starts as we expect it to. Run the kubectl command against *my* storefront pod

```
$ kubectl logs storefront-b44457b4d-29jr7 
2020.01.02 16:18:58 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Starting server
2020.01.02 16:19:00 INFO org.jboss.weld.Version Thread[main,5,main]: WELD-000900: 3.1.1 (Final)
2020.01.02 16:19:01 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-000020: Using jandex for bean discovery
2020.01.02 16:19:02 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] public org.glassfish.jersey.ext.cdi1x.internal.ProcessAllAnnotatedTypes.processAnnotatedType(@Observes ProcessAnnotatedType<?>, BeanManager) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 16:19:02 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] private io.helidon.microprofile.openapi.IndexBuilder.processAnnotatedType(@Observes ProcessAnnotatedType<X>) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 16:19:04 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-002003: Weld SE container 53fe34a2-0291-4b72-a00e-966bab7ab2ad initialized
2020.01.02 16:19:04 INFO io.helidon.tracing.zipkin.ZipkinTracerBuilder Thread[main,5,main]: Creating Zipkin Tracer for 'sf' configured with: http://zipkin:9411/api/v2/spans
2020.01.02 16:19:05 INFO io.smallrye.openapi.api.OpenApiDocument Thread[main,5,main]: OpenAPI document initialized: io.smallrye.openapi.api.models.OpenAPIImpl@2282400e
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[main,5,main]: Version: 1.3.1
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel 'admin' started: [id: 0x2d51c995, L:/0.0.0.0:9080]
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-2,10,main]: Channel '@default' started: [id: 0x7331b80e, L:/0.0.0.0:8080]
2020.01.02 16:19:07 INFO io.helidon.microprofile.server.ServerImpl Thread[nioEventLoopGroup-2-2,10,main]: Server started on http://localhost:8080 (and all other host addresses) in 261 milliseconds.
2020.01.02 16:19:07 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Running on http://localhost:8080/store
```

If however the 60 seconds has passed and the liveness call has started we will see calls being made to the status resource, in this case I gathered the log data about 2 mins **after** the pods were started, so about 60 seconds **after** the liveness probe had started running. Run the kubectl command against *my* storefront pod

```
$$ kubectl logs storefront-b44457b4d-29jr7 
2020.01.02 16:18:58 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Starting server
2020.01.02 16:19:00 INFO org.jboss.weld.Version Thread[main,5,main]: WELD-000900: 3.1.1 (Final)
2020.01.02 16:19:01 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-000020: Using jandex for bean discovery
2020.01.02 16:19:02 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] public org.glassfish.jersey.ext.cdi1x.internal.ProcessAllAnnotatedTypes.processAnnotatedType(@Observes ProcessAnnotatedType<?>, BeanManager) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 16:19:02 INFO org.jboss.weld.Event Thread[main,5,main]: WELD-000411: Observer method [BackedAnnotatedMethod] private io.helidon.microprofile.openapi.IndexBuilder.processAnnotatedType(@Observes ProcessAnnotatedType<X>) receives events for all annotated types. Consider restricting events using @WithAnnotations or a generic type with bounds.
2020.01.02 16:19:04 INFO org.jboss.weld.Bootstrap Thread[main,5,main]: WELD-ENV-002003: Weld SE container 53fe34a2-0291-4b72-a00e-966bab7ab2ad initialized
2020.01.02 16:19:04 INFO io.helidon.tracing.zipkin.ZipkinTracerBuilder Thread[main,5,main]: Creating Zipkin Tracer for 'sf' configured with: http://zipkin:9411/api/v2/spans
2020.01.02 16:19:05 INFO io.smallrye.openapi.api.OpenApiDocument Thread[main,5,main]: OpenAPI document initialized: io.smallrye.openapi.api.models.OpenAPIImpl@2282400e
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[main,5,main]: Version: 1.3.1
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-1,10,main]: Channel 'admin' started: [id: 0x2d51c995, L:/0.0.0.0:9080]
2020.01.02 16:19:07 INFO io.helidon.webserver.NettyWebServer Thread[nioEventLoopGroup-2-2,10,main]: Channel '@default' started: [id: 0x7331b80e, L:/0.0.0.0:8080]
2020.01.02 16:19:07 INFO io.helidon.microprofile.server.ServerImpl Thread[nioEventLoopGroup-2-2,10,main]: Server started on http://localhost:8080 (and all other host addresses) in 261 milliseconds.
2020.01.02 16:19:07 INFO com.oracle.labs.helidon.storefront.Main Thread[main,5,main]: Running on http://localhost:8080/store
2020.01.02 16:20:02 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:06 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:11 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:16 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:21 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:26 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:31 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:36 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:41 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:46 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:51 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:20:56 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:21:01 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:21:06 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:21:11 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:21:16 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
```

If we look at *my* pods detailed info we can see it's state is fine

```
$  kubectl describe pod storefront-b44457b4d-29jr7 
<lots of removed text>
Events:
  Type    Reason     Age    From                     Message
  ----    ------     ----   ----                     -------
  Normal  Scheduled  4m30s  default-scheduler        Successfully assigned tg-helidon/storefront-b44457b4d-29jr7 to docker-desktop
  Normal  Pulling    4m29s  kubelet, docker-desktop  Pulling image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal  Pulled     4m28s  kubelet, docker-desktop  Successfully pulled image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal  Created    4m28s  kubelet, docker-desktop  Created container storefront
  Normal  Started    4m28s  kubelet, docker-desktop  Started container storefront
```

It's started and no unexpected events !

Now is the time to explain that "Not frozen ..." text in the status. To enable us to actually simulate the service having a deadlock or resource starvation problem there's a bit of a cheat in the storefront LivenessChecker code.

```
	@Override
	public HealthCheckResponse call() {
		// don't test anything here, we're just reporting that we are running, not that
		// any of the underlying connections to thinks like the database are active

		// if there is a file /frozen then just lock up for 60 seconds
		// this let's us emulate a lockup which will trigger a pod restart
		// if we have enabled liveliness testing against this API
		if (new File("/frozen").exists()) {
			log.info("/frozen exists, locking for " + FROZEN_TIME + " seconds");
			try {
				Thread.sleep(FROZEN_TIME * 1000);
			} catch (InterruptedException e) {
				// ignore for now
			}
			return HealthCheckResponse.named("storefront-live").up()
					.withData("uptime", System.currentTimeMillis() - startTime).withData("storename", storeName)
					.withData("frozen", true).build();
		} else {
			log.info("Not frozen, Returning alive status true, storename " + storeName);
			return HealthCheckResponse.named("storefront-live").up()
					.withData("uptime", System.currentTimeMillis() - startTime).withData("storename", storeName)
					.withData("frozen", false).build();
		}
```

Every time it's called it checks to see it a file names /frozen exists in the root directory of the container. If it does then it will do a delay (about 60 seconds) before returning the response. Basically this means that by connecting to the container and creating the /frozen file we can simulate the container having a problem. The `Not Frozen...` is just text in the log data so we can see what's happening. Of course you wouldn't do this in a production system !

Let's see what happens in this case.

First let's start following the logs of *my* pod. In a new window type

```
$ kubectl logs -f --tail=10 storefront-b44457b4d-29jr7 
2020.01.02 16:24:16 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:21 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:26 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:31 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:36 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:24:41 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
```

In a different terminal we'll log in to the *my* container and create the /frozen file, then wait and see what happened

```
$ kubectl exec -ti storefront-b44457b4d-29jr7 -- /bin/bash
root@storefront-dcc76cccb-6ztsf:/# touch /frozen
root@storefront-dcc76cccb-6ztsf:/# rm /frozen
root@storefront-dcc76cccb-6ztsf:/# command terminated with exit code 137
```

Kubernetes detected that the liveness probes were not responding in time, and after 3 failures it restarted the pod.

In the logs we see the following 

```
2020.01.02 16:25:41 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:46 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:51 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: Not frozen, Returning alive status true, storename My Shop
2020.01.02 16:25:56 INFO com.oracle.labs.helidon.storefront.health.LivenessChecker Thread[nioEventLoopGroup-3-1,10,main]: /frozen exists, locking for 60 seconds
Weld SE container 53fe34a2-0291-4b72-a00e-966bab7ab2ad shut down by shutdown hook
```

Kubectl tells us there's been a problem and a pot has done a restart for us

```
$  kubectl get pods
NAME                            READY   STATUS    RESTARTS   AGE
stockmanager-6456cfd8b6-29lmk   1/1     Running   0          7m50s
storefront-b44457b4d-29jr7      1/1     Running   1          7m50s
zipkin-88c48d8b9-bftvx          1/1     Running   0          7m50s
```

If we look at the deployment events for *my* pod we'll see this

```
$ kubectl describe pod storefront-b44457b4d-29jr7
<lots of removed text>
Events:
  Type     Reason     Age                  From                     Message
  ----     ------     ----                 ----                     -------
  Normal   Scheduled  8m13s                default-scheduler        Successfully assigned tg-helidon/storefront-b44457b4d-29jr7 to docker-desktop
  Warning  Unhealthy  57s (x3 over 67s)    kubelet, docker-desktop  Liveness probe failed: Get http://10.1.0.170:9080/health/live: net/http: request canceled (Client.Timeout exceeded while awaiting headers)
  Normal   Killing    57s                  kubelet, docker-desktop  Container storefront failed liveness probe, will be restarted
  Normal   Pulling    56s (x2 over 8m12s)  kubelet, docker-desktop  Pulling image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal   Pulled     56s (x2 over 8m11s)  kubelet, docker-desktop  Successfully pulled image "fra.ocir.io/oractdemeabdmnative/tg_repo/storefront:0.0.1"
  Normal   Created    56s (x2 over 8m11s)  kubelet, docker-desktop  Created container storefront
  Normal   Started    55s (x2 over 8m11s)  kubelet, docker-desktop  Started container storefront
```

The pod became unhealthy, then the container was killed and a new container restarted

### Readiness
The first two probes determine if a pod is alive and running, but it doesn't actually report if it's able to process events. That can be a problem if for example a pod has a problem connecting to a backend service, perhaps there is a network configuration issue and the pods path to a back end service like a database is not available.

In this situation restarting the pod / container won't do anything useful, it's not a problem with the container itself, but something outside the container, and hopefully once that problem is resolved the front end service will recover (it's it's been properly coded and doesn't crash, but in that case one of the other health mechanisms will kick in and restart it) **BUT** there is also no point in sending requests to that container as it can't process them.

Kubernetes supports a readiness probe that we can call to see is the container is ready. If the container is not ready then it's removed from the set of available containers that can provide the service, and any requests are routed to other containers that can provide the service. 

Unlike a liveness probe is a container fails it's not killed off, and calls to the readiness probe continue to be made, if the probe startes reporting the service in the container is ready then it's added back to the list of containers that can deliver the servcie and requests will be routed to it once more.

In the helidon-kuberneties folder edit the storefront-deployment.yaml file again

Look for the section (just after the Liveness probe) where we define the readiness probe. It should have text like this

```
#        readinessProbe:
#          exec:
#            command:
#            - /bin/bash
#            - -c
#            - 'curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""'
#          # No point in checking until it's been running for a while 
#          initialDelaySeconds: 15
#          # Allow a short delay for the response
#          timeoutSeconds: 5
#          # Check every 10 seconds
#          periodSeconds: 10
#          # Need at least only one fail for this to be a problem
#          failureThreshold: 1
```
Remove the # (and only the #, not spaces or anything else) and save the file. The ReadinessProbe section should look like this

```
       # This checks if the pod is ready to process requests
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - 'curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""'
          # No point in checking until it's been running for a while 
          initialDelaySeconds: 15
          # Allow a short delay for the response
          timeoutSeconds: 5
          # Check every 10 seconds
          periodSeconds: 10
          # Need at least only one fail for this to be a problem
          failureThreshold: 1
```

The various options for readiness are similar to those for readiness, except you see here we've got an exec instead of httpGet.

The exec means that we are going to run code **inside** the pod to determine if the pod is ready to serve requests. The command section defined the command that will be run ant the arguments. In this case we run the /bin/bash shell, -c means to extuse the arguments as a command (so it won't try and be interactive) and the 'curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""' is the command.

Some points about 'curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""'

Firstly this is a command string that actually runs 3 commands connecting the output of one to the input of the other. If you exec to the pod you can actually run these by hand if you like

```
$ kubectl exec -ti storefront-b44457b4d-29jr7  -- /bin/bash
root@storefront-b44457b4d-29jr7:/#
```

The first (the curl) gets the readiness data from the service (you may remember this in the Helidon labs) 

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready 
{"outcome":"UP","status":"UP","checks":[{"name":"storefront-ready","state":"UP","status":"UP","data":{"storename":"My Shop"}}]}
```

The resulting data however is a single line, so not easy to look at just the results of the outcome. We then use json_pp to take that single line of data and reformat it into something nice 

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready | json_pp
{
   "status" : "UP",
   "outcome" : "UP",
   "checks" : [
      {
         "state" : "UP",
         "status" : "UP",
         "data" : {
            "storename" : "My Shop"
         },
         "name" : "storefront-ready"
      }
   ]
}
```

Now each element is on a line of it's own, and we can just use the final command the grep to look for a line containing `"outcome" : "UP"` We do however have to be careful because " is actually part of what we want to look for, as are the space characters, so to define these as a constant we need to ensure it's a single string, we do this by enclosing the entire thing in quotes `"` and to prevent the `"` within the string being interpresed as end of string (and thuis new argument) characters we need to escape them, hence we end up with `"\"outcome\" : \"UP\""`

The final command becomes the following

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""
   "outcome" : "UP",
```

In this case the pod is ready, so the grep command returns what it's found. We are not actually concerned with what the pod returns in terms of string output, we are looking for the exit code, interactivly we can find that by looking in the $? variable

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""
   "outcome" : "UP",
root@storefront-b44457b4d-29jr7:/# echo $?
0
```

And can see that the variable value (which is what's returned back to the Kubernetes readiness infrastructure) is 0. In Unix / Linux terms this means success.

If you want to see what it would do if the outcome was not UP try running the command changing the UP to DOWN (or actually anything other than UP) **Important** While you can run this command in the pods shell •DO NOT• modify the yaml like this.

```
root@storefront-b44457b4d-29jr7:/# curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"DOWN\""
root@storefront-b44457b4d-29jr7:/# echo $?
1
```

In this case the return value in $? is 1, not 0, and in Unix / Linux terms that means something's gone wrong.

The whole thing is held in single quotes `'curl -s http://localhost:9080/health/ready | json_pp | grep "\"outcome\" : \"UP\""'` to that it's treated as a single string by the yaml file parser and when being handed to the bash shell.

That's all we're going to do with bash shell programming for now !

Having made the changes let's undeploy the existing configuration and then deploy the new one

In a terminal in the helidon-kubernties folder run the undeploy.sh script

```
$ ./undeploy.sh 
Deleting storefront deployment
deployment.extensions "storefront" deleted
Deleting stockmanager deployment
deployment.extensions "stockmanager" deleted
Deleting zipkin deployment
deployment.extensions "zipkin" deleted
Kubenetes config is
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-29lmk   1/1     Running   0          46m
pod/storefront-b44457b4d-29jr7      1/1     Running   1          46m
pod/zipkin-88c48d8b9-bftvx          1/1     Running   0          46m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h41m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h41m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h41m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       46m
replicaset.apps/storefront-b44457b4d      1         1         1       46m
replicaset.apps/zipkin-88c48d8b9          1         1         1       46m
```

As usual it takes a few seconds for the deployments to stop, this was about 30 seonds later

```
$ kubectl get all
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h42m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h42m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h42m
```

Now let's deploy them again, run the deploy.sh script, be prepared to run kubectl get all within a few seconds of the deploy finishing.

```
$ ./deploy.sh
Creating zipkin deployment
deployment.extensions/zipkin created
Creating stockmanager deployment
deployment.extensions/stockmanager created
Creating storefront deployment
deployment.extensions/storefront created
Kubenetes config is
NAME                                READY   STATUS              RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   0/1     ContainerCreating   0          0s
pod/storefront-74cd999d8-dzl2n      0/1     Pending             0          0s
pod/zipkin-88c48d8b9-vdn47          0/1     ContainerCreating   0          0s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h42m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h42m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h42m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   0/1     1            0           0s
deployment.apps/storefront     0/1     1            0           0s
deployment.apps/zipkin         0/1     1            0           0s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         0       0s
replicaset.apps/storefront-74cd999d8      1         1         0       0s
replicaset.apps/zipkin-88c48d8b9          1         1         0       0s
```

**Within a few seconds** run kubectl get all to see the status of the system

```
$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   1/1     Running   0          12s
pod/storefront-74cd999d8-dzl2n      0/1     Running   0          12s
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          12s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h43m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h43m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h43m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           12s
deployment.apps/storefront     0/1     1            0           12s
deployment.apps/zipkin         1/1     1            1           12s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       12s
replicaset.apps/storefront-74cd999d8      1         1         0       12s
replicaset.apps/zipkin-88c48d8b9          1         1         1       12s
```

We see something different, usually within a few seconds of starting a pod it's in the Running state (as are these) **BUT** if we look at the pods we see that though it's running the storefront is not actually ready, the replica set doesn't have any storefront ready (even though we've asked for 1) and neither does the deployment. 

If after a minute or so we re-do the kubectl command it's as we expected.

```
$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-vqq7c   1/1     Running   0          95s
pod/storefront-74cd999d8-dzl2n      1/1     Running   0          95s
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          95s

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   4h44m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   4h44m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            4h44m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           95s
deployment.apps/storefront     1/1     1            1           95s
deployment.apps/zipkin         1/1     1            1           95s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       95s
replicaset.apps/storefront-74cd999d8      1         1         1       95s
replicaset.apps/zipkin-88c48d8b9          1         1         1       95s
```

Now everything is ready, but why the delay ? What's caused it ? And why didn't it happen before ?

Well the answer is simple. If there is a readiness probe enabled a pod is not considered ready *until* the readiness probe reports success. Prior to that the pod is not in the set of pods that can deliver a service.

As to why it didn't happen before if a pod does not have a readiness probe specified then it is automatically assumed to be in a ready state as soon as it's running

What happens if a request is made to the service while before the pod is ready ? Well if there are other pods in the service (the selector for the service matches the labels and the pods are ready to respond) then the service requests are sent to those pods. If there are **no** pods ab.e to servcie the requests than a 503 "Service Temporarily Unavailable" is generated back to the caller

To see what happens if the readiness probe does not work we can simply undeploy the stock manager service.

First let's check it's running fine (remembering to substitute localhost for the IP address of the ingres load balancer.)

```
$ curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:30:32 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

Now let's use kubectl to undeploy just the stockmanager service. You need to run this command in the helidon-labs folder.

```
$ kubectl delete -f stockmanager-deployment.yaml
deployment.extensions "stockmanager" deleted
```
Let's check the pods status

```
macbook-pro:helidon-kubernetes tg13456$ kubectl get pods
NAME                            READY   STATUS        RESTARTS   AGE
stockmanager-6456cfd8b6-vqq7c   0/1     Terminating   0          26m
storefront-74cd999d8-dzl2n      1/1     Running       0          26m
zipkin-88c48d8b9-vdn47          1/1     Running       0          26m
```
The stock manager service is being stopped, after a short while if we run kubectl again to get everything we see it's gone

```
$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/storefront-74cd999d8-dzl2n   0/1     Running   0          28m
pod/zipkin-88c48d8b9-vdn47       1/1     Running   0          28m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h11m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h11m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h11m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/storefront   0/1     1            0           28m
deployment.apps/zipkin       1/1     1            1           28m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/storefront-74cd999d8   1         1         0       28m
replicaset.apps/zipkin-88c48d8b9       1         1         1       28m

```
Something else has also happened though, the storefront service has no pods in the ready state, neither does the storefront deployment and replica set. The readiness probe has run against the storefront pod and when the probe checked the results it found that the storefront pod was not in a position to operate, because the service it depended on (the stock manager) was no longer available. 

Let's try accessing the service

```
$ curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 503 Service Temporarily Unavailable
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:37:29 GMT
Content-Type: text/html
Content-Length: 203
Connection: keep-alive

<html>
<head><title>503 Service Temporarily Unavailable</title></head>
<body>
<center><h1>503 Service Temporarily Unavailable</h1></center>
<hr><center>openresty/1.15.8.2</center>
</body>
</html>
```
The service is giving us a 503 Service Temporarily Unavailable message. Well to be precise this is coming from the Kubernetes as it can't find a storefront service that is in the ready state.

Let's start the stockmager service using kubectl again

```
$ kubectl apply -f stockmanager-deployment.yaml
deployment.extensions/stockmanager created
```

Now let's see what's happening with our deployments 

```
macbook-pro:helidon-kubernetes tg13456$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-4mpl2   1/1     Running   0          7s
pod/storefront-74cd999d8-dzl2n      0/1     Running   0          33m
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          33m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h16m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h16m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h16m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           7s
deployment.apps/storefront     0/1     1            0           33m
deployment.apps/zipkin         1/1     1            1           33m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       7s
replicaset.apps/storefront-74cd999d8      1         1         0       33m
replicaset.apps/zipkin-88c48d8b9          1         1         1       33m
```

The stockmanager is running, but the storefront is still not ready, and it won't be until the readiness check is called again and determines that it's ready to work.

Looking at the kubectl output abut 90 seconds later

```
$ kubectl get all
NAME                                READY   STATUS    RESTARTS   AGE
pod/stockmanager-6456cfd8b6-4mpl2   1/1     Running   0          96s
pod/storefront-74cd999d8-dzl2n      1/1     Running   0          35m
pod/zipkin-88c48d8b9-vdn47          1/1     Running   0          35m

NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/stockmanager   ClusterIP   10.100.65.58    <none>        8081/TCP,9081/TCP   5h18m
service/storefront     ClusterIP   10.96.237.252   <none>        8080/TCP,9080/TCP   5h18m
service/zipkin         ClusterIP   10.104.81.126   <none>        9411/TCP            5h18m

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/stockmanager   1/1     1            1           96s
deployment.apps/storefront     1/1     1            1           35m
deployment.apps/zipkin         1/1     1            1           35m

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/stockmanager-6456cfd8b6   1         1         1       96s
replicaset.apps/storefront-74cd999d8      1         1         1       35m
replicaset.apps/zipkin-88c48d8b9          1         1         1       35m
```

The storefront readiness probe has kicked in and the services are all back in the ready state once again

```
$ curl -i -X GET -u jack:password http://localhost:80/store/stocklevel
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 02 Jan 2020 17:42:40 GMT
Content-Type: application/json
Content-Length: 184
Connection: keep-alive

[{"itemCount":4980,"itemName":"rivet"},{"itemCount":4,"itemName":"chair"},{"itemCount":981,"itemName":"door"},{"itemCount":25,"itemName":"window"},{"itemCount":20,"itemName":"handle"}]
```

### Startup probes
You may have noticed above that we had to wait for the readiness probe to complete on a pod before it became ready, and worse we had to wait the intialDelaySeconds before we could sensibly even start testing to see if the pod was ready. This means that is we wanted to do add extra pods then there is a delay before the new capacity come online and support the service, In the case of the storefront this is not to much of a problem as the service starts up fast, but for a more complex service, especially a legacy service that may have a startup time that varies a lot depending on other factors, this could be a problem, after all we want to respond to request as soon as we can.

To solve this in Kubernetes 1.16 the concept of startup probes was introduced. A startup probe is a very simple probe that tests to see if the service has started running, usually at a basic level, and then starts up the liveness and readiness probes. Effectively the startupProbe means there is no longer any need for the initialDelaySeconds.

Unfortunately however the startup probes are not supported in versions of Kubernetes prior to 1.16, and as our Kubernetes environment (and most other cloud providers) are not yet on that version we can't demo that. But there is example configuration in the storefront-deployment.yaml file to show how this would would be defined (the initialDeploymentSeconds would need to be removed from the liveness and readiness configurations.

Once we have a 1.16 or later production deployment this section of the lab will be updated to cover the startup probes in more detail.





---

You have reached the end of this lab !!

Use your **back** button to return to the **C. Deploying to Kubernetes** section