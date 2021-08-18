# workload-identity-is-up
## What
Tiny Docker image to run as initContainer to avoid race conditions with a slow-to-start Metadata server.

## Why
[Workload Identity](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) is a GKE feature that grants Kubernetes workloads (pods) ability to assume unique Google Service Account identity without relying on JSON Keys or node's identity. However, it has a [well documented](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) flaw:

> The GKE metadata server needs a few seconds before it can start accepting requests on a newly created Pod. Therefore, attempts to authenticate using Workload Identity within the first few seconds of a Pod's life might fail. Retrying the call will resolve the problem. See the [Troubleshooting section](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity#troubleshoot-timeout) for more details.

When this happens, your applications are likely to fail with `Could not load the default credentials.` message.

Troubleshooting section recommends adding an `initContainer` to prevent main application container(s) from starting before metadata server is up and running. Recommended snippet looks like this:
```yaml
  initContainers:
  - image: gcr.io/google.com/cloudsdktool/cloud-sdk:326.0.0-alpine
    name: workload-identity-initcontainer
    command:
    - '/bin/bash'
    - '-c'
    - |
      curl -s -H 'Metadata-Flavor: Google' 'http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token' --retry 30 --retry-connrefused --retry-max-time 30 > /dev/null || exit 1
```
There are two problems with this approach:
1. Suggested `gcr.io/google.com/cloudsdktool/cloud-sdk:326.0.0-alpine` image is 6 months old (as of 18th of August)
2. It is also **630MB** in size

## Solution
Use `alpinecurl` based `loveholidays/workload-identity-is-up` instead which is **8.3MB** in size (75 times smaller!) and 5 lines of YAML shorter.
```yaml
initContainers:
  - image: loveholidays/workload-identity-is-up
    name: workload-identity-is-up
```

## Warning
Our image is hosted on DockerHub for demonstration purposes only. We do not recommend using Docker Hub for production workloads due to the download rate limits that can impact availability of your production applications. 