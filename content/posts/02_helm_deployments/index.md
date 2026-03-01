---
date: '2026-03-01T08:58:18+01:00'
title: 'Using Helm to manage Kubernetes deployments'
draft: false
summary: In this post I go over the steps of getting into a shift-left mindset. I will go over the four steps of software release (DTAP), and take a practical approach to learning this concept. 
tags: [quarkus, deployment, kubernetes, minikube, cloud-native, docker]
series: Creating an application with Quarkus
---

Getting to production in a reliable and quick way happens in stages. These stages are often referred as the DTAP stages, which stands for (D)evelop (T)est (A)cceptence and (P)roduction. Apart from being stages in the DevOps cycle they are also physical environments in which the application runs. In this post we will take an application and prepare it for deployment in each environment. 

## Prerequisites
In the [previous](../01_deploy_quarkus/index.md) post we've deployed an application to a local Minikube cluster. If you want to follow along you can checkout [this](https://github.com/woutervanveen/quarkus-validation/tree/deploying-quarkus-to-minikube) branch which is the start of this post. Make sure you have all the tools installed we need for this tutorial:

- [Docker](https://www.docker.com/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [Minikube](https://minikube.sigs.k8s.io/docs/start/?arch=%2Flinux%2Fx86-64%2Fstable%2Fbinary+download)
- [Helm](https://helm.sh/docs/intro/install)


## Goal
After reading this post you should have a clear grasp on how to separate the settings of your [Quarkus]() application for the four distinct stages of deployment. In addition, you will learn on how to manage Kubernetes deployements with Helm

## Environments
Like I said in the introduction we need four environments. Normally you would not host all four environments on your development computer, but for this tutorial we will simulate a real environment by creating each environment. 
### Development
This is actually the Minikube cluster we've created before. When you are developing your application locally you want to develop "as closely" to the production environment. That's why we've chosen to setup a local cluster. 

### Test
When you finish a new feature, fixing a bug, or just finishing your ticket you will need to merge your code into the main branch. Your colleagues or collaborators will check your code, but it would be nice if they can also see the application running. This is something that can be done in the test environment. This is an environment which is production like, with test data, that should only be used by developers. The idea is that this is a safe space to break things. This is also a place where you want to be able to run multiple versions of your application in parallel. However, for the sake of simplicity we will not do this in this post just yet. To create the test-environment locally we will use minikube:

```bash
minikube start -p test-environment
```
We are going to need access to our testing environment, so enable the ingress:
```bash
minikube -p test-environment addons enable ingress
```

### Acceptance
After your code has been merged it is time for your stakeholders, project managers, and maybe a group of your actual users to check the new features that you've build. This is an important step in the development process, because it wouldn't be the first time that a feature is misunderstood and implemented functionally wrong. Like we did before we are going to create the acceptance environment like so:

```bash
minikube start -p acceptance-environment
minikube -p acceptance-environment addons enable ingress
```

### Production
This is the real deal, this is the place where we don't test anymore but where everything should run reliable. Breaking production often costs money, reputation and can cause serious business disruptions. Therefore from an early stage on you should be in the "production" mindset. The sooner you catch bugs the better production will be. This mentality is often called "shift left", meaning that you catch bugs earlier in the development cycle. 

However, don't be scared of deploying to production. Deploy often to production, keep the changes small, and practice. Deploying multiple times a day should become normal and second nature. Your code should be so good and reliable that the point from going from acceptance to production is merely a formality. 

Create your production environment:
```bash
minikube start -p production-environment
minikube -p production-environment addons enable ingress

```


### Cleaning up your environments
Running 4 Kubernetes clusters might be demanding on your hardware, stopping a cluster is as simple as starting it:
```bash
minikube stop -p production-environment
```
If you want to see all the clusters that you've configured on your system run:
```bash
minikube profile list
```

Finally, if you want to delete a cluster you can run:
```bash
minikube delete -p production-environment
```

## Preparing your application
In the [previous](../01_deploy_quarkus/) post we've developed a simple application that displays the pod our application is currently running on. For this tutorial we want to expand this to also show in which environment our application is running. Open the `PodInformationResource.java` and create a new class field called `welcomeMessage`. We are going to read this welcome message from the configuration file that ships with Quarkus (`applciation.properties`). The whole file should look as:

```java
package org.cinema.debug.api;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@Path("/pod-info")
public class PodInformationResource {

  @ConfigProperty(name = "POD_NAME", defaultValue = "unknown")
  String podName;

  @ConfigProperty(name = "org.cinema.debug.welcome-message", defaultValue = "Not welcome")
  String welcomMessage;

  @GET
  @Produces(MediaType.TEXT_PLAIN)
  public String getPodInfo() {
    return "The welcome message is: " + welcomMessage + " and I am running on pod: " + podName;
  }
}
```
Now we need to add the welcome message to our `application.properties`, but we want the message to be different for each environment. Therefore, we are going to create three new files, one for each of our environments. We are going to create a `.properties` file for each of our environment. They have very specific naming for Quarkus to pick them up:
- `application-test.properties`
- `application-acceptance.properties`
- `application-production.properties`
In each file create the value of our welcome message, for instance for the acceptance environment:
```
org.cinema.debug.welcome-message="Hello and welcome to the acceptance environment!"
```
To activate a different environment in Quarkus we can simply set the environment variable `QUARKUS_PROFILE=<environment_name>`. 

### Switching to Helm
Helm will help us to manage our deployments in Kubernetes. At the moment we are deploying our applications with single `kubectl` apply commands. Switching to Helm at this point is quite simple. In your `deployments/` folder create a new folder called `templates/` and move all your current `.yaml` files into this folder. Now create a `Chart.yaml` with the following content:
```yaml
apiVersion: v2
name: Reservation cinema
version: 0.01
```
Now create a file called `values.yaml`, in this file we are going to manage the values of all the variables we can use in our Helm charts. For now I've defined the following three variables:
```yaml
# version of image to pull
imageVersion: v0.0.2
# number of replicas to run
replicas: 1
# port on which the application can be reached
port: 8080
```
It is common in Helm charts to write a comment on each line followed by the value. I don't like to use nested values, because it can become quite messy, and the makers of Helm actually advice against it. We can use the values inside our `.yaml` files which we moved into our `templates/` folder like so:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rescin-ticket-reservation-deployment
  namespace: rescin
spec:
  template:
    metadata:
      name: rescin-ticket-reservation
      labels:
        app: rescin-ticket-reservation
    spec:
      containers:
        - name: rescin-ticket-reservation
          image: rescin/ticket-reservation:{{ .Values.imageVersion }} 
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: {{ .Values.port }}
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
  replicas: {{ .Values.replicas }}
  selector:
     matchLabels:
       app: rescin-ticket-reservation

```
For instance the `replicas` field can be accessed with the `{{ .Values.replicas }}` shortcut. Let's also adapt the ingress controller:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rescin-ticket-reservation-ingress
  namespace: rescin
spec:
  rules:
    - http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: rescin-ticket-reservation-service
                port:
                  number: {{ .Values.port }}
```
And finally the `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rescin-ticket-reservation-service
  namespace: rescin
spec:
  selector:
    app: rescin-ticket-reservation
  ports:
    - port: {{ .Values.port }}
```
At the moment we can already deploy our service to our test environment. So let's do exactly that. First make sure that you build your docker containers into the test-environment:
```bash
mvn package
eval $(minikube -p test-environment docker-env)
docker build -f src/main/docker/Dockerfile.jvm -t rescin/ticket-reservation:v0.0.2 .
```
Now we can simply run the command:
```bash
helm install -f values.yaml rescin .
```
This will take all of our template files and apply the `values.yaml` file and deploy it into our cluster. Run the following command to see if your application is running and inspect the response:
```bash
curl -X GET <minikube cluster ip>/pod-info
```
If you don't know the ip-address if the cluster you can find it like so: `minikube -p test-environment ip`. The above command should give a response similar to:
```text
The welcome message is: Not welcome and I am running on pod: rescin-ticket-reservation-deployment-6888fbd7cb-4n85x%
```

#### Quarkus profiles
At this moment you should have a pod running in your test-environment. However, we haven't connected the internal data of our application to the correct environment. Actually, when we check the logs with `kubectl logs <pod-name> -n rescin` we see that the application profile `prod` is loaded:
```text
__  ____  __  _____   ___  __ ____  ______ 
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/ 
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \   
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/   
2026-03-01 06:54:59,668 INFO  [io.quarkus] (main) validation-with-quarkus 1.0.0-SNAPSHOT on JVM (powered by Quarkus 3.30.3) started in 0.973s. Listening on: http://0.0.0.0:8080
2026-03-01 06:54:59,671 INFO  [io.quarkus] (main) Profile prod activated. 
2026-03-01 06:54:59,671 INFO  [io.quarkus] (main) Installed features: [cdi, hibernate-validator, rest, rest-jackson, smallrye-context-propagation, smallrye-openapi, vertx]

```
In the beginning of this tutorial we've defined three Quarkus profiles:
- application-test.properties
- application-acceptance.properties
- application-production.properties
We need to enable the correct profile for the correct environment. We will start of by improving the deployment running to our test-environment. First create a file called `values-test.yaml`:
```yaml
# the profile used to start our quarkus application
quarkusProfile: test
```
Add the following line to your `deployment.yaml` under the `env` section:
```yaml
            - name: QUARKUS_PROFILE
              value: {{ .Values.quarkusProfile }}
```
Now, we try to get into a shift left mindset, so we are not just going to try this, we are first going to check if this will actually work. We can run the `helm upgrade` command with the `--debug` and the `--dry-run` flags and see if our templates are actually rendered correctly:
```bash
helm upgrade -f values.yaml -f values-test.yaml --debug --dry-run rescin .
```
If everything went well you should see the templates being rendered correctly and you should also see the Quarkus profile environment variable being set to `test`. Now run the above command without the `--debug` and the `--dry-run` commands. If you now run `helm list` you should see an entry for the deployment rescin with revision 2. Now run the curl command again and if all went well you should get the correct welcome message:
```text
The welcome message is: "Hello and welcome to the test environment!" and I am running on pod: rescin-ticket-reservation-deployment-66fd9675d5-4ljkb%          
```
Nice, we have a working test-environment! Let's extend this to the `acceptance` and `production` environments. First make sure to start both environments and build the images for both the environments. The next step is to create two more files: `values-acceptance.yaml` and `values-production.yaml`:
```yaml
# the profile used to start our quarkus application
quarkusProfile: acceptance
```

```yaml
# the profile used to start our quarkus application
quarkusProfile: production
```

To deploy to the acceptance environment we need to run the following steps:
```bash
minikube -pl acceptance-environment start
minikube -p acceptance-environment addons enable ingress

# create the namespace
cd cluster/
helm install -f values.yaml cluster . 
cd ..

# build the docker image
eval $(minikube -p acceptance-environment docker-env)
docker build -f src/main/docker/Dockerfile.jvm -t rescin/ticket-reservation:v0.0.2 .

# start the helm installation
cd deployment/
helm install -f values.yaml -f values-acceptance.yaml rescin .
```
Now again we can check the ip address with `minikube -pl acceptance-environment ip` and to check if all went well run the `curl` for this ip address. If all went well you should see the following message:
```text
The welcome message is: "Hello and welcome to the acceptance environment!" and I am running on pod: rescin-ticket-reservation-deployment-7c44998b96-rxhnq
```
The last deployment, maybe the most important one I leave to you. You can repeat the steps I showed above and you should be able to get this environment running. 

## Conclusion
I hope that at the end of this post you've developed a feel for the shift-left approach. I often see beautiful words about concepts like shift left, but not a single concrete example on what to do. That is off-course getting into writing production code as quickly as possible. In addition to shift-left I went over four stages of software release; development, test, acceptance and production (DTAP). I hope you've enjoyed this post so far and the code is available [here](https://github.com/woutervanveen/quarkus-validation/tree/deploying-with-helm).

