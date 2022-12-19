# Styarburst Addon Operator

**TOC**
- [scaffolding](#scaffolding)
- [local development](#local-development)
- [Staging Testing](#staging-testing)
- [Staging Testing Detailed](#staging-testing-detailed)
- [Expected Secrets](#expected-secrets)
- [Parameters for Generic Operator](#parameters-for-generic-operator)
- [Helpful Links](#helpful-links)

## Scaffolding

The operator was created with `operator-sdk` using the following parameters.  

```bash
operator-sdk init --domain redhat.com --repo github.com/RHEcosystemAppEng/starburstaddon-operator 

operator-sdk create api --group managed-tenants --version v1alpha1 --kind StarburstAddon --resource --controller
```

## Local Development

This section it dedicated to developing and running the operator locally. The operator expects Prometheus CRDs and secrets to be deployed.  

Prereqs:
- Change line 32 of `Makefile` to your dockerhub username

The order of options for local development is the following:
- Create kind cluster
- Install Prometheus CRDs in the kind cluster
- Install secrets in the kind cluster that would come from vault and userParameters
- Build and Deploy the Operator


**Create Environment**
_in one terminal_
```bash
kind delete cluster --name=operator; # delete previous cluster
docker system prune -a -f; # prune docker (at your own risk)
kind create cluster --name=operator; 


kubectl create ns redhat-starburst-operator
kubectl config set-context $(kubectl config current-context) --namespace=redhat-starburst-operator

# Neededed for Prometheus to come up (Just a work around for dev)
kubectl create sa starburst-enterprise-helm-operator-controller-manager
kubectl create clusterrolebinding starb-admin --clusterrole=cluster-admin --serviceaccount=addon-operator-system:starburst-enterprise-helm-operator-controller-manager

make generate;

make manifests;

make docker-build docker-push;

# Prometheus Dependencies
kubectl apply -f \
https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/bundle.yaml -n redhat-starburst-operator

# Prometheus CR is too long, this is the workaround
kubectl apply \
-f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/v0.52.0/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml \
--force-conflicts=true \
--server-side \
-n redhat-starburst-operator

#error- The CustomResourceDefinition "addons.addon.redhat.com" is invalid: metadata.annotations: Too long: must have at most 262144 bytes 
kubectl apply -f config/crd/bases/addon.redhat.com_addons.yaml --force-conflicts=true \
  --server-side

# Deploy Addon Operator
make deploy;


```


_after you create the environment_

**Watch operator logs**
   
In one terminal...   

```bash
kubectl logs -l control-plane=controller-manager -f
```

**Deploy userParam, vault Secrets, and Instance of AddonOperator**

In another terminal...

```bash
kubectl apply -f stage # Get the staging folder from https://github.com/cmwylie19/addon-operator/tree/main/stage
```
## Staging Testing

1. Add a pull secret to a OSD 4.11 cluster [Link](https://marketplace.redhat.com/api-security/en-us/login/landing)
2. Install Starburst Addon in OSD v4.11 from tile
3. Delete Prometheus, ServiceMonitors, and StarburstEnterprise
4. Install the bundle without `dependencies.yaml`
5. Manually deploy the bundled `operator-sdk run bundle..`
5. Deploy staging folder `kubectl deploy -f staging/` 
  5.1 Get the staging folder from https://github.com/cmwylie19/addon-operator/tree/main/stage

## Staging Testing Detailed

Build and Push Bundle (prior to this the operator image **must** be built `make docker-build docker-push;`)

```bash
docker system prune -a -f 

make generate;

make manifests;

make docker-build docker-push;

make bundle bundle-build bundle-push 
```

Run the Bundle in OSD

```bash
kubectl create ns redhat-starburst-operator

operator-sdk run bundle quay.io/cwylie1/addon-operator-bundle:v0.0.1

kubectl config set-context $(k config current-context) --namespace=redhat-starburst-operator
```

Remove 

```bash
kubectl delete csv,subs --all --force 

kubectl delete prometheus --all -n redhat-starburst-operator

kubectl patch starburstenterprise/starburstenterprise -p '{"metadata":{"finalizers":[]}}' -n redhat-starburst-operator --type=merge

kubectl  delete po --force --grace-period=0  --all

kubectl delete ns redhat-starburst-operator
```



_after you create the environment_

**Watch operator logs**
   
In one terminal...   

```bash
kubectl logs -l control-plane=controller-manager -f
```

**Deploy userParam, vault Secrets, and Instance of AddonOperator**

In another terminal...

```bash
kubectl apply -f stage
```

## Expected Secrets
Names:
- addon-managed-starburst-parameters (Params)
- managed-addon (Vault)

Params:
- Operand
- Scrape Labels

Vault
- Client ID
- Client Secret
- TokenURL
- RemoteWriteUrl

## Parameters for Generic Operator

- User Parameter addon secret
- License 


## Helpful Links
- [docs](https://docs.google.com/spreadsheets/d/1EQZaUm8s-QwwYwKyFv2tZze46YfcxpBzVeAYAI6fwF8/edit?pli=1#gid=868520042)  

**Helpful links**
[nvidia](https://github.com/rh-ecosystem-edge/nvidia-gpu-addon-operator/blob/53015d8932f54eb34e1f76b01ced9e1b03d493f9/controllers/monitoring/monitoring_controller.go)
[Prometheus Operator]()
[memcachedcontroller](https://github.com/operator-framework/operator-sdk/blob/latest/testdata/go/v3/memcached-operator/controllers/memcached_controller.go)

