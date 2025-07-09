# Helm chart for Itential Automation Platform

This repo contains helm charts for running Itential Automation Platform in Kubernetes.

## Itential Automation Platform (IAP)

The chart will not install the Redis and MongoDB dependencies of the IAP application. The chart
assumes that those are running, configured, and bootstrapped with all necessary data. The
application is installed using a Kubernetes statefulset. It also includes persistent volume claims,
ingress, and other Kubernetes objects suitable to run the application.

This chart is optimized for Itential version P6 and beyond and will not work with older Itential
versions.

### Usage

This will install IAP according to how its configured in the values.yaml file ("latest").

```bash
helm install iap . -f values.yaml
```

This will install IAP with the "6.0.4" image.

```bash
helm install iap . -f values.yaml --set image.tag=6.0.4
```

### Requirements & Dependencies

#### Secrets

The chart assumes the following secrets, they are not included in the Chart.

##### imagePullSecrets

This is the secret that will pull the image from the Itential ECR. Name to be determined by the user
 of the chart and that name must be provided in the values file (`imagePullSecrets[0].name`).

##### itential-platform-secrets

This secret contains several sensitive values that the application may use. They are loaded into the
pod as environment variables. Some are optional and depend on your implementation.

| Secret Key | Description | Required? |
|:-----------|:------------|:----------|
| ITENTIAL_DEFAULT_USER_PASSWORD | The password for the default local user, normally only used during installation and initial configuration. | Yes |
| ITENTIAL_ENCRYPTION_KEY | Used by the application for native encryption of sensitive values. 64-length hex string describing a 256 bit encryption key. | Yes |
| ITENTIAL_MONGO_PASSWORD | The MongoDB password for the `itential` user. | Yes |
| ITENTIAL_MONGO_URL | The MongoDB connection string, which can sometimes contain passwords. | Yes |
| ITENTIAL_REDIS_PASSWORD | The Redis password for the `itential` user. | Yes |
| ITENTIAL_REDIS_SENTINEL_PASSWORD | The Redis Sentinel password for the `itential` user. Only required if connecting to a Redis replica set that also uses Redis Sentinel. | No |
| ITENTIAL_VAULT_SECRET_ID | The Hashicorp Vault Secret ID when using Hashicorp Vault as the secrets manager. Only required if using Hashicorp Vault for secrets. | No |
| ITENTIAL_WEBSERVER_HTTPS_PASSPHRASE | The passphrase used in conjunction with an HTTPS certificate. | No |

##### iap-ca

This secret represents the CA used by cert-manager to derive all the TLS certificates. Name to be
provided by the user of the chart in the values file (`issuer.caSecretName`) if using cert-manager.

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

For more information see the [Cert Manager project](https://cert-manager.io/).

#### DNS

This is an optional requirement.

Itential used the ExternalDNS project to facilitate the creation of DNS entries. ExternalDNS
synchronizes exposed Kubernetes Services and Ingresses with DNS providers. This is not a
requirement for running the chart. The chart and the application can run without this if needed. We
chose to use this to provide us with external addresses.

For more information see the [ExternalDNS project](https://github.com/kubernetes-sigs/external-dns).

### Volumes

| Name | Type | Description |
|:-----|:-----|:------------|
| iap-logs-volume | Persistent Volume Claim | A persistent volume claim to mount a directory to write IAP log files to |
| iap-assest-volume | Persistent Volume Claim | A persistent volume claim to mount a directory that includes adapters and apps |

### How to construct the iap-asset-volume

This volume is intended to store the applications and adapters unique to a customer. Its contents
will reflect a customer's unique usage of IAP and contain all of the adapters and custom
applications required. There is an expectation in the container of the structure of the files in
this volume. All adapters and applications can be added into the same parent directory that will
then be mounted in the container.

```bash
.
├── adapter1/
├── adapter2/
├── custom-application1/
└── custom-application2/
```

This will be correctly translated inside the container to the appropriate directories for IAP to
understand.
