# Trusted Resources

- [Overview](#overview)
- [Instructions](#Instructions)
 - [Sign Resources](#sign-resources)
 - [Enable Trusted Resources](#enable-trusted-resources)

## Overview

Trusted Resources is a feature which can be used to sign Tekton Resources and verify them. Details of design can be found at [TEP--0091](https://github.com/tektoncd/community/blob/main/teps/0091-trusted-resources.md). This feature is under `alpha` version and support `v1beta1` version of `Task` and `Pipeline`.

Verification failure will mark corresponding taskrun/pipelinerun as Failed status and stop the execution.

**Note:** KMS is not currently supported and will be supported in the following work.


## Instructions

### Sign Resources
We have added `sign` and `verify` into [Tekton Cli](https://github.com/tektoncd/cli) as a subcommand in release [v0.28.0 and later](https://github.com/tektoncd/cli/releases/tag/v0.28.0). Please refer to [cli docs](https://github.com/tektoncd/cli/blob/main/docs/cmd/tkn_task_sign.md) to sign and Tekton resources.

A signed task example:
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  annotations:
    tekton.dev/signature: MEYCIQDM8WHQAn/yKJ6psTsa0BMjbI9IdguR+Zi6sPTVynxv6wIhAMy8JSETHP7A2Ncw7MyA7qp9eLsu/1cCKOjRL1mFXIKV
  creationTimestamp: null
  name: example-task
  namespace: tekton-trusted-resources
spec:
  steps:
  - image: ubuntu
    name: echo
```

### Enable Trusted Resources

#### Enable feature flag

Update the config map:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: tekton-pipelines
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-pipelines
data:
  resource-verification-mode: "enforce"
```

**Note:** `resource-verification-mode` needs to be set as `enforce` or `warn` to enable resource verification.

`resource-verification-mode` configurations:
 * `enforce`: Failing verification will mark the taskruns/pipelineruns as failed.
 * `warn`: Log warning but don't fail the taskruns/pipelineruns.
 * `skip`: Directly skip the verification.

Or patch the new values:
```bash
kubectl patch configmap feature-flags -n tekton-pipelines -p='{"data":{"resource-verification-mode":"enforce"}}
```


#### Config key at configmap (will be deprecated)
Multiple keys reference should be separated by comma. If the resource can pass any key in the list, it will pass the verification.

**Note:** key configuration in configmap will be deprecated, the issue [#5852](https://github.com/tektoncd/pipeline/issues/5852) will track the deprecation.

We currently hardcode SHA256 as hashfunc for loading public keys as verifiers.

Public key files should be added into secret and mounted into controller volumes. To add keys into secret you may execute:

```shell
kubectl create secret generic verification-secrets \
  --from-file=cosign.pub=./cosign.pub \
    --from-file=cosign.pub=./cosign2.pub \
  -n tekton-pipelines
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-trusted-resources
  namespace: tekton-pipelines
  labels:
    app.kubernetes.io/instance: default
    app.kubernetes.io/part-of: tekton-pipelines
data:
  publickeys: "/etc/verification-secrets/cosign.pub, /etc/verification-secrets/cosign2.pub"
```

#### Config key at VerificationPolicy
VerificationPolicy supports SecretRef or encoded public key data.

How does VerificationPolicy work?
You can create multiple `VerificationPolicy` and apply them to the cluster.
1. Trusted resources will look up policies from the resource namespace (usually this is the same as taskrun/pipelinerun namespace).
2. If multiple policies are found. For each policy we will check if the resource url is matching any of the `patterns` in the `resources` list. If matched then this policy will be used for verification.
3. If multiple policies are matched, the resource needs to pass all of them to pass verification.
4. To pass one policy, the resource can pass any public keys in the policy.

Take the following `VerificationPolicies` for example, a resource from "https://github.com/tektoncd/catalog.git", needs to pass both `verification-policy-a` and `verification-policy-b`, to pass `verification-policy-a` the resource needs to pass either `key1` or `key2`.

Example:
```yaml
apiVersion: tekton.dev/v1alpha1
kind: VerificationPolicy
metadata:
  name: verification-policy-a
  namespace: resource-namespace
spec:
  # resources defines a list of patterns
  resources:
    - pattern: "https://github.com/tektoncd/catalog.git"  #git resource pattern
    - pattern: "gcr.io/tekton-releases/catalog/upstream/git-clone"  # bundle resource pattern
    - pattern: " https://artifacthub.io/"  # hub resource pattern
  # authorities defines a list of public keys
  authorities:
    - name: key1
      key:
        # secretRef refers to a secret in the cluster, this secret should contain public keys data
        secretRef:
          name: secret-name-a
          namespace: secret-namespace
        hashAlgorithm: sha256
    - name: key2
      key:
        # data stores the inline public key data
        data: "STRING_ENCODED_PUBLIC_KEY"
```

```yaml
apiVersion: tekton.dev/v1alpha1
kind: VerificationPolicy
metadata:
  name: verification-policy-b
  namespace: resource-namespace
spec:
  resources:
    - pattern: "https://github.com/tektoncd/catalog.git"
  authorities:
    - name: key3
      key:
        # data stores the inline public key data
        data: "STRING_ENCODED_PUBLIC_KEY"
```

`namespace` should be the same of corresponding resources' namespace.

`pattern` is used to filter out remote resources by their sources URL. e.g. git resources pattern can be set to https://github.com/tektoncd/catalog.git. The `pattern` should follow regex schema, we use go regex library's [`Match`](https://pkg.go.dev/regexp#Match) to match the pattern from VerificationPolicy to the `ConfigSource` URL resolved by remote resolution. Note that `.*` will match all resources.
To learn more about regex syntax please refer to [syntax](https://pkg.go.dev/regexp/syntax).
To learn more about `ConfigSource` please refer to resolvers doc for more context. e.g. [gitresolver](./git-resolver.md)

 `key` is used to store the public key, note that secretRef and data cannot be configured at the same time.

`hashAlgorithm` is the algorithm for the public key, by default is `sha256`. It also supports `SHA224`, `SHA384`, `SHA512`.

