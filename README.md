# Configuring namespace specific policies

One cluster can be shared between multiple tenants. For multi-tenancy, the namespaces used by different tenants should have different policies according to the best practices for multi-tenancy. This doc walks you through the steps for configuring namespace specific policies, such as Role, RoleBinding and NetworkPolicy for a cluster shared between tenants.

## Objectives
In this tutorial, you will
- Learn how to use Kustomize to get namespace specific policies from a common base.
- Learn how to sync policies in your Git repository to a cluster.

## Before you begin
This section describes prerequisites you must meet before this tutorial.
- ConfigSync is installed on your cluster, with the version at least 1.7.0. If not, you can install it following the installation instructions.
- Configure ConfigSync so that it can sync from multiple repositories. If not, you can configure it following the steps to enable it.
- You already read the tutorial syncing from multiple repositories and familiar with the concept root repository.
- `git` is installed in your local machine.
- `kustomize` is installed in your local machine. If not, you can install the binary following the installation instructions.

## Create namespace specific policies

### Get the example configuration
The example Git repository contains three namespaces for different tenants. The repository contains the  following directories and files.
```
├── kustomization
│   ├── base
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── networkpolicy.yaml
│   │   ├── rolebinding.yaml
│   │   └── role.yaml
│   ├── tenant-a
│   │   ├── kustomization.yaml
│   │   └── rolebinding.yaml
│   ├── tenant-b
│   │   ├── kustomization.yaml
│   │   └── rolebinding.yaml
│   └── tenant-c
│       ├── kustomization.yaml
│       └── rolebinding.yaml
├── namespaces
│   ├── tenant-a
│   │   └── manifest.yaml
│   ├── tenant-b
│   │   └── manifest.yaml
│   └── tenant-c
│       └── manifest.yaml
└── README.md
```

The directory `kustomization` contains the configuration in kustomize format. They are for one base and  three overlays `tenant-a`, `tenant-b` and `tenant-c`. Each overlay is a customization of the shared `base`. The difference between different overlays is from two parts:
- Namespace. The configuration inside the directory `kustomization/<TENANT>` is all in the namespace `<TENANT>`. This is achieved by adding the namespace directive in `kustomization/<TENANT>/kustomization.yaml`. For example, in `kustomization/tenant-a/kustomization.yaml`:
Namespace: tenant-a

- RoleBinding. For each tenant, the RoleBinding is for a different Group. For example, in `kustomization/tenant-a`, the RoleBinding is for the group `tenant-a-admin@mydomain.com`. This is achieved by applying the patch file `kustomization/tenant-a/rolebinding.yaml`. So the RoleBinding from the `base` is overwritten.

Fork the example repository into your organization and clone the forked repo locally.

```
$ git clone https://github.com/<YOUR_ORGANIZATION>/namespaced-configuration.git configuration
```

After this, the example configuration is under your local directory `configuration`.

### [optional] Update the namespace specific policies
If you need to update some configuration, you can follow the instructions in this session. It is optional and shouldn’t affect the steps below.
#### Update the base
When you add new configuration or update configuration under the directory `configuration/kustomization/base`, the change will be propagated to configuration for all of `tenant-a`, `tenant-b` and `tenant-c`.
#### Update an overlay
An overlay is a kustomization that depends on another customization. In this example, there are three overlays: `tenant-a`, `tenant-b` and `tenant-c`. If you only need to update some configuration in one overlay, for example, add another Role to  `tenant-a`. Then you only need to touch the directory `configuration/kustomization/tenant-a`.


After the update, you should rebuild the kustomize output for each namespace following the commands.
```
$ kustomize build configuration/kustomization/tenant-a -o configuration/namespaces/tenant-a/manifest.yaml
$ kustomize build configuration/kustomization/tenant-b -o configuration/namespaces/tenant-b/manifest.yaml
$ kustomize build configuration/kustomization/tenant-c -o configuration/namespaces/tenant-c/manifest.yaml
```

Then you can commit and push the update.

```
$ git add .
$ git commit -m 'update configuration'
$ git push origin main
```

## Sync namespace specific policies

### No existing RootSync
If there is no existing RootSync custom resource in your cluster, you can create one with the following configuration.

```yaml
# root-sync.yaml
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured
  git:
    repo: https://github.com/<YOUR_ORGANIZATION>/namespaced-configuration.git
    branch: main
    dir: "namespaces"
    # We recommend securing your source repository.
    # Other supported auth: `ssh`, `cookiefile`, `token`, `gcenode`.
    auth: none
    # Refer to a Secret you create to hold the private key, cookiefile, or token.
    # secretRef:
    #   name: SECRET_NAME
```

Then apply it to the cluster

```shell script
$ kubectl apply -f root-sync.yaml
```

### With existing RootSync
If there is an existing RootSync custom resource in your cluster. It syncs to your root Git repository for the current cluster <ROOT_REPO>. To sync the namespace specific policies, you need to declare both the namespaces and policies for those namespaces. This can be done with the following steps.

Clone the root repository to your local machine.
```
$ git clone <ROOT_REPO> root_repo
```

Copy the namespace specific policies into the root repository

```
$ cp configuration/namespaces/tenant-a/manifest.yaml  root_repo/namespaces/tenanat-a/manifest.yaml
$ cp configuration/namespaces/tenant-b/manifest.yaml  root_repo/namespaces/tenanat-b/manifest.yaml
$ cp configuration/namespaces/tenant-c/manifest.yaml  root_repo/namespaces/tenanat-c/manifest.yaml
```

Commit and push the changes to the remote root repository.

```
$ git add .
$ git commit -m 'add namespaces and policies'
$ git push origin main
```

## Verify namespace specific policies are synced
Now you can verify that the namespace specific policies are synced to the cluster. This can be done by commands similar to the following
```
# Verify the RoleBinding exist
$ kubectl get RoleBinding/tenant-admin-rolebinding -n tenant-a

# Verify the Role exist
$ kubectl get Role/tenant-admin -n tenant-a

# Verify the NetworkPolicy exist
$ kubectl get NetworkPolicy/deny-all -n tenant-a
```


## Cleanup
To clean up the tenant namespaces and policies for them, we recommend removing the directories that contain their configuration from the root repository.

```
$ rm -r root_repo/namespaces/tenant-a configuration/namespaces/tenant-b configuration/namespaces/tenant-c
$ git add .
$ git commit -m 'clean up'
$ git push
```

When the last commit from the root repository is synced, the three namespace `tenant-a`, `tenant-b` and `tenant-c` are deleted from the cluster.


