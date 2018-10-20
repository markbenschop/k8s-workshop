# k8s demo op PKS

# Preparation
## Network
Create a hostfile entry 

    1.2.3.3. traefik.demo.intern ${je naam}.demo.intern

Configure your browser so that '.demo.intern' is accessed not via a proxyserver.

## Work directory
Create a working directory called workshop and cd into it.

    mkdir workdir
    cd workdir

Clone de k8s-workshop git repository

    git clone http://dhcp-scope-karc-001.int-prod.rws.nl:8080/benschopm/k8s-workshop. 

The workshop excercises we are going to do are in DEMO.md


Create a directory where you manage the files you create and make it a git repo so you can save working versions.

    mkdir files
    cd files
    git init
    cd ..


# Setup kubectl
To connect to k8s you need access to the api. 

## Install kubectl
You need to install kubectl, a go binary.

https://kubernetes.io/docs/tasks/tools/install-kubectl/

## Create configuration file 
To get access to the demo api you need some settings.

These are in the file named 'config' in this repo. 

Put this file in ~/.kube/config and have a look at the file to see what's in it.

## Check access
Now check if you have access to the k8s api.

    kubectl cluster-info

List the running pods in all namespaces

    kubectl get pods --all-namespaces

You will be typing 'kubectl' many times. Make a short little alias maybe ?

    alias k=kubectl


# Create your own namespace
We are going to work with multiple people on a single k8s cluster. To stay out of each others way we are going to work in our own namespaces.

List current namespaces with :

    kubectl get namespaces

or the short version 

    kubectl get ns


Create a namespace which has your own name. Do this by creating namespace.yml.

    ---
    apiVersion: v1
    kind: Namespace
    metadata:
        name: <your-name-here>

Apply the namespace :

    kubectl apply -f namespace.yaml

Liste namespaces  with

    kubectl get ns

From now on you will work in your own namespace. To prevent from having to add '-n <your-name-here>' to every kubectl command :
    kubectl config set-context demo --namespace=<your_name_here>

Check if your context is set with

    kubectl config get-context demo 

# Deploy demo app
We are going to work with an app that's called flask-demo.
Clone this git repository

    git clone https://github.com/markbenschop/flask-demo.git

In the flask-demo/k8s directory there are example files you can copy and use as an example.
The files are also in this document.

## Create configmap
Create configmap.yaml from flask-demo/k8s/10-configmap.yaml

A configmap is just that a configuration map. We can put all kinds of thing inside a map.
It can contain environment variables for the container or a complete configuration file that is mapped into a container.

    ---
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: flask-demo-environment
    data:
      NAME: MARK


## Create service 
Create service.yaml from example flask-demo/k8s/20-service.yaml

A service is needed for pods to communicate with each other. So pods never directly talk with each other. Traffic goes via the services.

Note that a service and pods are linked by using labels and selectors.

A deployment has a label. The service finds pods by using a selector that references the label of the pod.

    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: flask-demo-service
    spec:
      selector:
        app: flask-demo
      ports:
        - name: flask
          protocol: TCP
          port: 80
          targetPort: 5000


## Create deployment
Create deployement.yaml from flask-demo/k8s/30-deployment.yaml.

A deployment actually consists of several k8s components that can also be created seperately.

The deployment component is created to bundle some often combined items together in a logical way.

The parts are replication controller, selector, labels and containers.

You can forget about the seperate parts really and treat the deployment as one object.

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: flask-demo-deployment
      labels:
        app: flask-demo-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: flask-demo
      template:
        metadata:
          labels:
            app: flask-demo
        spec:
          containers:
          - name: flask-demo-container
            image: markbenschop/flask-demo:0.16
            ports:
            - containerPort: 5000
            envFrom:
            - configMapRef:
                name: flask-demo-environment
          restartPolicy: Always


## Everything ok ?
Is the deployment running ?

    kubectl get pods


# Expose flask-demo 
So now we have the flask-demo pods running. But we can not access them yet. And what't the point of having a nice app that we can't reach ?

## Create a nodePort service
The simplest way to expose pods is by creating a service with a nodePort. Look at flask-demo/k8s/nodeport-service.yml for an example.

To expose pods we can create a service that uses a nodePort. This will open up the specified portnumber on every node and route traffic to the service, which sends it to the pods with the labels that are specified in the selector.

Note that a port number can only be used once on a node and everyone who is doing the workshop is creating a service.
So alter the portnumber into some unique number (between 30000 and 32000 !) when you get an error like this :

    The Service "flask-demo-external-service" is invalid: spec.ports[0].nodePort: Invalid value: 30090: provided port is already allocated

    ---
    kind: Service
    apiVersion: v1
    metadata:
      name: flask-demo-external-service
    spec:
      selector:
        app: flask-demo
      ports:
        - protocol: TCP
          port: 5000
          nodePort: 30090
      type: NodePort


Now point your browser to http://${je naam}.demo.intern:30090


## Use a proper ingress controller
Clearly opening a different port for every service we want to access is not a scalable solution.

A better way is to use an ingress controller and ingress rules.

The ingress controller itself is exposed via a nodePort configuration. Normally a loadbalancer is put in front of the ingress controller.

The ingress controller we are using is traefik.

Open the url to traefik in your browser.
[http://traefik.demo.intern:30080/dashboard/]



## Create ingress rules
We are going to create an ingress rule for url http://${je_naam}.demo.intern:30080 to your flask-demo deployment.

One of the advantages is that with an ingress controller we can expose services with virtual host like functionality.


    ---
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
      name: my-flask-ingress
      annotations:
        kubernetes.io/insgress.class: traefik
    spec:
      rules:
        - host: mark.demo.intern
          http:
            paths:
              - path: /
                backend:
                  serviceName: flask-demo-service
                  servicePort: flask

Now the app should be accessible at [http://je_naam.demo.intern:30080].

Note that we now expose only one service on the / path. We could expose more services under a different path.

For experimentation create a different deployment and expose it on another path.

You can copy the configmap, service and deployment we created earlier, give it different names, labels and selectors and change the NAME variable in the configmap and expose it under e.g. /other_service in this ingress rule.


# Further experimentation with a deployment
## Scaling out
Scaling a deployment in or out is quite easy.

List your deployments

    kubect get deploy

Now let's scale out to 10 pods

    kubectl scale --replicas=10 deployment flask-demo-deployment 

Now you scale back to 5 replicas

## Upgrade pod version
We can update a deployment in a few ways. Let's say we want to upgrade the image we use from 0.16 to 0.17.

First way is via cli : 

    kubectl set image deployment/flask-demo-deployment flask-demo-container=markbenschop/flask-demo:0.17

Second way is with edit which will put you in your default editor :

    kubectl edit deployment/flask-demo-deployment

## Set livenessProbe
K8s wants the pods in your deployment to run. So it checks that and if a pod crashes it starts another one.

Now wouldn't it be nice if kubernetes could check if your service is running like it should ?

A running pod can run an applications that fails to answer requests right ?

Please meet the livenessProbe :D

K8s can 

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: flask-demo-deployment
      labels:
        app: flask-demo-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: flask-demo
      template:
        metadata:
          labels:
            app: flask-demo
        spec:
          containers:
          - name: flask-demo-container
            image: markbenschop/flask-demo:0.16
            ports:
            - containerPort: 5000
            envFrom:
            - configMapRef:
                name: flask-demo-environment
            livenessProbe:
              httpGet:
                path: /
                port: 5000
              initialDelaySeconds: 10
              timeoutSeconds: 5
    restartPolicy: Always


## Set resourceLimits
Isn't there a way to influence the resources a pod can use you ask yourself.

Luckilly there is !  We can use resourceLimits. 

ResourceLimits is actually an enourmous subject in k8s. It can be set on many items like cpu, memory, storage and namespaces. Resource classes can be created to put pods in quota classes. 

Today we only look at resourcelimits that set the initial and maximum memory and cpu a container can use.

So 'requests' limits set the initial amount and 'limits' the maximum.

Setting these will let k8s make better decisions as to where (on what node) to schedule a pods.

A memory heavy pod will only be schedules on a node that has enough memory available. 

Note that for cpu's 1000m is 1 cpu and for memory Mi, Gi. 


Have a look here :

    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: flask-demo-deployment
      labels:
        app: flask-demo-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: flask-demo
      template:
        metadata:
          labels:
            app: flask-demo
        spec:
          containers:
          - name: flask-demo-container
            image: markbenschop/flask-demo:0.16
            ports:
            - containerPort: 5000
            envFrom:
            - configMapRef:
                name: flask-demo-environment
            resources:
              # mem 15-20Mi           # Mi = Mebibytes
              # cpu 100-250 millicore # 1 core = 1000m
              requests:
                # Initial request
                memory: "15Mi"
                cpu: "100m"
                # Hard limit
              limits:  
                memory: "20Mi"
                cpu: "250m"
            livenessProbe:
              httpGet:
                path: /
                port: 5000
              initialDelaySeconds: 10
              timeoutSeconds: 5
    restartPolicy: Always

# GitOps with flux
We have been creating yaml files and applying them by hand with kubectl so far. In a production environment these files would be in a network accessable git repository so your team can work together on the configuration of your yaml files. Git is ideal for this since every change is recorded and revertable.

There is actually tooling you can install inside k8s that can poll the git repository and apply new configuration.  Meet weave/flux !

Flux is installed inside k8s. 

    kubectl get deploy --all-namespaces

Flux will monitor the git repo at {fill in url to repo }. Whenever yaml files are added flux will apply them. 

We will demonstrate by merging a branch that contains ivsnext with master.


