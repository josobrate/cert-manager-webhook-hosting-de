# ACME webhook for hosting.de DNS API

This solver can be used when you want to use cert-manager with hosting.de DNS API. API documentation is [here](https://www.hosting.de/api/?json#dns)

**Note** This is almost direct copy of vadikim's [cert-manager-webhook-hetzner](https://github.com/vadimkim/cert-manager-webhook-hetzner) with heavy adjustments to accomodate using hosting.de's Dynamic DNS interface.

This is not considered ready for production, nor has it been tested. I do not know enough Go to make heads or tails about the code but I know enough to be dangerous.

This version is provided as-is, without any guarantees that it won't wreck your coffee maker or kubernetes installation. **Use at your own risk!**

## Requirements

- [go](https://golang.org/) >= 1.17.0
- [helm](https://helm.sh/) >= v3.0.0
- [kubernetes](https://kubernetes.io/) >= v1.21.1
- [cert-manager](https://cert-manager.io/) >= 1.7.0

## Installation

### cert-manager

Follow the [instructions](https://cert-manager.io/docs/installation/) using the cert-manager documentation to install it within your cluster.

### Webhook

#### Using public helm chart

```bash
helm repo add cert-manager-webhook-hosting-de https://josobrate.github.io/cert-manager-webhook-hosting-de
# Replace the groupName value with your desired domain
helm install --namespace cert-manager cert-manager-webhook-hosting-de cert-manager-webhook-hosting-de/cert-manager-webhook-hosting-de --set groupName=acme.yourdomain.tld
```

#### From local checkout

```bash
helm install --namespace cert-manager cert-manager-webhook-hosting-de deploy/cert-manager-webhook-hosting-de
```

**Note**: The kubernetes resources used to install the Webhook should be deployed within the same namespace as the cert-manager.

To uninstall the webhook run

```bash
helm uninstall --namespace cert-manager cert-manager-webhook-hosting-de
```

## Issuer

Create a `ClusterIssuer` or `Issuer` resource as following:
(Keep in Mind that the Example uses the Staging URL from Let's Encrypt. Look at [Getting Start](https://letsencrypt.org/getting-started/) for using the normal Let's Encrypt URL.)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: mail@example.com # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging

    solvers:
      - dns01:
          webhook:
            # This group needs to be configured when installing the helm package, otherwise the webhook won't have permission to create an ACME challenge for this API group.
            groupName: acme.yourdomain.tld
            solverName: hosting-de
            config:
              secretName: hosting-de-secret
              zoneName: example.com # (Optional): When not provided the Zone will searched in hosting.de API by recursion on full domain name
              apiUrl: https://secure.hosting.de/api/dns/v1/json/recordsUpdate
```

### Credentials

In order to access the hosting.de API, the webhook needs an API token.

If you choose another name for the secret than `hosting-de-secret`, you must install the chart with a modified `secretName` value. Policies ensure that no other secrets can be read by the webhook. Also modify the value of `secretName` in the `[Cluster]Issuer`.

The secret for the example above will look like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: hosting-de-secret
  namespace: cert-manager
type: Opaque
data:
  api-key: your-key-base64-encoded
```

### Create a certificate

Finally you can create certificates, for example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-cert
  namespace: cert-manager
spec:
  commonName: example.com
  dnsNames:
    - example.com
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  secretName: example-cert
```

## Development

### Running the test suite

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

**It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

First, you need to have hosting.de account with access to DNS control panel. You need to create API token and have a registered and verified DNS zone there.
Then you need to replace `zoneName` parameter at `testdata/hosting-de/config.json` file with actual one.
You also must encode your api token into base64 and put the hash into `testdata/hosting-de/hosting-de-secret.yml` file.

You can then run the test suite with:

```bash
# first install necessary binaries (only required once)
./scripts/fetch-test-binaries.sh
# then run the tests
TEST_ZONE_NAME=example.com. make verify
```

## Creating new package

To build new Docker image for multiple architectures and push it to hub:

```shell
docker buildx build --platform linux/amd64,linux/arm64,linux/arm/v7 -t josobrate/cert-manager-webhook-hosting-de:1.2.0 . --push
```

To compile and publish new Helm chart version:

```shell
helm package deploy/cert-manager-webhook-hosting-de
git checkout gh-pages
helm repo index . --url https://josobrate.github.io/cert-manager-webhook-hosting-de/
```
