![spaceinvader](https://www.myabandonware.com/media/screenshots/s/space-invaders-44p/space-invaders_4.png)

# This repo is to install Space invader into K8s for killing pods

## Key features

* cert-manager
* free domain on https://my.freenom.com/ domain name killpods.ga-- check video for know issue: https://www.youtube.com/watch?v=ubBCeUNHzPs&t=89s&ab_channel=tipswithpunch
* CI wuth GitHub actions


## Possible Improvements or Security issue

* Please use existing Cluster from the Perpetual team to make sure to have proper security in place such as RBAC, secret mgment and much more ...

## Pre-requisites

1. Cluster - please check https://github.com/Maersk-Global/aks-module/tree/dev to use AKS terraform module
  1. Clone above repo and use Terragrunt command to create your cluster
  2. Attached ACR repo to cluster
    az aks update -n cilium-tutorial-1 -g rgpazewsmlit-sandbox-pgr095-001 --attach-acr pgr065

2. kubctl

3. Docker

## Installed your web application using manifest (docs from website use Helm)

### Ingress Controller and Resource (Component (ie pod) and rules)

We’ll begin by first creating the Kubernetes resources required by the Nginx Ingress Controller. These consist of ConfigMaps containing the Controller’s configuration, Role-based Access Control (RBAC) Roles to grant the Controller access to the Kubernetes API, and the actual Ingress Controller Deployment which uses v0.35.0 of the Nginx Ingress Controller image. To see a full list of these required resources, consult the manifest from the Kubernetes Nginx Ingress Controller’s GitHub repo.

To create these mandatory resources, use kubectl apply and the -f flag to specify the manifest file hosted on GitHub:

``` bash

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.4.0/deploy/static/provider/cloud/deploy.yaml

```

Confirm that the Ingress Controller Pods have started

``` bash

$ kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx

```

Check that the Azure LoadBalancer has been created

``` bash

$ kubectl get svc --namespace=ingress-nginx

```
You should see

``` bash

Output
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)                   AGE
ingress-nginx   LoadBalancer   10.245.247.67   203.0.113.0   80:32486/TCP,443:32096/TCP   2m

```

We can now point our DNS records at this external Load Balancer and create some Ingress Resources to implement traffic routing rules.

### Create Service and Deployment using manifest

The repository contains a service yaml file, please modify the image value to your own repository.

``` bash

$ kubernetes apply -f kubeinvaders-deployment.yaml            
namespace/kubeinvaders created
service/kubeinvaders created
deployment.apps/kubeinvaders created

```

check your newly service

``` bash

$ kubectl -n kubeinvaders get svc kubeinvaders
NAME           TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubeinvaders   ClusterIP   10.0.82.157   <none>        8080/TCP    1m

```
You can check your service using your browser

``` bash

$ kubectl port-forward svc/kubeinvaders -n kubeinvaders 12345:8080 &
[1] 14467

``` 
It will start the forwarding service in the background, remember the output number (or use the command jobs) if you want to stop it.
You should be able to see your site on http://localhost:12345

### Create NGinx resource

``` bash

$ kubectl apply -f kubeinvaders-ingress.yaml

``` 

The rules of your ingress created, you will be able to go to your dns. The next step is to make sure that your site use ssl and port 80 is redirected to to port 443 (secured port)

## Cert-manager for services

The certificate will be issued by Lets Encrypt, cert-manager CRD will take care of renewing the certificate.

### Installing and Configuring Cert-Manager for Staging certificate

We’ll install cert-manager and its Custom Resource Definitions (CRDs) like Issuers and ClusterIssuers. Do this by applying the manifest directly from the cert-manager GitHub repository :

``` bash

$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.10.0/cert-manager.yaml

```

Verify our installation by checking cert-manager Namespace for running pods:

``` bash

$ kubectl get pods -n cert-manager

```

Output should look like this:

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-749df5b4f8-94nvn              1/1     Running   0          71s
cert-manager-cainjector-67b7c65dff-x2zfs   1/1     Running   0          71s
cert-manager-webhook-7d5d8f856b-mfdtd      1/1     Running   0          71s

Before we begin issuing certificates for our Ingress hosts, we need to create an Issuer, which specifies the certificate authority from which signed x509 certificates can be obtained. In this guide, we’ll use the Let’s Encrypt certificate authority, which provides free TLS certificates and offers both a staging server for testing your certificate configuration, and a production server for rolling out verifiable TLS certificates.

### Create an Issuer to test the webhook works okay.

Create the test resources by using provided test resource file.

``` bash

$ kubectl apply -f test-resources.yaml

```

Check the status of the newly created certificate. You may need to wait a few seconds before cert-manager processes the certificate request.

``` bash

$ kubectl describe certificate -n cert-manager-test

```

Output:
...
Spec:
  Common Name:  example.com
  Issuer Ref:
    Name:       test-selfsigned
  Secret Name:  selfsigned-cert-tls
Status:
  Conditions:
    Last Transition Time:  2019-01-29T17:34:30Z
    Message:               Certificate is up to date and has not expired
    Reason:                Ready
    Status:                True
    Type:                  Ready
  Not After:               2019-04-29T17:34:29Z
Events:
  Type    Reason      Age   From          Message
  ----    ------      ----  ----          -------
  Normal  CertIssued  4s    cert-manager  Certificate issued successfully

Clean up the test resources.

``` bash

$ kubectl delete -f test-resources.yaml

```

If all the above steps have completed without error, you are good to go!


### Rolling Out Production Issuer
In this step we’ll modify the procedure used to provision staging certificates, and generate a valid, verifiable production certificate for our Ingress hosts.

To begin, we’ll first create a production certificate ClusterIssuer.

Open the file called prod_issuer.yaml in your favorite editor to check the syntax.

Note the different ACME server URL, and the letsencrypt-prod secret key name.

Now, roll out this Issuer using kubectl:

``` bash

$ kubectl create -f prod-issuer.yaml

```

You should see the following output:

Output

clusterissuer.cert-manager.io/letsencrypt-prod created

Update kubeinvaders-ingress.yaml to use this new Issuer, commenting out tls lines.

Once you’re satisfied with your changes, save and close the file.

Roll out the changes using kubectl apply:

``` bash

$ kubectl apply -f kubeinvaders-ingress.yml

```
Wait a couple of minutes for the Let’s Encrypt production server to issue the certificate. You can track its progress using kubectl describe on the certificate object:

``` bash

$ kubectl describe certificate echo-tls

```

Output

Status:
Conditions:
Last Transition Time:  2021-04-23T12:49:59Z
Message:               Certificate is up to date and has not expired
Reason:                Ready
Status:                True
Type:                  Ready
Not After:               2020-10-23T12:49:59Z
Events:
Type    Reason        Age                From          Message
----    ------        ----               ----          -------k apply -f 
Normal  GeneratedKey  20m                cert-manager  Generated a new private key
Normal  Requested     20m                cert-manager  Created new CertificateRequest resource <"tls name">
Normal  Requested     36s                cert-manager  Created new CertificateRequest resource <"tls name">
Normal  Issued        10s (x2 over 20m)  cert-manager  Certificate issued successfully

Check your HTTPS either using curl or browser.
