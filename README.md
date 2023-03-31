# K8S.SocialMedia

React front end on nginx, Node APIs and MongoDB

## Preparation

First need to get the following tools installed:

- Install Bash

- Install Kubernetes

- Install Minikube

- Install Docker

Run script `install.sh` to clone the necessary repositories and build the docker images.

## Application packaging and deployment

### Project Structure

The React frontend application is contained within the folder `frontend` and the backend Node project is
contained within `backend`. The kubernetes `yaml` files for both front end and the backend are inside
the folder `k8s`.

### MongoDB

We will use the official `mongo` image from dockerhub and run a mongodb `service` in our kubernetes cluster. This
`service` will be used by the `backend` as data storage.

The `mongo.yaml` defines a `ReplicationController` which ensures that there is always one instance of `mongo`
running in our cluster. This mongodb `pod` is listening on container port `27017` and the data storage path is `/data/db`
within the container which is mounted on host at `/data/storage/mongodb`.

The mongodb `pod` above is exposed as a service within our kubernetes cluster as a service with name `mongo-service` as
described in `mongo.yaml`.

#### Interacting with Mongo from within the POD

```bash
egarv:k8s estebangarviso$ kubectl get pods
egarv:k8s estebangarviso$ kubectl get pod mongo-controller-5b3dl
NAME                     READY     STATUS    RESTARTS   AGE
mongo-controller-5b3dl   1/1       Running   0          5m
egarv:k8s estebangarviso$ kubectl exec -it mongo-controller-5b3dl -- /bin/bash
```

### Node Service

### React FrontEnd

### nginx Reverse Proxy

## Complete Deployment

- `minikube start` to start the local kubernetes cluster which takes a while to get the cluster ready.

  ```bash
       egarv:k8s estebangarviso$ minikube start
       There is a newer version of minikube available (v0.19.0).  Download it here:
       https://github.com/kubernetes/minikube/releases/tag/v0.19.0

       To disable this notification, run the following:
       minikube config set WantUpdateNotification false
       Starting local Kubernetes cluster...
       Starting VM...
       SSH-ing files into VM...
       Setting up certs...
       Starting cluster components...
       Connecting to cluster...
       Setting up kubeconfig...
       Kubectl is now configured to use the cluster.
       egarv:k8s estebangarviso$
  ```

- Deploy and start up the `mongo` service using the descriptor file `mongo.yaml`

  ```bash
   egarv:k8s estebangarviso$ kubectl create -f mongo.yaml
   replicationcontroller "mongo-controller" created
   service "mongo-service" created
   egarv:k8s estebangarviso$
  ```

  Check that the pod and service are created and running successfully:

  ```bash
  egarv:k8s estebangarviso$ kubectl get pods
  NAME                     READY     STATUS    RESTARTS   AGE
  mongo-controller-srq9f   1/1       Running   0          8m


  egarv:k8s estebangarviso$ kubectl get services
  NAME            CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
  kubernetes      10.0.0.1     <none>        443/TCP     27d
  mongo-service   10.0.0.111   <none>        27017/TCP   8m
  egarv:k8s estebangarviso$

  ```

  Check the logs to make sure mongodb started successfully:

  `kubectl logs mongo-controller-srq9f` in my case. Run command with the correct `pod` name.

- Deploy and start up the Node service

  ```bash
  egarv:k8s estebangarviso$ kubectl create -f backend.yaml
  deployment "backend" created
  service "backend" created

  egarv:k8s estebangarviso$ kubectl get pods
  NAME                       READY     STATUS    RESTARTS   AGE
  backend-2415081020-tmx44   1/1       Running   0          51s
  mongo-controller-srq9f     1/1       Running   0          21m


  egarv:k8s estebangarviso$ kubectl get services
  NAME            CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
  backend         10.0.0.88    <none>        8080/TCP    1m
  kubernetes      10.0.0.1     <none>        443/TCP     27d
  mongo-service   10.0.0.111   <none>        27017/TCP   21m
  egarv:k8s estebangarviso$

  ```

- Deploy and start the Front End

  ```bash
  egarv:k8s estebangarviso$ kubectl create -f client.yaml
  deployment "frontend" created
  service "frontend" created
  egarv:k8s estebangarviso$

  egarv:k8s estebangarviso$ kubectl get pods
  NAME                        READY     STATUS    RESTARTS   AGE
  backend-2415081020-tmx44    1/1       Running   0          8m
  frontend-3251706051-p7770   1/1       Running   0          31s
  mongo-controller-srq9f      1/1       Running   0          29m
  egarv:k8s estebangarviso$

  egarv:k8s estebangarviso$ kubectl get services
  NAME            CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
  backend         10.0.0.88    <none>        8080/TCP         9m
  frontend        10.0.0.86    <nodes>       8888:32318/TCP   52s
  kubernetes      10.0.0.1     <none>        443/TCP          27d
  mongo-service   10.0.0.111   <none>        27017/TCP        29m
  egarv:k8s estebangarviso$

  ```

- Access the application from outside the cluster

  ```bash
  egarv:k8s estebangarviso$ minikube service frontend
  Opening kubernetes service default/frontend in default browser...
  egarv:k8s estebangarviso$

  ```

  To just list the url of the service:

  ```bash
  egarv:k8s estebangarviso$ minikube service frontend --url
  http://192.168.99.100:32318
  egarv:k8s estebangarviso$

  ```

### Using ConfigMap

We hardcoded value for the `DATABASE_URL` env variable to `mongo-service` in our deployment yaml for backend.

```bash
    spec:
      containers:
        - name: backend
          image: "estebangarviso/Api.SocialMedia"
          ports:
          - containerPort: 8080
          env:
            - name: DATABASE_URL
              value: mongo-service
```

Also there is `customer.message` defined in `application.properties` which we would like to populate via configmap
to fully externalise conig from code.

We want to change it so that the value to the environment variable is fed from config map so that configuration
is completely separate from the image.

Create a backend config map yaml `backend-config.yaml` with the relevant config data:

```bash
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: backend-config
      namespace: default
    data:
      DATABASE_URL: mongo-service
      customer_message: message overridden from configmap
```

Then execute `kubectl create -f backend-config/backend-config.yaml`

or from literal

`kubectl create configmap backend-config --from-literal=mongo.host=mongo-service --from-literal=customer.message='original message modified'`

```bash
    egarv:k8s estebangarviso$ kubectl create configmap backend-config --from-file=backend-config
    configmap "backend-config" created
    egarv:k8s estebangarviso$ kubectl describe configmap backend-config
    Name:		backend-config
    Namespace:	default
    Labels:		<none>
    Annotations:	<none>

    Data
    ====
    backend-config.properties:
    ----
    DATABASE_URL=mongo-service
    customer.message="Message overwritten from configmap"
    egarv:k8s estebangarviso$

```

The to create backend specifying the environment variable names,
`kubectl create -f backend-with-configmap-specify-names.yaml`

Or to create backend importing everything in the configmap as env variables,

`kubectl create -f backend-with-everything-in-configmap.yaml`

### Useful Docker and Kubernetes commands

### Create the front end

First install `React cli` gloablly with `npm install -g @React/cli`.
Now generate a new project with React cli by executing
`ng new client`. We have named our application `client`.

Create a `Dockerfile` using `nginx` image to deploy our React app.

`docker build -t estebangarviso/Web.SocialMedia`
`docker push estebangarviso/Api.SocialMedia`

To remove all stopped containers
`docker rm $(docker ps -a -q)`

### Running locally

- MongoDB - exec `mongod --dbpath /data/db`, where `/data/db` is your local drive for mongodb to write to.
  `chmod` write permission on it.
- Backend - `./gradlew bootRun`

- Front End - `ng serve --proxy-conf proxy.conf.json`

### Running locally using Docker

- MongoDB - `docker run -p 27017:27017 mongo`
- Backeend - prepare docker image `./gradlew buildDocker -x test` - and run using `docker run  -p 8080:8080 estebangarviso/Api.SocialMedia`
- FrontEnd - run `ng build` to produce dist artefact and `docker build -t estebangarviso/Web.SocialMedia `
  in the folder `client` where we have our `Dockerfile`. Then `docker run estebangarviso/Web.SocialMedia -p 4300:80`

But need Docker compose to make a bridge to make them talk to one another.

## Running Kubernetes in AWS

Easiest way is to do it using stackpoint.io

Initial set up:

- Sign up with AWS, and have a free tier ubuntu instance set up
- Locall install aws client for CLI
- Locally install KOPS

- Create Route53 domain for the cluster with name `dev.demo.com`
  `aws route53 create-hosted-zone --name dev.demo.com --caller-reference 1`
  And verify with `dig NS dev.demo.com`
- Create S3 bucket to store the cluster state
  `aws s3 mb s3://clusters.dev.demo.com`
- Export `export KOPS_STATE_STORE=s3://clusters.dev.demo.com`, store it as env variable in `.bash-profile`.

- Build the cluster
  `kops create cluster --zones=us-east-2a useast2.dev.demo.com`
