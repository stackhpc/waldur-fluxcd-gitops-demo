# Waldur FluxCD Demo

This repository demonstrates how to deploy Waldur on top of a GitOps-managed Kubernetes cluster running on OpenStack.

The tooling is based on the [capi-helm-fluxcd-config](https://github.com/stackhpc/capi-helm-fluxcd-config)
repository as a template for GitOps-based Kubernetes clusters. See the base repo's README for a general
introduction.

This downstream repo adds some additional configuration for deploying an instance of Waldur on the provisioned
cluster, which includes the required FluxCD Helm configuration for installing the
[Waldur Helm Chart](https://artifacthub.io/packages/helm/waldur-charts/waldur) as well as some example Helm values
for customising the Waldur installation (see `components/waldur`).

To view all of the Waldur-specific additions made to the base capi-helm-fluxcd-config repo, you can check out this
downstream repo locally and run the following:

```
git clone https://github.com/stackhpc/waldur-fluxcd-gitops-demo
git remote add upstream https://github.com/stackhpc/capi-helm-fluxcd-config
git fetch --all
git diff upstream/main
```

## Replicating this deployment example

To deploy your own Waldur instance based on this example you should make a copy (i.e. a detached fork) of this
repository:

```
# Clone the repository
git clone https://github.com/stackhpc/waldur-fluxcd-gitops-demo.git my-waldur-deployment
cd my-waldur-deployment

# Rename the origin remote to upstream so that we can pull changes in future
git remote rename origin upstream

# Add the new origin remote and push the initial commit
git remote add origin <url>
git push -u origin main
```

You should then replace the `clusters/waldur/credentials.yaml` file with your own OpenStack application
credential for your target cloud (see [here](https://github.com/stackhpc/capi-helm-fluxcd-config/tree/main?tab=readme-ov-file#usage)
for details). The credentials file will be encrypted as part of the initial deployment process before
it is committed to git.

Once the appropriate credentials have been added, modify the following fields in
`clusters/waldur/configmap.yaml` to match the desired flavours, images etc. for your target
cloud:

```
    # Must match the name of the (sealed) secret in credentials.yaml
    cloudCredentialsSecretName: waldur-cluster-config-credentials

    kubernetesVersion: 1.29.5
    machineImageId: f08b688e-645a-4444-a811-c181bd82cc50 # == ubuntu-jammy-kube-v1.29.5-240605-0529

    clusterNetworking:
      externalNetworkId: 57add367-d205-4030-a929-d75617a7c63e

    controlPlane:
      machineFlavor: vm.ska.cpu.general.small
      machineCount: 3

    # NOTE: Waldur is surprisingly resource hungry so it is easier
    # to start with an initially oversized cluster and scale it down
    # later once you have confirmed via the monitoring dashboards
    # that a smaller cluster will suffice.
    nodeGroups:
      - name: group-1
        machineFlavor: vm.ska.cpu.general.small
        machineCount: 3
      - name: group-2
        machineFlavor: vm.ska.cpu.general.eighth
        machineCount: 1
```

For a full list of available configuration options for the Kubernetes cluster, consult the capi-helm-charts
[documentation](https://github.com/stackhpc/capi-helm-charts/tree/main/charts/openstack-cluster) and
[values.yaml](https://github.com/stackhpc/capi-helm-charts/blob/main/charts/openstack-cluster/values.yaml).

Once you are happy with your configuration, the Kubernetes cluster is ready to be deployed by following
https://github.com/stackhpc/capi-helm-fluxcd-config/tree/main#usage. NOTE: It is normal for this deployment
to take a while (e.g. 30 minutes) since there are many steps involved in the bootstrapping process.

After the initial deployment, the Waldur configuration can be modified to suit your needs. The main sources for this
config can be found in the `components/waldur/{helmrelease,configmap}.yaml` files. The Flux controllers running
on the deployed cluster will automatically watch the target GitHub repository for changes to the main branch, so
adding or modifying configuration options on your deployment should be done by proposing, reviewing and merging pull
requests in the repository.

## Adding secrets to the Waldur deployment

The Waldur Helm chart configuration may need to contain some sensitive values such as OIDC client secrets,
this can pose a challenge when all of the required configuration is maintained in a remote Git repository.
The recommended solution is to use Kubernetes [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets).
The following is a 'worked example' of adding Keycloak login functionality to your Waldur deployment while
following recommended best practices for secure secret management.

First, create a client in your Keycloak for Waldur to use - see [these](https://docs.waldur.com/admin-guide/identities/keycloak/)
Waldur docs for detailed instructions but ignore the final section on 'Configuring Waldur', this will be done
via the Helm values instead.

Next, run `cp components/waldur/{example-,}secret.yaml` to create a `secrets.yaml` file and then replace the
Keycloak client name, secret and discovery URL variables with the values for your newly created Keycloak client.

Overwrite the secret with a sealed secret using `kubeseal`:

```
kubeseal \
  --kubeconfig clusters/waldur/kubeconfig \
  --format yaml \
  --controller-name sealed-secrets \
  --controller-namespace sealed-secrets-system \
  --secret-file components/waldur/secret.yaml \
  --sealed-secret-file components/waldur/secret.yaml
```

Inspecting the content of this file should now show that the secret values have been encrypted.

Next, configure Flux to include this sealed secret in the list of files that it watches by adding `secret.yaml`
to the resources list in `components/waldur/kustomization.yaml`.

Add the following content to the `valuesFrom` list in `components/waldur/helmrelease.yaml` to tell Flux that the new
secret should be used as a source of Helm chart values:

```
  - kind: Secret
    name: waldur-keycloak-config
    valuesKey: keycloakClientId
    targetPath: waldur.socialAuthMethods[0].clientId
  - kind: Secret
    name: waldur-keycloak-config
    valuesKey: keycloakClientSecret
    targetPath: waldur.socialAuthMethods[0].clientSecret
  - kind: Secret
    name: waldur-keycloak-config
    valuesKey: keycloakDiscoveryUrl
    targetPath: waldur.socialAuthMethods[0].discoveryUrl
```

Finally, add the non-secret sections of the required Waldur config to the rest of the Helm chart values in
`components/waldur/configmap.yaml`:

```
    waldur:
      authMethods:
      - LOCAL_SIGNIN
      - SOCIAL_SIGNUP
      socialAuthMethods:
      - label: Keycloak
        provider: keycloak
        # clientId: <stored-in-sealed-secret>
        # clientSecret: <stored-in-sealed-secret>
        # discoveryUrl: <stored-in-sealed-secret>
        managementUrl: ""
        protectedFields:
        - full_name
        - email
```

With all of these changes made, commit them to git and propose a pull request to the remote repository. Once
the PR has been reviewed and merged, Flux will notice the change to the GitHub main branch and apply the new
config to the live deployment.

Once the PR has merged, you can check the status of the various Flux components on the remote cluster using
`kubectl` and the kubeconfig for your cluster (which should have been written to `clusters/waldur/kubeconfig`
as part of the initial cluster deployment). For example, by running

```
kubectl get -A gitrepositories.source.toolkit.fluxcd.io,kustomizations.kustomize.toolkit.fluxcd.io,helmreleases.helm.toolkit.fluxcd.io
```

or by using the Flux CLI:

```
flux get source git
flux get kustomizations
flux get helmrelease -n waldur
```

If the changes have synced correctly from GitHub and the `helmrelease` object is up to date but the Waldur
login UI is still not displaying a 'Login with Keycloak' option then you may need to restart the Waldur
pods using:

```
kubectl rollout restart deployment -n waldur
```

A similar process can be followed for adding any other secret/sensitive configuration options required by
Waldur.

Any general (i.e. non-secret) configuration should be added to the Waldur Helm chart values in
`components/waldur/configmap.yaml`.
