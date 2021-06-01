# SealedSecrets

Problem: "I can manage all my K8s config in git, except Secrets."

Solution: Encrypt your Secret into a SealedSecret, which is safe to store - even to a public repository. The SealedSecret can be decrypted only by the controller running in the target cluster and nobody else (not even the original author) is able to obtain the original Secret from the SealedSecret.

# Overview

Sealed Secrets is composed of two parts:

A cluster-side controller / operator
A client-side utility: kubeseal
The kubeseal utility uses asymmetric crypto to encrypt secrets that only the controller can decrypt.

These encrypted secrets are encoded in a SealedSecret resource, which you can see as a recipe for creating a secret. Here is how it looks:

```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: mynamespace
spec:
  encryptedData:
    foo: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq.....
```

Once unsealed, it'll look like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
data:
  foo: bar  # <- base64 encoded "bar"
```

This normal kubernetes secret will appear in the cluster after a few seconds and you can use it as you would use any secret that you would have created directly (e.g. reference it from a Pod).

The previous example only focused on the encrypted secret items themselves, but the relationship between a SealedSecret custom resource and the Secret it unseals into is similar in many ways (but not in all of them) to the familiar Deployment vs Pod.

In particular, the annotations and labels of a SealedSecret resource are not the same as the annotations of the Secret that gets generated out of it.

To capture this distinction, the SealedSecret object has a template section which encodes all the fields you want the controller to put in the unsealed Secret.

This includes metadata such as labels or annotations, but also things like the type of the secret.

```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: mynamespace
  annotation:
    "kubectl.kubernetes.io/last-applied-configuration": ....
spec:
  encryptedData:
    .dockerconfigjson: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq.....
  template:
    type: kubernetes.io/dockerconfigjson
    # this is an example of labels and annotations that will be added to the output secret
    metadata:
      labels:
        "jenkins.io/credentials-type": usernamePassword
      annotations:
        "jenkins.io/credentials-description": credentials from Kubernetes
```

The controller would unseal something that looks like this:

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
  labels:
    "jenkins.io/credentials-type": usernamePassword
  annotations:
    "jenkins.io/credentials-description": credentials from Kubernetes
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: mysecret
    uid: 5caff6a0-c9ac-11e9-881e-42010aac003e
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ewogICJjcmVk...
```

As you can see, the generated Secret resource is a "dependent object" of the SealedSecret and as such it will be updated and deleted whenever the SealedSecret object gets updated or deleted.

# Public Key/Certificate

The key certificate (public key portion) is used for sealing secrets, and needs to be available wherever kubeseal is going to be used. The certificate is not secret information, although you need to ensure you are using the correct one.

kubeseal will fetch the certificate from the controller at runtime (requires secure access to the Kubernetes API server), which is convenient for interactive use, but it's known to be brittle when users have clusters with special configurations such as private GKE clusters that have firewalls between master and nodes.

An alternative workflow is to store the certificate somewhere (e.g. local disk) with kubeseal --fetch-cert >mycert.pem, and use it offline with kubeseal --cert mycert.pem. The certificate is also printed to the controller log on startup.

Since v0.9.x certificates get automatically renewed every 30 days. It's good practice that you and your team update your offline certificate periodically. To help you with that, since v0.9.2 kubeseal accepts URLs too. You can setup your internal automation to publish certificates somewhere you trust.

`kubeseal --cert https://your.intranet.company.com/sealed-secrets/your-cluster.cert`