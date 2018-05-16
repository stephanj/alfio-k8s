# alfio-k8s

Documentation on how to setup ALF.io on Google Cloud using Kubernetes with NGINX Ingress controller and self renewing 
Let's Encrypt HTTPS certificates. 

## Pre-requisites
You have to install:

* Docker
* kubectl
* Google Cloud SDK since we will perform most of the operations from a terminal


## Initialize gcloud and kubectl

First of all, if you never used gcloud, you need to initialize it with the following command:

```
 $ gcloud init
```

gcloud allows you to perform most the operations you could do from the Google Cloud Web console from the comfort of your terminal. 

Now install kubectl

```
 $ gcloud components install kubectl
```

kubectl is a command line interface for running commands against Kubernetes clusters. You can also install it directly from the Kubernetes website.

Now you need to create a google cloud project. For this purpose you will need to go through the web console, as gcloud does not allow you to create projects from CLI (not yet as it is an alpha feature). Alternatively you can use the Resource Manager API.

1. Go to Google Cloud Platform Console

2. Click Create Project

3. Pick a project name, click Create and note the project ID, and/or customize as you feel.


Finally you need to tell gcloud on which project you are currently working:

```
 $ gcloud config set project alfio
```

You can also tell it where you want your instances to be created by default. 

```
 $ gcloud config set compute/zone europe-west1-b
```

## Create a container cluster

Let's create a Google container cluster using gcloud.

```
 $ gcloud container clusters create alfio --zone=europe-west1-b --machine-type=g1-small --num-nodes=1
```

We'll use only 1 small node. In production, you will want at least 3 nodes.

Now lets get the kubeconfig setup so we can use kubectl.

```
 $ gcloud container clusters get-credentials alfio

Fetching cluster endpoint and auth data.
kubeconfig entry generated for alfio.
```

## Install Helm

Helm is a tool that streamlines installing and managing Kubernetes applications. Think of it like apt/yum/homebrew for Kubernetes.
More info @ https://github.com/kubernetes/helm

We'll need helm for the SSL certificate manager installation.

Helm has two parts: a client (helm) and a server (tiller).

Tiller runs inside of your Kubernetes cluster, and manages releases (installations) of your charts.

Install the Helm client on your development machine:

https://docs.helm.sh/using_helm/#installing-helm-client

Install the Helm server-side components (Tiller) on your GKE cluster:

```
 $ kubectl create serviceaccount -n kube-system tiller
```

```
 $ kubectl create clusterrolebinding tiller-binding \
    --clusterrole=cluster-admin \
    --serviceaccount kube-system:tiller
```

```
 $ helm init --service-account tiller
```

Once tiller pod becomes ready, update chart repositories:

```
 $ helm repo update
```  

## Install NGINX Ingress controller

We'll use an NGINX Ingress controller (with RBAC enabled) which will also enforce HTTPS redirects.

The following helm command is used to install our nginx-ingress controller: 

```
 $ helm install --name nginx-ingress stable/nginx-ingress --set rbac.create=true
```

The install command will configure two kubernetes  services, an nginx-ingress-controller and nginx-ingress-default-backend. 

```
 $ kubectl get svc

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller        LoadBalancer   10.51.251.184   XX.XXX.XX.XX    80:30946/TCP,443:30653/TCP   5m
nginx-ingress-default-backend   ClusterIP      10.51.250.129   <none>          80/TCP                       5m
```

The external IP address mentioned in the output (XX.XXX.XX.XX) is the one you'll use to link your DNS domain names to.

In our example you'll need to create a DNS A Record www.alfio.com and point it to whatever value you see for XX.XXX.XX.XX. 

 
## Install cert-manager

We want to have an auto renewable SSL certificate using Let's Encrypt.  
For this we need to install a cert-manager, at the time of this writing it's "pre-stable" software, so use it at your own risk.
 
We'll install this again using helm:

```
$ helm install --name cert-manager --namespace kube-system stable/cert-manager
```

You can verify if the cert-manager is running with the following command:

```
 $ kubectl get pod -n kube-system

NAME                                                   READY     STATUS    RESTARTS   AGE
...
cert-manager-cert-manager-5f65b75bb-r682c              2/2       Running   0          1d
...
```

## Set up Let‘s Encrypt

To have Let’s Encrypt provide your certs, you need to create an Issuer (namespace-scoped) or a ClusterIssuer (cluster-wide) resource on Kubernetes.

First, set your email address in a variable (this is to get expiry notifications from Let's Encrypt, and is required by their Terms of Service):

```
 $ export EMAIL=[YOUR EMAIL HERE]
```

Now run this to deploy the Issuer manifests (the command below will populate your email address in the manifest file):

```
 $ curl -sSL https://rawgit.com/ahmetb/gke-letsencrypt/master/yaml/letsencrypt-issuer.yaml | \
    sed -e "s/email: ''/email: $EMAIL/g" | \
    kubectl apply -f-
```

You will see output:

```
clusterissuer "letsencrypt-staging" created
clusterissuer "letsencrypt-prod" created
```

Let's Encrypt has both staging and production endpoints. 
You should use the staging environment to test the automation out. 
Once you get things working, you can switch to the production environment.

CREDITS: The above curl command uses an issuer yaml template from Ahmet Alp Balkan, which has done an amazing job documenting this process. 
Everything from the cm-manager and let's encrypt comes from his documentation @ https://github.com/ahmetb/gke-letsencrypt     
    
## Get a certificate for your domain name

We need to create a certificate yaml file which will request a certificate for a domain name from the letsencrypt-prod issuer:

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: www-alfio-com-tls
  namespace: default
spec:
  secretName: www-alfio-com-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: www.alfio.com
  dnsNames:
  - www.alfio.com
  acme:
    config:
    - http01:
        ingress: alfio
      domains:
      - www.alfio.com
```

In the above example we'll ask a certificate for the domain name www.alfio.com  

Modify a few things before deploying this manifest:

* Replace www.alfio.com with your domain name
* Replace www-alfio-com-tls (will be used to create the TLS Secret) with a name that is suitable
* Replace 'alfio' with the Ingress name that your website is running on

Then apply this manifest:

```
 $ kubectl apply -f certificate.yaml
```

This will take some time. 

What's happening behind the scenes:
 
* cert-manager updates your Ingress to handle GET /.well-known/acme-challenge/* requests with a temporary Service it created in your cluster. This will be used to prove that you own the domain name.
* You can run the following command to see how it is modified.

```
 $ kubectl get ingress -o=yaml alfio 
```

* Since Ingress is updated, the Google Cloud Load Balancer is being updated too!
* It will take about 5-10 minutes for the changes to take effect.
* After Ingress changes take effect, cert-manager will notice that the /.well-known/* URL starts working.
* cert-manager will ask Let's Encrypt to provide certificates.
* Let's Encrypt will come and visit /.well-known/* URL to see the proof that you own the domain name.
* Let's Encrypt will provide you certificates.
* cert-manager will save the TLS certificates to the specified secretName.

When it is complete, you will see the specified secretName in the Secrets list:
 
```
 $ kubectl get secrets
 
 NAME               TYPE                DATA      AGE
 www-alfio-com-tls  kubernetes.io/tls   2         4m  
``` 

You can also look at the status of the Certificate resource you just created:

``` 
 $ kubectl describe -f certificate.yaml

 Type     Reason                Message
 ----     ------                -------
 Warning  ErrorCheckCertificate Error checking existing TLS certificate: secret "www-alfio-com-tls" not found
 Normal   PrepareCertificate    Preparing certificate with issuer
 Normal   PresentChallenge      Presenting http-01 challenge for domain foo.kubernetes.tips
 Normal   SelfCheck             Performing self-check for domain www.dogs.com
 Normal   ObtainAuthorization   Obtained authorization for domain www.dogs.com
 Normal   IssueCertificate      Issuing certificate...
 Normal   CeritifcateIssued     Certificated issued successfully
 Normal   RenewalScheduled      Certificate scheduled for renewal in 1438 hours
```
 
When you see the "CeritificateIssued" event, it means it has worked!
 
Congrats, you now have a TLS certificate for your domain!

## Start serving HTTPS traffic

You need to update your Ingress to add a tls section under the spec with the secretName that stores the TLS certificate and the domain name.

Because we'll start two ALF.io instances we also need to enable 'sticky sessions' using cookies.  

The following annotations make this happen:

```
nginx.ingress.kubernetes.io/affinity: "cookie"
nginx.ingress.kubernetes.io/session-cookie-name: "route"
nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
```

An example Ingress yaml file looks as follows: 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: alfio
  namespace: default
  annotations:
      kubernetes.io/ingress.class: "nginx"
      nginx.ingress.kubernetes.io/affinity: "cookie"
      nginx.ingress.kubernetes.io/session-cookie-name: "route"
      nginx.ingress.kubernetes.io/session-cookie-hash: "sha1"
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - www.alfio.com
    secretName: www-alfio-com-tls
  rules:
  - host: www.alfio.com
    http:
      paths:
      - path: /
        backend:
          serviceName: alfio
          servicePort: 8080
```

## The ALF.io Kubernetes Service 

The service yaml file

```
apiVersion: v1
kind: Service
metadata:
  name: alfio
  namespace: default
  labels:
    app: alfio
spec:
  selector:
    app: alfio
  type: NodePort
  ports:
  - name: web
    port: 8080
```

## The ALF.io Kubernetes Deployment

The example deployment yaml file will start 2 replicas and retrieve the Docker image from 'alfio/alf.io'.

When using Google Cloud Registry the image link would be something like

```
gcr.io/[PROJECT NAME]/[IMAGE NAME]:[VERSION]
```

for example

```
gcr.io/devoxx-registration/alfioadapter:v1.9.3    
```

The example deployment uses a PostgreSQL database using a Google Cloud SQL managed instance.

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: alfio
  namespace: default
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: alfio
    spec:
      containers:
      - name: alfio-app
        image: alfio/alf.io
        imagePullPolicy: IfNotPresent
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: jdbc-session
        - name: POSTGRES_PORT_5432_TCP_PORT
          value: "5432"
        - name: POSTGRES_ENV_POSTGRES_DB
          value: alfio
        - name: POSTGRES_PORT_5432_TCP_ADDR
          value: "127.0.0.1"
        - name: POSTGRES_ENV_POSTGRES_USERNAME
          valueFrom:
            secretKeyRef:
              name: alfio-postgresql-credentials
              key: username
        - name: POSTGRES_ENV_POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: alfio-postgresql-credentials
              key: password
        - name: JAVA_OPTS
          value: "-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:+UseConcMarkSweepGC -Xmx256m -Xms256m"
        resources:
          requests:
            memory: "256Mi"
            cpu: "300m"
          limits:
            memory: "512Mi"
            cpu: "1"
        ports:
        - name: web
          containerPort: 8080 
        readinessProbe:
          httpGet:
            path: /healthz
            port: web
        livenessProbe:
          httpGet:
            path: /healthz
            port: web
          initialDelaySeconds: 180          
```

You should consider using Kubernetes secrets for the database password but it's out-of-scope of this documentation.

I need to check with the ALF.io team if they have a 'health' endpoint which we can use for the readiness and liveness Probe. 

## The MySQL Deployment

For a staging environment you can use the following MySQL yaml file, however for production make sure to use a Cloud SQL 
instance instead.

Also ALF.io will drop MySQL support moving forward and demand a Postgres database instead.   

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: alfio-mysql
  namespace: default
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: alfio-mysql
    spec:
      volumes:
      - name: data
        emptyDir: {}
      containers:
      - name: mysql
        image: mysql:5.7.20
        env:
        - name: MYSQL_USER
          value: root
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: 'yes'
        - name: MYSQL_DATABASE
          value: alfio
#        command:
#        - mysqld
#        - --lower_case_table_names=1
#        - --skip-ssl
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql/
---
apiVersion: v1
kind: Service
metadata:
  name: alfio-mysql
  namespace: default
spec:
  selector:
    app: alfio-mysql
  ports:
  - port: 3306
```

## Deploy everything

You can place the kubernetes yaml files in a sub-directory named 'k8s' and apply them using the following command

```
 $ kubectl apply -f k8s   
```

To verify the kubernetes pod status, run the following command:

```
 $ kubectl get pod

NAME                                             READY     STATUS    RESTARTS   AGE
alfio-58f7587685-688r8                           2/2       Running   0          1h
alfio-58f7dd3768-68dr8                           2/2       Running   0          1h
alfio-mysql-55b5cc54ff-vg72v                     1/1       Running   0          1h
nginx-ingress-controller-5644dd49d-jswb6         1/1       Running   0          1d
nginx-ingress-default-backend-58bf6f478b-6hhz2   1/1       Running   0          1d
```

Listing the kubernetes services, should give you the following

```
 $ kubectl get svc

NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)                      AGE
alfio                           NodePort       10.51.248.136   <none>          8080:31762/TCP               1d
alfio-mysql                     ClusterIP      10.51.245.130   <none>          3306/TCP                     1d
kubernetes                      ClusterIP      10.51.240.1     <none>          443/TCP                      2d
nginx-ingress-controller        LoadBalancer   10.51.251.184   XX.XXX.XXX.XX   80:30946/TCP,443:30653/TCP   1d
nginx-ingress-default-backend   ClusterIP      10.51.250.129   <none>          80/TCP                       1d
```

And the kubernetes ingress command should show you the alfio host

```
 $ kubectl get ing

NAME           HOSTS                 ADDRESS         PORTS     AGE
alfio          com.alfio.com         YY.YYY.YYY.YY   80, 443   1d
```

Now you can start using ALF.io on Kubernetes using HTTPS 

Enjoy!

Stephan Janssen
   


