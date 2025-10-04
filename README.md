[![Docker Image CI](https://github.com/fc4soda/cert-manager-webhook-namesilo/actions/workflows/docker-image.yml/badge.svg)](https://github.com/fc4soda/cert-manager-webhook-namesilo/actions/workflows/docker-image.yml)

# Kubernetes Cert Manager Webhook for Namesilo.com

Cert Manager Webhook for Namesilo.com is an [ACME webhook solver](https://cert-manager.io/docs/configuration/acme/dns01/webhook/) for [cert-manager](https://cert-manager.io/) that enables the use of DNS01 challenges with [Namesilo.com](https://namesilo.com/) as the DNS provider, via the [Namesilo.com API](https://www.namesilo.com/api-reference).

## :toolbox: Requirements

- [go](https://golang.org/) `>= 1.13.0`
- [helm](https://helm.sh/) `>= v3.0.0` [installed](https://helm.sh/docs/intro/install/) on your computer
- [kubernetes](https://kubernetes.io/) `>= v1.14.0` (`>=v1.19` recommended)
- [cert-manager](https://cert-manager.io/) `>= 0.12.0` [installed](https://cert-manager.io/docs/installation/) on the cluster
- A Namesilo.com account with a [Namesilo.com API token](https://www.namesilo.com/account/api-manager)
- A valid domain registered and [configured with Namesilo.com's default nameservers](https://www.namesilo.com/api-reference)

## :package: Installation

### 1. Webhook

Use a local checkout of this repository and install the webhook with Helm:

```shell{:copy}
helm install --namespace cert-manager cert-manager-webhook-namesilo  ./deploy/cert-manager-webhook-namesilo/
```

> :bell: **Note:** The webhook should be deployed into the same namespace as `cert-manager`. If you changed that, you should update the `certManager.namespace` value in the deploy template file, [`values.yaml`](deploy/cert-manager-webhook-namesilo/values.yaml), before installation.
#### Uninstallation

You can also remove the webhook using Helm:

```shell{:copy}
helm uninstall --namespace cert-manager cert-manager-webhook-namesilo
```

### 2. API token secret

Create a [Kubernetes Secret](https://kubernetes.io/docs/concepts/configuration/secret/) to store the value of your Namesilo.com API token:

```shell{:copy}
kubectl create secret generic namesilo-credentials --from-literal=api-token=<your API token> --namespace cert-manager 
```

> :bulb: **Note:** The secret should also be in the same namespace as `cert-manager`. If you change the name of the secret or key, don't forget to use those values in the Issuer below.
### 3. Certificate issuer

Define a [cert-manager Issuer (or ClusterIssuer)](https://cert-manager.io/docs/concepts/issuer/) resource that uses the webhook as the solver. Create a file called, e.g. `cert-issuer.yml`, and use the following content as the template:

###### `cert-issuer.yml`
```yaml{:copy}
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-namesilo
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # For production use this URL instead:
    # server: https://acme-v02.api.letsencrypt.org/directory
    email: <you@your-email-address.com>
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
    - dns01:
        webhook:
          groupName: acme.namesilo.com
          solverName: namesilo
          config:
            username: <your Namesilo.com username>
            apitokensecret:
              name: namesilo-credentials
              key: api-token
```

> :bulb: **Note:** The `config` key for the webhook defines your Namesilo.com API credentials—the `apitokensecret.name` and `apitokensecret.key` values must match those for your secret, above.
Apply the file to your cluster to create the resource:

```shell{:copy}
kubectl apply -f cert-issuer.yaml
```

> :bell:**Note:** If you defined an `Issuer` rather than a `ClusterIssuer`, you should create it in the same namespace as `cert-manager`.
## :scroll: Issue a certificate

Create a certificate by defining a [cert-manager Certificate](https://cert-manager.io/docs/concepts/certificate/) resource and applying it to your cluster:

###### `example-cert.yml`
```yaml{:copy}
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
spec:
  dnsNames:
    - test.example.com
  issuerRef:
    name: letsencrypt-namesilo
    kind: ClusterIssuer
  secretName: example-cert
```

> :bulb: **Note:** If you defined an `Issuer` rather than a `ClusterIssuer`, you can omit the `issuerRef.kind` key.
```shell{:copy}
kubectl apply -f example-cert.yml
```

After allowing a short period for the challenge, order and issuing process to complete, the certificate should be available for use: :partying_face:

```shell
$ kubectl get certificate example-com
NAME          READY   SECRET             AGE
example-com   True    example-com-cert   1m12s
```

## :wrench: Development

### Running the test suite

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

> :heavy_check_mark: **It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

Before running the test suite, you must supply valid credentials for the Namesilo.com API. See the [test data README](testdata/namesilo/README.md) for more information.

You can run the test suite with:

```bash
TEST_ZONE_NAME=example.com. make test
```

> :bell: **Note:** `example.com` must also be a domain registered to your Namesilo.com and [configured with Namesilo.com's default nameservers](https://www.namesilo.com/api-reference) so that DNS records can be [managed via Namesilo.com DNS](https://www.namesilo.com/api-reference).