# Tekton Chains
Supply Chain Security in Tekton Pipelines

## Getting Started

Tekton Chains is currently experimental, and does not have any published releases.
To use Tekton Chains, you'll have to build and deploy your own release from git.

### Installation

See [DEVELOPMENT.md](DEVELOPMENT.md) for a guide on how to build and deploy your own version.

### Configuration

Chains uses a `ConfigMap` called `chains-config` in the `tekton-pipelines` namespace for configuration.
Supported keys include:

| Key | Description | Supported Values |
| --- | --- | --- |
| `artifacts.taskrun.format` | The format to store `TaskRun` payloads in. | `tekton` |
| `artifacts.taskrun.storage` | The storage backend to store `TaskRun` signatures in. | `tekton`, `oci`, `gcs` |
| `artifacts.taskrun.signer` | The signature backend to sign `Taskrun` payloads with. | `pgp`, `x509`, `kms` |
| `artifacts.oci.format` | The format to store `OCI` payloads in. | `tekton`, `simplesigning` |
| `artifacts.oci.storage` | The storage backend to store `OCI` signatures in. | `tekton`, `oci`, `gcs` |
| `artifacts.oci.signer` | The signature backend to sign `OCI` payloads with. | `pgp`, `x509`, `kms` |
| `signers.kms.kmsref` | The URI reference to a KMS service to use in `KMS` signers. | `gcpkms://projects/<project>/locations/<location>/keyRings/<keyring>/cryptoKeys/<key>`|

### Overview

`Chains` works by observing `TaskRun` executions, capturing relevant information, and storing it in a cryptographically-signed format.

`TaskRuns` can indicate inputs and outputs which are then captured and surfaced in the `Chains` payload formats, where relevant.
`Chains` uses the standard mechanisms (`Results` and `PipelineResouces`) where possible, and provides a few other mechanisms to *hint* at the correct inputs and outputs. These are outlined below:

#### Chains Type Hinting

When outputing an OCI image without using a `PipelineResource`, `Chains` will look for the following Results:

* `IMAGE_URL` - The URL to the built OCI image
* `IMAGE_DIGEST` - The Digest of the built OCI image

Note that these are provided automatically when using `PipelineResources`.

### Signing Secrets

To get started signing things with Chains, you first have to instruct Chains on how to sign things.
Chains supports a few different signature schemes, including PGP/GPG, x509 and KMS systems.
The private key material (or access to it) is required by Chains in order to create signatures.

This section explains how to configure this for each type.


#### GPG/PGP

Chains expects the private key and passphrase to be in a secret called `signing-secrets` with the following structure:

* pgp.private-key (the private key)
* pgp.passphrase (the optional passphrase)

You can set all of these as fields in the Kubernetes secret `signing-secrets`:

```shell
kubectl create secret generic signing-secrets -n tekton-pipelines --from-file=pgp.passphrase --from-file=pgp.private-key --from-file=pgp.public-key
```

##### Setup
You'll need to create or upload an existing private key as Kubernetes secret.
If you don't have one already, but you can usually use something like this:

```shell
gpg --gen-key
```

Enter a passprase (make sure you remember it!) and a name for the key.

Next, you'll need to upload the private key as a Kubernetes `Secret` so Tekton can use it
to sign payloads.
To do that, export the public and private keys as files:

```shell
gpg --export-secret-key --armor $keyname > pgp.private-key
gpg --export --armor $keyname > pgp.public-key
```

And save the passphrase (if you set one) in it's own file:

```shell
echo -n $passphrase > pgp.passphrase
```


#### x509

Chains expects the private key to be stored as an unencrpyted PKCS8 PEM file (`BEGIN PRIVATE KEY`).
Chains only supports `ed25519` and `ecdsa` keys today.

The private key is expected to be stored in a secret named `signing-secrets` under the key `x509.pem`.

##### Setup

To create an ecdsa keypair, you can do something like:

```shell
openssl ecparam -genkey -name prime256v1 > ec_private.pem
openssl pkcs8 -topk8 -in ec_private.pem  -nocrypt
```

You can add this to the secret with something like:

```shell
kubectl create secret generic signing-secrets -n tekton-pipelines --from-file=x509.pem
```

#### KMS

Chains uses a "go-cloud" URI like scheme to refer to KMS references.
Chains supports GCP KMS and Hashicorp Vault today, but we would love to add support for more.

You can configure Chains to use a specific KMS key using the `signers.kms.kmsref` config key in `chains-config`.

For GCP, this should have the structure of `gcpkms://projects/<project>/locations/<location>/keyRings/<keyring>/cryptoKeys/<key>`
where <location>, <keyring>, and <key> are filled in appropriately.

For Vault, this should have the structure of `hashivault://<keyname>`, where the `keyname` is filled out appropriately.

The `chains-controller` deployment must have access to this somehow.
For GCP/GKE, we suggest using Workload Identity.
Other Service Account techniques would work as well.

---
**NOTE**

If your signing secrets is already populated, you may get the following error:

```shell
Error from server (AlreadyExists): secrets "signing-secrets" already exists
```

Simply prepend a delete:

```shell
kubectl delete secret signing-secrets -n tekton-pipelines && kubectl create secret generic signing-secrets -n tekton-pipelines --from-file=pgp.passphrase --from-file=pgp.private-key --from-file=pgp.public-key
```
---

## Usage

In its default mode of operation, Chains works by observing all `TaskRuns` in your cluster.
When `TaskRuns` complete, Chains takes a snapshot of them.
Chains then converts this snapshot to one or more standard payload formats, signs them and stores them somewhere.

To try it out, create a `TaskRun` of your choice.
For a sample one, try this:

```shell
$ kubectl create -f examples/task-output-image.yaml
taskrun.tekton.dev/build-push-run-output-image-abcd created
```

Then, take the name of the `TaskRun` you just created, and wait for it to finish (SUCCEEEDED should be True).

```shell
$ kubectl get taskrun.tekton.dev/taskrun home-is-set-rwhzs
NAME                SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
home-is-set-rwhzs   True        Succeeded   105s        100s
```

Next, retrieve the signature and payload from the object (they are stored as base64-encoded annotations):

```shell
$ kubectl get taskrun home-is-set-rwhzs -o=json | jq  -r '.metadata.annotations["chains.tekton.dev/payload-taskrun"]' | base64 --decode > payload
$ kubectl get taskrun home-is-set-rwhzs -o=json | jq  -r '.metadata.annotations["chains.tekton.dev/signature-taskrun"]' | base64 --decode > signature
```

Finally, we can check the signature:

```shell
 gpg --verify signature payload
gpg: Signature made Mon Jul 27 14:42:01 2020 CDT
gpg:                using RSA key 4FCFD41D993B806C
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   4  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 4u
gpg: next trustdb check due at 2022-03-16
gpg: Good signature from "Tekton Unit Tests <testing@tekton.dev>" [ultimate]
```

Now we have a verifiable record of the `TaskRuns`!


## Want to contribute

We are so excited to have you!

See [CONTRIBUTING.md](CONTRIBUTING.md) for an overview of our processes
See [DEVELOPMENT.md](DEVELOPMENT.md) for how to get started
See [ROADMAP.md](ROADMAP.md) for the current roadmap
Look at our good first issues and our help wanted issues

