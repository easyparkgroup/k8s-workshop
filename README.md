# Kubernetes workshop

## Step 0: Preparations

Let's get ready for the workshop, so everyone would be prepared.

### Install required software
Please install these:
* docker (https://docs.docker.com/install/)
* Windows users:
    * install WSL https://docs.microsoft.com/en-us/windows/wsl/install-win10
    * install some linux distro via WLS, for example Ubuntu (and use its command-line)
    * if `docker ps` fails, open "Docker Desktop" GUI in Windows -> Settings -> Resources -> WSL INTEGRATION -> "Enable integration with my default WSL distro"
    * Use WSL console instead of Windows cmd or PowerShell for this workshop, including installing other software
      (docker is already installed in Ubuntu).
* gcloud SDK (https://cloud.google.com/sdk/install)
  * Windows users:
    * Install gcloud SDK via WSL
    * If installing Google Cloud SDK fails according to
      [official linux instructions](https://cloud.google.com/sdk/install) 
      via apt-get or yum because docker-credential-gcr is not supported, then
      * download the tar.gz file 
      `wget "https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-332.0.0-linux-x86_64.tar.gz"`.
      > Make sure to use the recent version of the SDK.
      * then unpack 
      `tar -zxf google-cloud-sdk-332.0.0-linux-x86_64.tar.gz`
      * run the shell installation process with
      `./google-cloud-sdk/install.sh`
* kubectl (https://kubernetes.io/docs/tasks/tools/install-kubectl/) - if you already have one, then check that it is at least version 1.20

### Connect to workshop k8s cluster and create your personal k8s namespace
### NB!! Always replace `[[variable here]]` with a valid variable 

We will be making a k8s namespace with your name e.g
```shell
k8sNamespace=[[aksel-allas]]
```

Open the terminal and run following lines one by one:
```shell
# after following command browser will be opened, where you should log into google cloud with Concise email to authenticate `gcloud` CLI
gcloud auth login

# updates a kubeconfig file (~/.kube/config) with appropriate credentials and endpoint information to point kubectl at a specific cluster in Google Kubernetes Engine.
gcloud container clusters get-credentials k8s-ws-12 --zone europe-west1-b --project k8s-ws-12

# register gcloud as a Docker credential helper (~/.docker/config.json)
gcloud components install docker-credential-gcr
docker-credential-gcr configure-docker

# check that `kubectl` is properly installed (at least version 1.16)
kubectl version

# get k8s nodes in the cluser to check that `kubectl` can communicate with the cluster
kubectl get nodes

# create your private namespace inside k8s-ws-8 cluster (isolates your stuff from other participants)
kubectl create namespace [[aksel-allas]]

# Configure your namespace as default
kubectl config set-context $(kubectl config current-context) --namespace=[[aksel-allas]]
```

### Verify configuration
Make sure that docker doesn't need sudo when running:
```sh
docker run hello-world
```
If it doesn't work without sudo, follow 
[Docker Post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/)
to allow non-privileged users to run Docker commands.

## Step 1: Create java application

Lets generate an application that has health endpoint (needed for k8s).
> No worries, you don't need to have Java, Gradle etc installed locally - that will be built in docker!

1. Go to this webpage: https://start.spring.io
2. Choose these options
    1. Project: Gradle Project
    2. Language: Java
    3. Spring Boot: 2.5.6
    4. Project metadata:
        1. defaults
        2. Java: 11
3. Dependencies -> Add dependencies:
     1. Spring Web
     2. Spring Boot Actuator
4. Generate (download)
5. Unzip to a folder in your computer


## Step 2: Dockerize the java application

Let's create a docker image, so that k8s wouldn't care what language or tech stack our application uses.

1. Copy [Dockerfile](Dockerfile) to the root folder of the java application (So dockerfile and unzipped java app is in the same folder)
2. Build it ```docker build --tag [[demo-app_aksel-allas]]:latest .```
3. Run it locally in the foreground: ```docker run --name [[demo-app_aksel-allas]] --rm -p 8080:8080 [[demo-app_aksel-allas]]:latest```
4. Open browser and check the health endpoint responds at http://localhost:8080/actuator/health
5. Tag the docker image ```docker tag [[demo-app_aksel-allas]]:latest eu.gcr.io/k8s-ws-12/[[demo-app_aksel-allas]]:1```
6. Push the docker image to docker repository ```docker push eu.gcr.io/k8s-ws-12/[[demo-app_aksel-allas]]:1```
   1. If you have problems, run `gcloud auth configure-docker`
8. Mac M1 owners this is only for you: In previous step, you pushed arm64 build, but the k8s cluster is running on amd64 nodes. 
   This means that your application will crash once you apply the deployment. There are now two options for you:
    1. Try to build amd64 build locally, but this often fails: ```docker buildx build --push --platform  linux/amd64 --tag eu.gcr.io/k8s-ws-12/[[demo-app_aksel-allas]]:2 .```
    2. In the next step, when you specify the image to run, you could use a prebuilt one such as `demo-app_aksel-allas:1` 
## Step 3: Create deployment

Let's create a deployment, specifying pods (instances) count, liveness/readiness probes and update strategy.

Configure k8s context and namespace to be used for following commands
(to avoid passing `--context` and `--namespace` with each following command)
```shell
# see what namespaces you could use
kubectl get namespaces

# Set kubectl against certain namespace (default ns is the default, but we want to deploy to your own ns)
kubectl config set-context $(kubectl config current-context) --namespace=[[aksel-allas]]

# see all contexts and which of them is currently selected and what namespace is currently selected:
kubectl config get-contexts
```
> You should see exactly one context when executing following command to check that your namespace is configured for current context:
`kubectl config get-contexts | grep "k8s-ws-" | grep "*" | grep [[aksel-allas]]`

See the current state of k8s resources:
```shell
# You can get different types of resources at once
kubectl get pods,deployments,nodes

# before running next command you shouldn't have any pods
kubectl get pods
```

Create [deployment](deployment.yaml) (uploads manifest from given file to kubernetes)
```shell
# NB! you probably want to replace image reference with your own, but you could try with default as well
kubectl apply -f deployment.yaml
```

See if deployment created pod (hopefully in ContainerCreating and soon in Running status)
```shell
# now you should see one pod (in selected namespace of selected k8s cluster)
kubectl get pods

# Investigate events (specially if pod isn't in running status)
kubectl describe pod [[podname]]

# Investigate pod logs
kubectl logs [[podname]]
```

If you have managed to get pod into "Running" state, experiment with deleting:
```shell
# try deleting pod...
kubectl delete pod [[podname]]

# ... and see what happened
kubectl get pods
```

Try adding more pods of the same deployment:
```shell
# open deployment manifest for in-line editing (note this doesn't change your deployment.yaml)
kubectl edit deployment demo

#change `replicas: 2`, save and quit

# check number of pods: 
kubectl get pods
```


## Step 4: Create service

Let's create a [service](service.yaml), so all our healthy application pods would be accessible from same (non-public) endpoint of the service.

```shell
kubectl apply -f service.yaml
```

Get information about the service, pods and namespaces used later
```shell
# Check which services you have now
kubectl get svc
# Check the details of that service
kubectl describe svc demo
# Check the ip of your pod
kubectl get pods -o wide
# Check other namespaces
kubectl get namespaces
```

Log into one container...
```shell
# "log in" to the running container
kubectl exec -it [[podname]] -- /bin/sh
```
... and execute following commands from there:
```shell
# How to access your running java app inside the same container
curl localhost:8080/actuator/health

# How service is accessing your pod, note the port of 8080
# (theoretical, usually you don't need the pod ip at all)
curl [[somePodip]]:8080/actuator/health

# How to access your java app via service ip (not via DNS)
curl [[svc-cluster-ip:-changeMe]]/actuator/health

# How to access a service in your own namespace (DNS)
curl demo/actuator/health

# How to access a service in any namespace (DNS)
curl demo.[[aksel-allas]].svc.cluster.local/actuator/health
```


## Step 5: Create ingress

Let's make the service accessible from the public web (via IP-address/hostname).

Replace the public path name in [ingress.yaml](ingress.yaml) from `${yourName}` to *your name*.

```shell
kubectl apply -f ingress.yaml
```
> This ingress configuration for our service gets read by Ingress Controller
(that uses Nginx server with this specific ingress setup).
Ingress Controller has Nginx routing configuration, 
which is combined from ingress configurations (with rules for rewriting paths) of different services.

You should be able to see host address using either of following commands:
```shell
# now you should see one ingress (in selected namespace of selected k8s cluster)
kubectl get ingress
kubectl describe ingress demo
```

You should be able to access
`http://[[hostName]]/[[aksel-allas]]/actuator/health`
from public internet (i.e. using your browser or curl). The full url should look like `http://104.199.94.225.nip.io/aksel-allas/actuator/health`

> Note, on linux you can use `watch` to monitor changes of outputs of one or more commands:
> `watch "kubectl get ingress && kubectl describe ingress demo && curl http://[[hostName]]/[[yourName]]/actuator/health"`


## Step 6: Create autoscaler

Let's make our service scale horizontally based on actual usage.

[Autoscaler](autoscaler.yaml) works on comparing actual resource usage
(see `kubectl top pods`)
to requested resources (see deployment resources.requests).

```shell
kubectl apply -f autoscaler.yaml
```

Use following commands:
```shell
# to see autoscalers
kubectl get horizontalpodautoscalers

# to see pods CPU and memory usage
kubectl top pods
```

Watch what happens to pods and autoscaler:
```
# on linux you can use `watch` to evaluate expression periodically:
watch "kubectl top pods && kubectl get pods,horizontalpodautoscalers"
```

In another console generate load to your service with following commands

1. Create [loadtest python script](loadtest.py)
2. Run **locust** locally ```docker run -p 8089:8089 -v $PWD:/mnt/locust locustio/locust -f /mnt/locust/loadtest.py```
3. Open browser `http://localhost:8089` and specify 100 users, 10 seconds and your public url in the host such as `http://104.199.94.225.nip.io/aksel-allas/actuator/health`

Now back in the watch terminal you should soon see an increase in CPU usage and after about half minute you should see effects of autoscaler.

## Step 7: Create configmap

Let's attach configmap as file to our containers.
> When creating deployment we provided some configuration values via environment variables.
> Using secrets would be another option (out of the scope for this WS).

Create configuration source file for k8s configmap, for example `some.conf`:
```properties
test=1
props=2
```

Use that configuration file to create configmap:
```shell
kubectl create configmap demo --from-file=some.conf
```

Inspect configmaps:
```shell
kubectl get configmaps
kubectl describe configmap demo
```

Update your deployment.yaml with configMap mounted from volume:
```diff
             httpGet:
               path: /actuator/health
               port: 8080
+          volumeMounts:
+            - mountPath: /conf
+              name: conf
+      volumes:
+        - name: conf
+          configMap:
+            name: demo
+            items:
+              - key: some.conf
+                path: some.conf
```

Apply changes in deployment:
```shell
kubectl apply -f deployment.yaml
```

Check that mount and volume appears in deployment description:
```shell
kubectl describe deployment demo | grep ount -A 6
```

Log into running container...
```shell
kubectl get pods
# "log in" to the running container
kubectl exec -it [[podname]] -- /bin/sh
```
... and check if conf was actually mounted as file by executing following commands:
```shell
# should see the same conf you created
cat /conf/some.conf
```
