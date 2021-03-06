# OIDC Webhook Authenticator

## Problem

In Kubernetes you can authenticate via several authentication strategies:

- x509 Client Certificates
- Static Token Files
- Bootstrap Tokens
- Static Password File (Basic authentication - deprecated and removed in 1.19)
- Service Account Tokens
- OpenID Connect TOkens
- Webhook Token Authentication
- Authenticating Proxy

End-users should use [OpenID Connect (OIDC) Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) created by OIDC-compatible Identity Provider (IDP) and present [id_token](https://openid.net/specs/openid-connect-core-1_0.html#IDToken) to the Kube APIServer. If the API server is configured to trust the IDP and the token is valid, then the user is authenticated and the [UserInfo](https://github.com/kubernetes/kubernetes/blob/99019502bd6ed038dbd1c444974d5e8c6a8dda19/staging/src/k8s.io/api/authentication/v1/types.go#L100-L117) is send to the authorization stack.

Ideally, operators of the Gardener cluster should be able to authenticate to end-user Shoot clusters with `id_token` generated by OIDC IDP, but in many cases, end-users might have already configured OIDC for their cluster and more than one OIDC configurations are not allowed.

Another interesting application of multiple OIDC providers would be per `Project` OIDC provider where end-users of Gardener can add their own OIDC-compatible IDPs.

To workaround the one OIDC per Kube APIServer limitation, a new `OIDC Webhook Authenticator` (OWA) could be implemented.

## Goals

- Dynamic registrations of OpenID Connect configurations.
- Close as possible to the Kubernetes build-in OIDC Authenticator.
- Build as an optional extension and not required for functional Shoot or Gardener cluster.

## Non-goals

- [Dynamic Authorization](https://kubernetes.io/docs/reference/access-authn-authz/webhook/) is out of scope.

## Proposal

The Kube APIServer can use [Webhook Token Authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication) to send a [Bearer Tokens (id_token)](https://tools.ietf.org/html/rfc6750#section-2.1) to external webhook for validation:

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "spec": {
    "token": "(BEARERTOKEN)"
  }
}
```

Where upon verification, the remote webhook returns the identity of the user (if authentication succeeds):

```json
{
  "apiVersion": "authentication.k8s.io/v1beta1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "janedoe@example.com",
      "uid": "42",
      "groups": [
        "developers",
        "qa"
      ],
      "extra": {
        "extrafield1": [
          "extravalue1",
          "extravalue2"
        ]
      }
    }
  }
}
```

### Registration of new OpenIDConnect

This new OWA can be configured with multiple OIDC providers and the entire flow can look like this:

1. Admin adds a new `OpenIDConnect` resource (via CRD) to the cluster.

    ```yaml
    apiVersion: authentication.gardener.cloud/v1alpha1
    kind: OpenIDConnect
    metadata:
      name: foo
    spec:
      issuerURL: https://foo.bar
      clientID: some-client-id
      usernameClaim: email
      usernamePrefix: "test-"
      groupsClaim: groups
      groupsPrefix: "baz-"
      supportedSigningAlgs:
      - RS256
      requiredClaims:
        baz: bar
      caBundle: LS0tLS1CRUdJTiBDRVJU...base64-encoded CA certs for issuerURL.
    ```

1. 1. OWA watches for changes on this resource and does [OIDC discovery](https://openid.net/specs/openid-connect-discovery-1_0.html). The [OIDC provider's  configuration](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfigurationResponse) has to be accessible under the `spec.issuerURL` with a [well-known path (.well-known/openid-configuration)](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig).
1. OWA uses the `jwks_uri` obtained from the OIDC providers configuration, to fetch the OIDC provider's public keys from that endpoint and stores them in the status of `OpenIDConnect`:

    ```yaml
    apiVersion: authentication.gardener.cloud/v1alpha1
    kind: OpenIDConnect
    metadata:
      name: foo
    spec:
      issuerURL: https://foo.bar
      ...
    status:
      keys: f31deA9b... #the content of jwks_uri base64-encoded
    ```

1. OWA uses those keys, issuer, client_id and other settings to add OIDC authenticator to a in-memory list of [Token Authenticators](https://pkg.go.dev/k8s.io/apiserver/pkg/authentication/authenticator?tab=doc#Token).

![alt text](assets/registration.svg "Authentication with OIDC webhook")

### End-user authentication via new OpenIDConnect IDP

When a user presents an `id_token` obtained from a OpenID Connect the flow looks like this:

1. The user authenticates in Custom IDP.
1. `id_token` is obtained from Custom IDP.
1. The user uses `id_token` to perform an API call to Kube APIServer.
1. As the `id_token` is not matched by any build-in or configured authenticators in the Kube APIServer, it is send to OWA for validation.

    ```json
    {
      "TokenReview": {
        "kind": "TokenReview",
        "apiVersion": "authentication.k8s.io/v1beta1",
        "spec": {
          "token": "ddeewfwef..."
        }
      }
    }
    ```

1. OWA uses `TokenReview` to authenticate the calling API server (the Kube APIServer for delegation of authentication and authorization is different from the calling API server).

    > Example: When a Shoot cluster's API Server is configured to verify tokens by OWA, that API server will be the callee API server. The Seed API server will be used for delegating authentication and authorization.

    ```json
    {
      "TokenReview": {
        "kind": "TokenReview",
        "apiVersion": "authentication.k8s.io/v1beta1",
        "spec": {
          "token": "api-server-token..."
        }
      }
    }
    ```

1. After the Authentication API server returns the identity of callee API server:

    ```json
    {
        "apiVersion": "authentication.k8s.io/v1",
        "kind": "TokenReview",
        "metadata": {
            "creationTimestamp": null
        },
        "spec": {
            "token": "eyJhbGciOiJSUzI1NiIsImtpZCI6InJocEdLTXZlYjV1OE5heD..."
        },
        "status": {
            "authenticated": true,
            "user": {
                "groups": [
                    "system:serviceaccounts",
                    "system:serviceaccounts:shoot--abcd",
                    "system:authenticated"
                ],
                "uid": "14db103e-88bb-4fb3-8efd-ca9bec91c7bf",
                "username": "system:serviceaccount:shoot--abcd:kube-apiserver"
            }
        }
    }
    ```

    OWA makes a `SubjectAccessReview` call to the Authorization API server to ensure that callee API server is allowed to validate tokens:

    ```json
    {
      "apiVersion": "authorization.k8s.io/v1",
      "kind": "SubjectAccessReview",
      "spec": {
        "groups": [
          "system:serviceaccounts",
          "system:serviceaccounts:shoot--abcd",
          "system:authenticated"
        ],
        "nonResourceAttributes": {
          "path": "/validate-token",
          "verb": "post"
        },
        "user": "system:serviceaccount:shoot--abcd:kube-apiserver"
      },
      "status": {
        "allowed": true,
        "reason": "RBAC: allowed by RoleBinding \"kube-apiserver\" of ClusterRole \"kube-apiserver\" to ServiceAccount \"system:serviceaccount:shoot--abcd:kube-apiserver\""
      }
    }
    ```

1. OWA then iterates over all registered `OpenIDConnect` Token authenticators and tries to validate the token.
1. Upon a successful validation it returns the `TokeReview` with user, groups and extra parameters:

    ```json
    {
      "TokenReview": {
        "kind": "TokenReview",
        "apiVersion": "authentication.k8s.io/v1beta1",
        "spec": {
          "token": "ddeewfwef..."
        },
        "status": {
          "authenticated": true,
          "user": {
            "username": "test-foo@bar.com",
            "groups": [
              "baz-employee"
            ],
            "extra": {
              "gardener.cloud/apiserver/groups": [
                "system:serviceaccounts",
                "system:serviceaccounts:shoot--abcd",
                "system:authenticated"
              ],
              "gardener.cloud/apiserver/uid": [
                "system:serviceaccount:shoot--abcd:kube-apiserver"
              ],
              "gardener.cloud/apiserver/username": [
                "system:serviceaccount:shoot--abcd:kube-apiserver"
              ],
              "gardener.cloud/oidc/name": [
                "foo"
              ],
              "gardener.cloud/oidc/uid": [
                "e5062528-e5a4-4b97-ad83-614d015b0979"
              ],
              "gardener.cloud/oidc/resourceVersion": [
                "3355876311"
              ]
            }
          }
        }
      }
    }
    ```

    It also adds some extra information which can be used by custom authorizers later on:

    1. `gardener.cloud/apiserver/groups` contains all the groups of the API server which is making the `TokenReview` request (it's the ServiceAccount of the API Server Pod in this case)
    1. `gardener.cloud/apiserver/uid` contains the UID of the API server which is making the `TokenReview` request (it's the ServiceAccount of the API Server Pod in this case)
    1. `gardener.cloud/apiserver/username` contains the username of the API server which is making the `TokenReview` request (it's the ServiceAccount of the API Server Pod in this case)
    1. `gardener.cloud/oidc/name` contains the name of the `OpenIDConnect` authenticator which was used.
    1. `gardener.cloud/oidc/uid` contains the `metadata.uid` of the `OpenIDConnect` authenticator which was used.
    1. `gardener.cloud/oidc/resourceVersion` contains the `metadata.resourceVersion` of the `OpenIDConnect` authenticator which was used.

1. Kube APIServer proceeds with authorization checks and returns response.

An overview of the flow:

![alt text](assets/authentication.svg "Authentication with OIDC webhook")

## Deployment for Shoot clusters

To save cost, a single (multi-replica) deployment of OWA can be deployed in the `Seed` cluster. All Shoot API Servers are started with

```text
--authentication-token-webhook-config-file=/etc/webhook/kubeconfig
```

where `/etc/webhook/kubeconfig` would contain a standard `kubeconfig`, with using for authentication the Service Account token of the API Server:

```yaml
apiVersion: v1
kind: Config
clusters:
- name: authenticator
  cluster:
    certificate-authority-data: LS0tLS1CRU...
    server: https://oidc-webhook-authenticator/odic-authenticator-system.svc/validate-token
users:
- name: token
  user:
    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
current-context: webhook
contexts:
- context:
    cluster: authenticator
    user: token
  name: webhook
```

Depending on the version of the Seed cluster and configuration, [Service Account Token Volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-token-volume-projection) should be used instead of static ServiceAccount Tokens:

```yaml
volumes:
- name: oidc-authenticator-token
  projected:
    sources:
    - serviceAccountToken:
        path: oidc-authenticator-token
        expirationSeconds: 7200
        audience: oidc-authenticator
```

OWA is deployed via `ControllerRegistration`, which deploys the necessary components and inject the necessary shoot kube-apiserver configuration via a `MutatingWebhookConfiguration`.
