---
layout: post
title: Troubleshooting a Java Application In Kubernetes
excerpt: |
   Sometimes your Java app stops serving requests, becoming unhealthy, and then being restarted by Kubernetes. Let's troubleshoot!
img_url: /images/2021-03-27-java-troubelshooting-k8s.png
img_credits: Picture by <a href="https://unsplash.com/@nikkotations">nikko macaspac on Unsplash</a>
---


When Developers Meet Ops
========================

You've built your Java application and deployed it on Kubernetes in your dev environment. Eventually, this app will go to production and you will experience real usage. And then new unexpected problems will come.
Those problems can be due to a lot of reasons: too many users, memory leaks, race conditions that are hard to identify in the development phase

Of course, the first thing that you want to do is to troubleshoot and run a Root Cause Analysis to find the culprit. On your laptop, it's easy: when the application is blocked, you can run thread dumps, heap maps and try to understand what is blocking.

On Kubernetes, it's a bit different. As you've been told to add a health-check endpoint, k8s decided to restart your pod that is now fresh and has lost all the information that you wanted. In this post, I'll show a very simple way to capture information in such a situation.


Lifecycle Hooks In Kubernetes
============================

The main feature of Kubernetes that we will use here is called [Container Lifecycle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/). It allows to run some commands at two moments in the lifecycle of the pod:

 * `PostStart` hook: just after it starts
 * `PreStop` hook: just before it stops

The thing that is important here is of course the `PreStop` hook. It is triggered whenever Kubernetes sends a stop command to the container. It can happen if the pod gets evicted or if the health checks stop answerings or simply if you manually or automatically stop the container (new deployments for instance)

To configure a hook, it's as simple as adding some configuration to our resource:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - image: myapp:latest
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - /scripts/pre-stop.sh

    ...

  terminationGracePeriodSeconds: 30

```

In this sample, Kubernetes will run our `pre-stop.sh` script just before stopping our container. Another important parameter here is `terminationGracePeriodSeconds` which is the amount of time that k8s will wait for the pod to stop: the regular process that was launched but also the stop hook. That's why the `preStop` script should be rather fast to execute.

A Testing Application
=====================

To demonstrate the full process, I've developed a sample Quarkus based Java app that will allow us to test some failure scenarios. The idea is to expose a REST API that allows setting the app in an unhealthy mode. As sometimes, code explains things better than sentences, here is how it works:

```java
        given()
          .when().get("/q/health/live")
          .then()
             .statusCode(200);

        given()
        .when().put("/shoot")
        .then()
           .statusCode(200)
           .body(is("Application should now be irresponsive"));


        given()
        .when().get("/q/health/live")
        .then()
           .statusCode(503);

```

This application has been pushed on DockerHub as `dmetzler/java-k8s-playground`.


Capturing The Troubleshoot Informations
=======================================

To capture some useful information, we can try to run the following script. Of course, one can add other commands to get more insights. This one will give you :

 * The list of open connections
 * The status of the java process
 * The result of the liveness probe check
 * The JVM memory usage
 * A thread dump

We could store this script directly in the image, but it would not help a lot when one wants to do some small changes. Instead, we will store it in a configuration map and mount the `ConfigMap` as a volume.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: pre-stop-scripts
data:
  pre-stop.sh: |
    #!/bin/bash
    set +e

    NOW=`date +"%Y-%m-%d_%H-%M-%S"`
    LOGFILE=/troubleshoot/${HOSTNAME}_${NOW}.txt

    {
      echo == Open Connections =======
      netstat -an
      echo

      echo == Process Status info ===================
      cat /proc/1/status
      echo

      echo == Health endpoint =========================
      curl -m 3 localhost:8080/q/health/live
      if [ $? -gt 0 ];  then
        echo "Health endpoint resulted in ERROR"
      fi
      echo

      echo == JVM Memory usage ======================
      jcmd 1 VM.native_memory
      echo

      echo == Thread dump ===========================
      jcmd 1 Thread.print

    } >> $LOGFILE 2>&1
```

This script expects to have writeable access to a `/troubleshoot` directory to store the result. We will need to provide an additional volume for that purpose.

Deploying The Application With The PreStop Hook
===============================================

Here is the complete deployment resource that we will create:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: java-k8s-playground
  name: java-k8s-playground
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-k8s-playground
  template:
    metadata:
      labels:
        app: java-k8s-playground
    spec:
      containers:
        - name: java-k8s-playground
          image: dmetzler/java-k8s-playground

          # Health probes (1)
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /q/health/live
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 10
          readinessProbe:
            failureThreshold: 15
            httpGet:
              path: /q/health/ready
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 5
            successThreshold: 1
            timeoutSeconds: 3

          # We ask to run the troubleshoot script when stopping (2)
          lifecycle:
            preStop:
              exec:
                command:
                  - /bin/sh
                  - /scripts/pre-stop.sh


          # Volumes to get the script and store the result (3)
          volumeMounts:
            # The config map that contains the script
            - mountPath: /scripts
              name: scripts
            # Where to store the information
            - mountPath: /troubleshoot
              name: troubleshoot
      volumes:
        - emptyDir: {}
          name: troubleshoot
        - configMap:
            defaultMode: 493
            name: pre-stop-scripts
          name: scripts

      terminationGracePeriodSeconds: 30

```

 1. We first configure the health checks so that when the application becomes unavailable, the pod is automatically restarted by Kubernetes.

 2. This is where we tell Kubernetes to run our script when the pod stops.

 3. We provide two volumes. One to mount the troubleshoot script, and another to store the result. Here we use an `emptyDir` directory. This may be a problem if the pod is rescheduled on another node (for instance when it's evicted), and we may prefer using a more persistent type of volume. In our experience, the container will just be restarted, so we do not have that problem.


Shoot To Kill
=============

Now it's time to run our experiment. We start by deploying our two resources:

```shell
$ kubectl create -f configmap.yaml
configmap "pre-stop-scripts" created
$ kubectl create -f deployment.yaml
deployment "java-k8s-playground" created
$ kubectl get deployment java-k8s-playground
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
java-k8s-playground   1         1         1            1           5s
```

Our app is up and running and we just need to `cURL` the `/shoot` endpoint to switch in an unhealthy state.
```shell
$ oc get pod -l app=java-k8s-playground
NAME                                   READY     STATUS    RESTARTS   AGE
java-k8s-playground-65d6b9b4f4-xrjgn   1/1       Running   1          23m
$ kubectl exec -it java-k8s-playground-65d6b9b4f4-xrjgn -- bash
bash-4.4$ curl -XPUT localhost:8080/shoot
Application should now be irresponsive
bash-4.4$  command terminated with exit code 137
$ # Here we eventually lose the connection as the pod has been restarted by k8s
```

The container has been restarted by Kubernetes. Let's see what we have in the newly created container.

```shell
$ kubectl exec -it java-k8s-playground-65d6b9b4f4-xrjgn -- bash
bash-4.4$ cd /troubleshoot/
bash-4.4$ ls
java-k8s-playground-65d6b9b4f4-xrjgn_2021-03-27_18-06-48.txt
bash-4.4$
bash-4.4$ head -n 5 java-k8s-playground-65d6b9b4f4-xrjgn_2021-03-27_18-06-48.txt
== Open Connections =======
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp6       0      0 :::8080                 :::*                    LISTEN
tcp6       0      0 172.16.247.87:8080      172.16.246.1:42238      TIME_WAIT
```

That's it, we now all the information we want in the file. The complete output of our script can be found [here](https://github.com/dmetzler/dmetzler.github.io/blob/master/raw/2021-03-27-troubleshoot-result.txt)


Using Kubernetes Termination Message
====================================

Adding a volume to hold the result of our investigation can be seen as a bit complex in production deployment scenarios. Kubernetes offers a lighter, but also limited, solution to store some termination information that is called the termination message.

In a pod definition, you can specify the `spec.terminationMessagePath` (defaults to `/dev/termination-log`) in which you can write some information on why the pod stopped. The [documentation](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/) states that :

> The log output is limited to 2048 bytes or 80 lines, whichever is smaller.

In our case, the output of the script is usually bigger than that (around 30Kb), so we cannot redirect all the content to that file. Instead, a good idea may be to just store the output of the health probe. So we can modify our script like this:

```shell
...
echo == Health endpoint =========================
curl -m 3 localhost:8080/q/health/live > /dev/termination-log
if [ $? -gt 0 ];  then
  echo "Health endpoint resulted in ERROR"
fi
cat /dev/termination-log
echo
...
```

After running the same detroying experiment, let's see what we have in the pod's description:

```shell
$ oc describe pod java-k8s-playground-65d6b9b4f4-xrjgn
Name:           java-k8s-playground-65d6b9b4f4-xrjgn
Namespace:      XXXXX
Node:           ip-XXXXXX.ec2.internal/XXXXXX
Start Time:     Sat, 27 Mar 2021 18:41:27 +0100

....

Containers:
  java-k8s-playground:

    ...

    State:         Running
      Started:     Sat, 27 Mar 2021 19:06:49 +0100
    Last State:    Terminated
      Reason:      Error
      Message:
{
    "status": "DOWN",
    "checks": [
        {
            "name": "App is down",
            "status": "DOWN"
        }
    ]
}
      Exit Code:    143
      Started:      Sat, 27 Mar 2021 18:42:29 +0100
      Finished:     Sat, 27 Mar 2021 19:06:49 +0100
    Ready:          True
    Restart Count:  2
$
```

Kubernetes was able to get the result of the probe and store it in the ***Last State*** section (also available in `status.containerStatuses[0].lastState.terminated.message` in the YAML representation). Of course, the amount of data we can add in there is limited as it probably ends up in `etcd`.

That can offer a quick way to get some useful statuses though, without having to provision and plan for a troubleshooting script. For instance, in Nuxeo, we can store the result of the `runningstatus` probe. Another option is to set `terminationMessagePolicy` to `FallbackToLogsOnError`. With this, if the message is empty, then Kubernetes takes the last line of the console logs as the message. So, a quick way to get a troubleshoot status when running Nuxeo in Kubernetes would be to add this in the pod definition:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
    - image: nuxeo:LTS
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/bash
              - -c
              - "/usr/bin/curl -m 3 localhost:8080/nuxeo/runningstatus > /dev/termination-log"
      terminationMessagePolicy: FallbackToLogsOnError

```

It will then save the result of the running status probe in the last state of our container. A very common mistake in the configuration of the S3 credentials, for instance, can be quickly spotted by just looking at the last state message.

Conclusion
==========

When dealing with applications in production, the first reaction when an application is failing is to have a look at the logs. If this is of course useful, it can sometimes be quite difficult to dig in thousands of lines of logs, especially in Java where stack traces can be super verbose. Thread dumps or heap map dumps are usually super useful when the solution is not obvious just reading the logs.

Kubernetes lifecycle hooks offer a good solution to capture the state of your application at the time it is restarted. Planning for such an event is a really good practice and by default, k8s offers 30 seconds to gather everything you need.



References
==========

 * [Container Lifecyle Hooks](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)
 * [Determine reason for pod failures](https://kubernetes.io/docs/tasks/debug-application-cluster/determine-reason-pod-failure/)
 * [The test application source on Github](https://github.com/dmetzler/java-k8s-playground)

