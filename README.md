# Deploying Kubeflow with ArgoCD

This repository contains Kustomize manifests that point to the upstream
manifest of each Kubeflow component and provides an easy way for people
to change their deployment according to their need. ArgoCD application
manifests for each componenet will be used to deploy Kubeflow. The intended
usage is for people to fork this repository, make their desired kustomizations,
run a script to change the ArgoCD application specs to point to their fork
of this repository, and finally apply a master ArgoCD application that will
deploy all other applications.

This is a work in progress at this point.
To run the below script [yq](https://github.com/mikefarah/yq) version 4
must be installed

Initial instructions:

- fork this repo
- modify the kustomizations for your purpose
- run `./setup_repo.sh <your_repo_fork_url>`
- commit and push your changes
- run `kubectl apply -f kubeflow.yaml`

## Folder setup

- [argocd](./argocd): Kustomize files for ArgoCD
- [argocd-applications](./argocd-applications): ArgoCD application for each Kubeflow component
- [cert-manager](./cert-manager): Kustomize files for installing cert-manager v1.2
- [kubeflow](./kubeflow): Kustomize files for installing Kubeflow componenets
  - [common/dex-istio](./kubeflow/common/dex-istio): Kustomize files for Dex auth installation
  - [common/oidc-authservice](./kubeflow/common/oidc-authservice): Kustomize files for OIDC authservice
  - [roles-namespaces](./kubeflow/common/roles-namespaces): Kustomize files for Kubeflow namespace and ClusterRoles
  - [user-namespace](./kubeflow/common/user-namespace): Kustomize manifest to create the profile and namespace for the default Kubeflow user
  - [katib](./kubeflow/katib): Kustomize files for installing Katib
  - [kfserving](./kubeflow/kfserving): Kustomize files for installing KFServing
    - [knative](./kubeflow/knative): Kustomize files for installing KNative
  - [central-dashboard](./kubeflow/notebooks/central-dashboard): Kustomize files for installing the Central Dashboard
  - [jupyter-web-app](./kubeflow/notebooks/jupyter-web-app): Kustomize files for installing the Jupyter Web App
    - [notebook-controller](./kubeflow/notebooks/notebook-controller): Kustomize files for installing the Notebook Controller
  - [pod-defaults](./kubeflow/notebooks/pod-defaults): Kustomize files for installing Pod Defaults (a.k.a. admission webhook)
  - [profile-controller_access-management](./kubeflow/notebooks/profile-controller_access-management): Kustomize files for installing the Profile Controller and Access Management
  - [tensorboards-web-app](./kubeflow/notebooks/tensorboards-web-app): Kustomize files for installing the Tensorboards Web App
    - [tensorboard-controller](./kubeflow/notebooks/tensorboard-controller): Kustomize files for installing the Tensorboard Controller
  - [volumes-web-app](./kubeflow/notebooks/volumes-web-app): Kustomize files for installing the Volumes Web App
  - [operators](./kubeflow/operators): Kustomize files for installing the various operators
  - [pipelines](./kubeflow/pipelines): Kustomize files for installing Kubeflow Pipelines
- [metallb](./metallb): Kustomize files for installing MetalLB
- [rook-ceph](./rook-ceph): Kustomize files for installing Rook Ceph with Block (default), Object and File storage

### Root files

- [kustomization.yaml](./kustomization.yaml): Kustomization file that references the ArgoCD application files in [argocd-applications](./argocd-applications)
- [kubeflow.yaml](./kubeflow.yaml): ArgoCD application that deploys the ArgoCD applications referenced in [kustomization.yaml](./kustomization.yaml)

## Installing ArgoCD

For this installation the HA version of ArgoCD is used.
Due to Pod Tolerations, 3 nodes will be required for this installation. 
If you do not wish to use a HA installation of ArgoCD,
edit this [kustomization.yaml](./argocd/kustomization.yaml) and remove `/ha`
from the URI.

1. Next, to install ArgoCD execute the following command:

    ```bash
    kustomize build argocd/ | kubectl apply -f -
    ```

2. Install the ArgoCD CLI tool from  https://github.com/argoproj/argo-cd/releases/latest
3. Access the ArgoCD UI by exposing it through a LoadBalander, Ingress or by port-fowarding
using `kubectl port-forward svc/argocd-server -n argocd 8080:443`
4. Login to the ArgoCD CLI. First get the default password for the `admin` user:
    `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

    Next, login with the following command:
    `argocd login <ARGOCD_SERVER>  # e.g. localhost:8080 or argocd.example.com`

    Finally, update the account password with:
    `argocd account update-password`
5. You can now login to the ArgoCD UI with your new password.
This UI will be handy to keep track of the created resources
while deploying Kubeflow.

## Installing Kubeflow

The purpose of this repository is to make it easy for people to customize their Kubeflow
deployment and have it managed through a GitOps tool like ArgoCD.
First, fork this repository and clone your fork locally.
Next, apply any customization you require in the kustomize folders of the Kubeflow
applications. Next will follow a set of recommended changes that we encourage everybody
to make.

### Credentials

The default `username`, `password` and `namespace` of this deployment are:
`user`, `12341234` and `kubeflow-user` respectively.
To change these, edit the `user` and `profile-name`
(the namespace for this user) in [params.env](./kubeflow/common/user-namespace/params.env).

Next, in [configmap-path.yaml](./kubeflow/common/dex-istio/configmap-patch.yaml)
under `staticPasswords`, change the `email`, the `hash` and the `username`
for your used account. 

```yaml
staticPasswords:
- email: user
  hash: $2y$12$4K/VkmDd1q1Orb3xAt82zu8gk7Ad6ReFR4LCP9UeYE90NLiN9Df72
  username: user
```

The `hash` is the bcrypt has of your password.
You can generate this using https://passwordhashing.com/BCrypt,
or with the command below:

```bash
python3 -c 'from passlib.hash import bcrypt; import getpass; print(bcrypt.using(rounds=12, ident="2y").hash(getpass.getpass()))'
```

### Ingress and Certificate

By default the Istio Ingress Gateway is setup to use a LoadBalancer
and to redirect HTTP traffic to HTTPS. Manifests for MetalLB are provided
to make it easier for users to use a LoadBalancer Service.
Edit the [configmap.yaml](./metallb/configmap.yaml) and set
a range of IP addresses MetalLB can use under `data.config.address-pools.addresses`.
This must be in the same subnet as your cluster nodes.

If you do not wish to use a LoadBalancer, change the `spec.type` in [gateway-service.yaml](./kubeflow/common/istio/gateway-service.yaml)
to `NodePort`.

To provide HTTPS out-of-the-box, the `kubeflow-self-signing-issuer` used by internal
Kubeflow applications is setup to provide a certificate for the Istio Ingress
Gateway.

To use a different certificate for the Ingress Gateway, change
the `spec.issuerRef.name` to the cert-manager ClusterIssuer you would like to use in [ingress-certificate.yaml](./kubeflow/common/istio/ingress-certificate.yaml)
and set the `spec.commonName` and `spec.dnsNames[0]` to your Kubeflow domain.

If you would like to use LetsEncrypt, a ClusterIssuer template if provided in
[letsencrypt-cluster-issuer.yaml](./cert-manager/letsencrypt-cluster-issuer.yaml).
Edit this file according to your requirements and uncomment the line in
the [kustomization.yaml](./cert-manager/kustomization.yaml) file
so it is included in the deployment.

### Customizing the Jupyter Web App

To customize the list of images presented in the Jupyter Web App
and other related setting such as allowing custom images,
edit the [spawner_ui_config.yaml](./kubeflow/notebooks/jupyter-web-app/spawner_ui_config.yaml)
file.

### Change ArgoCD application specs and commit

To simplify the process of telling ArgoCD to use your fork
of this repo, a script is provided that updates the
`spec.source.repoURL` of all the ArgoCD application specs.
Simply run:

```bash
./setup_repo.sh <your_repo_fork_url>
```

To change what Kubeflow or third-party componenets are included in the deployment,
edit the [root kustomization.yaml](./kustomization.yaml) and
comment or uncomment the components you do or don't want.

Next, commit your changes and push them to your repository.

### Deploying Kubeflow with ArgoCD

Once you've commited and pushed your changes to your repository,
you can either choose to deploy componenet individually or
deploy them all at once.
For example, to deploy a single component you can run:

`kubectl apply -f argocd-applications/kubeflow-roles-namespaces.yaml`

To deploy everything specified in the root [kustomization.yaml](./kustomization.yaml),
 execute:

 `kubectl apply -f kubeflow.yaml`

After this, you should start seeing applications being deployed in
the ArgoCD UI and what the resources each application create.

### Updating the deployment

By default, all the ArgoCD application specs included here are
setup to automatically sync with the specified repoURL.
If you would like to change something about your deployment,
simply make the change, commit it and push it to your fork
of this repo. ArgoCD will automatically detect the changes
and update the necessary resources in your cluster.
