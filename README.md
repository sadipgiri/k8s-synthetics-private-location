# Deploying a Datadog Private Location on Kubernetes

This repo is a simple example on how to deploy a Kubernetes Pod which acts as the private-location worker used for [Synthetics Private Locations](https://docs.datadoghq.com/synthetics/private_locations/#overview)

## Requirements

Having access to a Kubernetes Cluster to deploy the `private-worker-pod`.
If you don't you can simply spin up a one-node cluster with minikube by executing the following :

```
minikube start -p private_locations
```

If you do not have minikube installed yet, please refer to [this doc](https://kubernetes.io/docs/setup/learning-environment/minikube/).

## Setup

1. Run git clone and cd into the working directory

```
git clone git@github.com:hfaivre/k8s-synthetics-private-location.git
cd ./k8s-synthetics-private-location
```

2. Deploy the Datadog agent :

Start by copy pasting the output of running `echo -n <YOUR_DATADOG_API_KEY> | base64` under the `api-key` key in the `datadog/datadog.yaml` file :

```
apiVersion: v1
kind: Secret
metadata:
  name: datadog-secret
  labels:
    app: "datadog"
type: Opaque
data:
  api-key: "<YOUR_ENCODED_KEY_HERE>"
```

Then apply the following manifests:

```
kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrole.yaml"

kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/serviceaccount.yaml"

kubectl create -f "https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/rbac/clusterrolebinding.yaml"

kubectl apply -f datadog/
```



3. Create a new Private Location by clicking on "Add private location" at [app.datadoghq.com/synthetics/settings](app.datadoghq.com/synthetics/settings). Input Name, Description and Tags and click on "Save & Generate"
4. Create your Private Location JSON config file in your working directory by copy pasting the command displayed in the UI.
5. Create a Kubernetes ConfigMap with the previously created json file by executing the following :

```
kubectl create configmap private-worker-config --from-file=<worker-config-file>.json
```

6. Edit the `private-worker-pod.yaml` and modify `<MY_WORKER_CONFIG_FILE_NAME>.json` with the name of your Private Location JSON config file in the `subPath`section.
7. Run `kubectl apply -f private-worker-pod.yaml`



The steps above will setup a private location worker pod reporting logs to your Datadog account!

## Adding additional arguments

The Private Location Worker image accepts the following arguments :

```

Options:
  --accessKey         Access Key for Datadog API authentication  [string]
  --secretAccessKey   Secret Access Key for Datadog API authentication  [string]
  --dnsUseHost        Use local DNS config in addition to --dnsServer (currently ["192.168.65.1"])  [boolean] [default: false]
  --dnsServer         DNS server IPs used in given order (--dnsServer="1.1.1.1" --dnsServer="8.8.8.8")  [array] [default: ["8.8.8.8","1.1.1.1"]]
  --blacklistedRange  Deny access to IP ranges (e.g. --blacklistedRange.4="127.0.0.0/8" --blacklistedRange.6="::1/128")  [array] [default: IANA IPv4/IPv6 Special-Purpose Address Registry]
  --whitelistedRange  Grant access to IP ranges (has precedence over --blacklistedRange)  [array] [default: none]
  --help              Show help  [boolean]
  --config            Path to JSON config file
  --site              Datadog site (datadoghq.com or datadoghq.eu)  [string] [required] [default: "datadoghq.com"]
  --proxy             Proxy URL  [string]
  --logFormat, -f     Format log output  [choices: "pretty", "json"] [default: "pretty"]
  --verbosity, -v     Verbosity level (e.g. -v, -vv, -vvv, ...)  [number] [default: 3]

Advanced options:
  --concurrency             Maximum number of tests executed in parallel  [number] [default: 10]
  --maxTimeout              Maximum test execution duration, in milliseconds  [number] [default: 1min]
  --maxBodySize             Maximum HTTP body size for download, in bytes  [number] [default: 50Mb]
  --maxBodySizeIfProcessed  Maximum HTTP body size for assertion, in bytes  [number] [default: 5Mb]
  --regexTimeout            Maximum duration for regex execution, in milliseconds  [number] [default: 0.5sec]

```

These arguments can be passed to your pod by adding an element in the `args` array of the `private-worker-pod.yaml` :

```
(...)
spec:
  containers:
  - name: datadog-private-location-worker
    image: datadog/synthetics-private-location-worker
    args: ["-f=json", "-v"]  #sets the lowest verbosity level
(...)
```

# Notes
This a very simple pod definition, and there are a number of improvements that could be added to this repo.
