### Terraform: Provision GKE Cluster

The Terraform templates were taken from the repo,
<https://github.com/hashicorp/learn-terraform-provision-gke-cluster>, a
companion repo to the [Provision a GKE Cluster (Google
Cloud)](https://learn.hashicorp.com/terraform/kubernetes/provision-gke-cluster),
containing Terraform configuration files to provision an GKE cluster on GCP.

This repo also creates a VPC and subnet for the GKE cluster. This is not
required but highly recommended to keep your GKE cluster isolated.

### Quick Start

```bash
terraform apply
gcloud container clusters get-credentials $(terraform output -raw kubernetes_cluster_name) --zone $(terraform output -raw zone)
```

### [Creating the NGINX Ingress Controller on GKE](https://kubernetes.github.io/ingress-nginx/deploy/#gce-gke)

Setting up the load balancer is as simple as `kubectl apply`. I have customized:

```bash
kubectl apply -f nginx-ingress-controller.yml
```
Check the installed version
```bash
POD_NAMESPACE=ingress-nginx
POD_NAME=$(kubectl get pods -n $POD_NAMESPACE -l app.kubernetes.io/name=ingress-nginx --field-selector=status.phase=Running -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -n ingress-nginx -- /nginx-ingress-controller --version # --help is useful, too
```

### [TLS (cert-manager)](https://cert-manager.io/docs/installation/kubernetes/)

We choose to install with regular manifests (not `helm`):
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.4.0/cert-manager.yaml
```

(I did not seem to run into the GKE `permission denied` error that they warn
about). Let's check that the 3 pods are up & running:

```bash
kubectl get pods --namespace cert-manager
```

Now let's create an issuer to test the webhook:
```bash
cat <<EOF > test-resources.yaml
```
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
```
```
EOF
```

And now let's apply those resources:
```bash
kubectl apply -f test-resources.yaml
kubectl describe certificate -n cert-manager-test
kubectl delete -f test-resources.yaml
```

#### 4. [Deploy an Example Service](https://cert-manager.io/docs/tutorials/acme/ingress/#step-4-deploy-an-example-service)

Let's install the sample services to test the controller:
```bash
kubectl apply -f https://netlify.cert-manager.io/docs/tutorials/acme/example/deployment.yaml
kubectl apply -f https://netlify.cert-manager.io/docs/tutorials/acme/example/service.yaml
```

Let's download and edit the Ingress (I've already configured `gke.nono.io` to
point to the GCP/GKE load balancer at 104.155.144.4):
```bash
curl -o ingress-kuard.yml -L https://netlify.cert-manager.io/docs/tutorials/acme/example/ingress.yaml
sed -i '' "s/example.example.com/gke.nono.io/g" ingress-kuard.yml
kubectl apply -f ingress-kuard.yml
```

Let's use curl to check (note the cert is still self-signed at this point):
```bash
curl -kivL -H 'Host: gke.nono.io' 'http://104.155.144.4'
```

#### 6. [Configure Let’s Encrypt Issuer](https://cert-manager.io/docs/tutorials/acme/ingress/#step-6-configure-let-s-encrypt-issuer)

Let's deploy the staging & production issuers:
```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/staging-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/production-issuer.yaml |
  sed 's/user@example.com/brian.cunnie@gmail.com/')
kubectl describe issuer letsencrypt-staging
kubectl describe issuer letsencrypt-prod
```

#### 7. [Step 7 - Deploy a TLS Ingress Resource](https://cert-manager.io/docs/tutorials/acme/ingress/#step-7-deploy-a-tls-ingress-resource)

Let's deploy the ingress resource using annotations to obtain the certificate:
```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

Browse to <https://gke.nono.io> and notice that although the cert is still
invalid it's no longer self-signed; instead, it's issued by the Let's Encrypt
staging CA.

Let's do the production certificate:

```bash
kubectl apply -f <(
  curl -o- https://cert-manager.io/docs/tutorials/acme/example/ingress-tls-final.yaml |
  sed 's/example.example.com/gke.nono.io/')
kubectl delete secret quickstart-example-tls # triggers the process to get a new certificate
kubectl get certificate # takes ~30s to become ready ("READY" == "True")
kubectl describe certificate quickstart-example-tls
kubectl describe secret quickstart-example-tls
```

And now browse: <https://gke.nono.io/>

### Install sslip.io

```bash
kubectl apply -f <(curl -L https://raw.githubusercontent.com/cunnie/sslip.io/master/k8s/sslip.io.yml)
```

### Install Concourse CI

These instructions are patterned after
<https://github.com/concourse/concourse-chart>.

```bash
kubectl apply -f concourse.yml
helm repo add concourse https://concourse-charts.storage.googleapis.com/
helm install ci-nono-io concourse/concourse \
  -f concourse-values.yml \
  --set secrets.githubClientSecret=$(lpass show --note deployments.yml | yq e .github_concourse_nono_auth_client_secret -) \
  --set secrets.hostKey="$(lpass show --note deployments.yml | yq e .tsa_host_key.private_key -)" \
  --set secrets.hostKeyPub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDrchtcCq6WFbl2xlWaKQP1UIUDUPPjKrndtrPXFArPs4+zOn5RcGf4zpO2GUD8fZl+8tikG7b+YQTfyOB08zeuA+WqpBVmiaXyK7OJhzuEWwqa60p5Ni1SyNRtcgntY8DLkKnWqzhDaVT/FcXIbnDfyMCDxp7Gs023jha3IGKeIIhRsOkJDcsfByxF63GP70WEs49JNToDCC3CIo8JEGXunjF1matILpJhupsa3obMOk2OCGNI9nleiRfSjE51f9hzYAa1wKqCoBbgOtVQ3mz59yxTFobVZFBP6fZX2GWXaLWHPPiAUtMhiL87pHsa43K0iiV6Yk59yoZ67mOdachp web@ci.nono.io" \
  --set secrets.workerKey="$(lpass show --note deployments.yml | yq e .worker_key.private_key -)" \
  --set secrets.workerKeyPub="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC0SIrGT+qIE7w8i67B/YDCfHINEU0LUP67SesaPaesq26rb/HHckPvBfRj+gCxKMvmTipUIVaQLBZlsPEMb+1V8xJBs2s4+9MU6QG6i7CEYTWyYlhVSDxU4HtwxGGnW9c5lASBB1jPkx2gWv0kgQYXQfrcbXSJ4fUtgdo0ZtePnXV7Qd30YUoR2fcuqEdAGg0S317V54vgeD2tfL04Qwhyu2Hbz4ZwTyhNe1YNcKET6v8ttRjVOIfMe+FF+JHqGJUiu5jygJ2p+29sm50JHEuxK+HjYpajmW8BRmXK7fIvDX24RDIs9ACZ+s+asEU8yEKkmdnFB5kUAukQWRuqUWYd worker@ci.nono.io"
```

### Updating Concourse CI

Get the latest release version number from
<https://hub.docker.com/r/concourse/concourse/tags/?page=1&ordering=last_updated>.

```bash
NEW_RELEASE="7.4.0"
sed -i '' "s/^imageTag: .*/imageTag: $NEW_RELEASE/" concourse-values.yml
```
```bash
helm upgrade ci-nono-io concourse/concourse \
  -f concourse-values.yml \
  --wait
helm show chart concourse/concourse # to check that it's upgrading
# helm pull concourse/concourse # I don't know if I need this; delete this line if I don't
```