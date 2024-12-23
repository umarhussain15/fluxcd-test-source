# flux-upgrade-test

Repo to use as a source of Flux and test it's upgrades with Flux Operator

## TODOs

- [ ] Add setup scripts to initiate kind cluster with v 2.2 of flux
- [ ] Add branches for different versions to simulate version upgrade of flux

## Migrating the Flux bootstrap installations to Flux Operator

### To adopt running flux controllers created by `flux bootstrap` cli

We have a cluster where flux controllers are already deployed using `flux bootstrap` command, which are tracking a Git repo.

- Install Flux Operator with `HelmRelease` ( we can also simply use the `helm install` as well) in `flux-system` namespace
  - With HelmRelease make sure to enable the crd `CreateReplace`
- After operator is installed we need to create `FluxInstance` , there we need to set properties:
  - `distribution.version` : We set it to `2.x` to auto update the flux controllers when we have a new v2 release
  - `sync.kind` as GitRepository and `sync.url` pointing to remote git Repo from where flux controller will source it's config
  - `sync.ref` points to the branch in Git to track for new changes
  - `pullSecret` points to existing `Secret` which contains credentials e.g. ssh private key and it's password to access the Git repo over ssh.
  See `FluxInstance` repository for all the configurations: <https://fluxcd.control-plane.io/operator/fluxinstance/>

- Clean up `flux bootstrap` manifests <https://fluxcd.control-plane.io/operator/flux-bootstrap-migration/#cleanup-the-repository>
  - After flux operator starts to work, it will takeover management of the controllers ( by changing the label `app.kubernetes.io/managed-by`). You can check that via `flux trace kustomization flux-system` which will say it's not managed by flux cli.
  - To clean up the manifests checkout the source code branch which flux is tracking
  - Delete the flux resources from kustomization in `flux-system` folder, usually they are `gotk-components.yaml` and `gotk-sync.yaml`
  - You can now place `FluxInstance` resource under the k8s kustomization resources which we applied earlier.
    - If you had kustomize patches targeting flux controllers in old setup under `flux-system/kustomization.yaml` they need to be under `spec.kustomize` of `FluxInstance` <https://fluxcd.control-plane.io/operator/flux-kustomize/>


### To migrate flux operator and flux controllers in another cluster

We want to switch our Kubernetes cluster and want to deploy all the resources tracked by Flux in old cluster. The old cluster is using flux operator to manage
it's flux controllers.

- We first need to create `Secret` inside the new cluster with the credentials of Git which will be used by flux operator.
  - We can use `flux` cli to easily create this secret. First make sure that kube context points to new cluster and there is a `flux-system` namespace present there.
  - Then run the following command to create a secret (We will show ssh private key, [but others are also possible](https://fluxcd.control-plane.io/operator/fluxinstance/#sync-configuration)):

    ```bash
    flux create secret git <k8s-secret-name> \
    --namespace flux-system \
    --url=ssh://git@github.com/my-org/my-fleet.git \
    --private-key-file=path/my-private.key \
    --password=<ssh key password if key has it.>
    ```

- Since this a fresh cluster without any flux controller, we can't use `HelmRelease` currently for flux operator. [We will use `helm install`](https://fluxcd.control-plane.io/operator/install/#helm)and then later flux will adopt the install under existing HelmRelease for the operator.

    ```bash
    helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts flux-operator \
    --namespace flux-system \
    --create-namespace
    ```

- Now we need to apply the `FluxInstance` definition inside the new cluster under `flux-system` form our git repo using kubectl e.g. `kubectl apply -f <path to FluxInstance file>`
- Flux operator will now install the controllers of flux and flux controller will start syncing the state from Git
