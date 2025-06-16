# Helm chart for Itential Automation Platform

This repo contains helm charts for running Itential Automation Platform in Kubernetes.

## Itential Automation Platform (IAP)

The chart will not install any dependencies of the IAP application. The chart assumes that those are
running, configured, and bootstrapped with all necessary data. The application is installed using a
Kubernetes statefulset. It also includes persistent volume claims, ingress, and other Kubernetes
objects suitable to run the application.

### Usage

```bash
helm install iap . -f values.yaml
```

This will install IAP according to how its configured in the values.yaml file ("latest").

```bash
helm install iap . -f values.yaml --set image.tag=2023.2.7
```

This will install IAP with the "2023.2.7" image.

```bash
helm install iap ./iap --sete propertiesJson.dbUrl="mongodb+srv://<some-username>:<some-password>@<some-mongo-url>"
```

This will install IAP using the mongo connection string provided.

### Requirements & Dependencies

#### Certificates

The chart will require a Certificate Authority to be added to the Kubernetes environment. This is
used by the chart when running with `useTLS` flag enabled. The chart will use this CA to generate
the necessary certificates using a Kubernetes `Issuer` which is included. The Issuer will issue the
certificates using the CA. The certificates are then included using a Kubernetes `Secret` which is
mounted by the pods. Creating and adding this CA is outside of the scope of this chart.

Both the `Issuer` and the `Certificate` objects are realized by using the widely used Kubernetes
add-on called `cert-manager`. Cert-manager is responsible for making the TLS certificates required
by using the CA that was installed separately. The installation of cert-manager is outside the scope
of this chart. To check if this is already installed run this command:

```bash
kubectl get crds | grep cert-manager
```

#### DNS

This is not a requirement but more of an explanation.

Itential used the ExternalDNS project to facilitate the creation of DNS entries. ExternalDNS
synchronizes exposed Kubernetes Services and Ingresses with DNS providers. This is not a
requirement for running the chart. The chart and the application can run without this if needed. We
chose to use this to provide us with external addresses.

For more information see the [ExternalDNS project](https://github.com/kubernetes-sigs/external-dns).

### Volumes

| Name | Type | Description |
|:-----|:-----|:------------|
| config-volume     | Configmap               | Configuration properties for the IAP properties.json file. This is the main config file for the IAP application. |
| iap-logs-volume   | Persistent Volume Claim | A persistent volume claim to mount a directory to write IAP log files to                                               |
| iap-assest-volume | Persistent Volume Claim | A persistent volume claim to mount a directory that includes adapters and apps                                         |

### How to construct the iap-asset-volume

This volume is intended to store the applications and adapters. Its contents will reflect a customer's unique usage of IAP and contain all of the adapters and custom applications required. There is an expectation in the container of the structure of the files in this volume. It should look like this:

```bash
.
└── node_modules/
    ├── @itentialopensource/
    │   ├── opensource-adapter1
    │   └── opensource-adapter2
    ├── @itential/
    │   └── app-artifacts
    └── @customer-namespace/
        ├── custom-adapter1
        └── custom-adapter2
```

This will be correctly translated inside the container to the appropriate directories for IAP to understand.
