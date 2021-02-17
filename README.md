# Demo: Secrets

GitOps is a way of managing all your configurations through Git. It allows teams to version and manage environment configuration and infrastructure through declarative code.

While Kubernetes allows teams to manage their container workloads using resource manifests, storing Kubernetes Secrets in a Git repository has always been a challenge.

Kubernetes Secrets are resources that help you store sensitive information, such as passwords, keys, certificates, OAuth tokens, and SSH keys.

It decouples secret management and utilisation. While admins can create secrets, developers can simply refer to the Secret resource in their manifests instead of hardcoding the password within their pod definition.

That all sounds good, but the problem with Kubernetes secrets is they store sensitive information as a base64 string. Anyone can decode the base64 string to get the original token from the Secret manifest.

Thus, the Secret manifest file can’t be stored in a source-code repository. There’s always a manual element of creating secrets on the cluster, which makes 100% GitOps difficult.

Bitnami Labs has tried to solve this problem by creating an open-source tool called Sealed Secrets.

## Sealed Secrets

Sealed Secrets is comprised of two components:

* A cluster-side Kubernetes controller/operator
* A client-side utility called `kubeseal`

The `kubeseal` utility allows you to seal Kubernetes `Secrets` using the asymmetric crypto algorithm. The `SealedSecrets` are Kubernetes resources that contain encrypted `Secrets` that only the controller can decrypt. Therefore, the `SealedSecret` is safe to store even in a public repository.

`SealedSecret` resources are just a recipe for creating Kubernetes `Secret` resources. When you apply it on the Kubernetes cluster, the cluster-side operator reads the `SealedSecret`, generates the Kubernetes `Secret`, and applies the generated `Secret` on the cluster. The Kubernetes Pod can then use the `Secret` conventionally.

From the end-user perspective, a SealedSecret is a write-only device.

No one apart from the running controller can decrypt the SealedSecret, not even the author of the Secret.

## Install Sealed Secrets into your cluster

~~~bash
# Deploy on your cluster if not already there
$ kubectl apply \
    -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.12.4/controller.yaml \
    -n kube-system
~~~

You might want to backup the generated private key in case of disaster (when the cluster is destroyed for example):

~~~bash
# Download private key - store in OneLogin Notes or Vault
$ kubectl get secret -n kube-system \
    -l sealedsecrets.bitnami.com/sealed-secrets-key \
    -o yaml > sealing-key.yaml && \
    chmod 600 sealing-key.yaml
~~~

Let's download the certificate in order to generate sealed secrets for our cluster using `kubectl`:

~~~bash
# Fetch the certificate that you will use to encrypt your secrets
$ kubeseal --fetch-cert > mycert.pem
~~~

The certificate you can commit to your GitOps repo.

> Keep in mind by default the private key is rotated every 30 days, you need to export the certificate and re-commit to your repo also if you want to be able to generate new sealed secrets!

## Install Kubeseal

~~~bash
# For OSX...
$ brew install kubeseal
~~~

## Create Secret

Let's generate a common secret (which will be base64 encoded) first:

~~~bash
# Set your username of Git (to prevent resources 
# being overwritten in our shared lab environment)
export username=... ;# your Github username

# Create a secret
$ echo -n bar | kubectl create secret generic ${username}-mysecret \
    --dry-run=client \
    --from-file=foo=/dev/stdin \
    -o json > mysecret.json
~~~

It is really easy to "decrypt" this secret using a base64 decoder:

~~~bash
# Let's fetch the value for "foo" and decode
$ cat mysecret.json | jq -r .data.foo | base64 --decode
bar%
~~~
## Seal the secret

That was not impressive considering "encryption", right?

Let's do this better:

~~~bash
# Convert the non-secure secret into something harder to crack ;-)
$ kubeseal --cert mycert.pem < mysecret.json > mysealedsecret.json

# We can safely remove the unsafe secret:
$ rm mysecret.json
~~~

## Use the secret

If you do not specify any namespace, the default namespace (used within the secret encryption process) is `default`.

### Push manually

Let's create a secret in our `default` namespace using the sealed secret:

~~~bash
# Apply the generated json
$ kubectl apply -f mysealedsecret.json
sealedsecret.bitnami.com/username-mysecret created

# Let's inspect the secret
$ kubectl -n default get secrets -o yaml
~~~

... the value is (not surprising) back to `YmFy` - ready to be used by any application:

~~~yaml
...
- apiVersion: v1
  data:
    foo: YmFy
  kind: Secret
...
~~~

### Push using GitOps via ArgoCD

Step one is always to add the secret to your repo:

~~~bash
# Add, commit & push:
$ git add mysealedsecret.json
$ git commit -am 'add secret'
$ git push
~~~

Next, we are going to create an app (part of this repo) using the CLI:

~~~bash
# Create the app
$ argocd app create ${username}-secrets \
    --repo https://github.com/${username}/gitops-workshop-secrets.git \
    --path . \
    --revision HEAD \
    --dest-server https://kubernetes.default.svc \
    --dest-namespace default

# Sync so our changes are deployed
$ argocd app sync ${username}-secrets
~~~

Open the UI to observe the created secret, or use the CLI:

~~~bash
$ kubectl -n default get secrets -o yaml
~~~

You can decode the value of `foo` again using the code snippet shared earlier.

#### Clean-Up

Once we are done playing... let's cleanup the ArgoCD repository

~~~bash
$ argocd app delete ${username}-secrets
$ argocd app delete sealed-secrets
~~~

## Removing a secret

When you create a sealed secret, the secret is stored in 2 different places - hence to cleanup, run:

~~~bash
# Delete the sealed secret in the cluster
$ kubectl delete SealedSecret mysecret

# Delete the secret itself
$ kubectl delete secret mysecret
~~~
