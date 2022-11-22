In this course, you will learn the following:
- What is GitOps and why you should adopt it
- Benefits and common pitfalls of GitOps
- How Argo CD works
- How to manage applications with Argo CD
- Sync strategies, secrets, and template customization
- Declarative setup for applications
- What is Progressive Delivery and how it can level up your deployments
- Using Argo Rollouts for blue/green and canary deployments

# What is GitOps:
GitOps is a set of best practices where the entire code delivery process is controlled via Git, including infrastructure and application definition as code and automation to complete updates and rollbacks
The Key GitOps Principles:

    The entire system (infrastructure and applications) is described declaratively.
    The canonical desired system state is versioned in Git.
    Changes approved are automated and applied to the system.
    Software agents ensure correctness and alert on divergence.
The key points here are:

    The state of the cluster is always described in Git. Git holds everything for the application and not just the source code.
    There is no external deployment system with full access to the cluster. The cluster itself is pulling changes and deployment information.
    The GitOps controller is running in a constant loop and always matches the Git state with the cluster state.
# GitOps use cases:

+ Continuous deployment of applications: 
Popular use of GitOps and the one that is applicable to most organizations. Using GitOps for the applications developed by your own organization is one of the most straightforward scenarios for applying GitOps and seeing the benefits it offers.
+ Continuous deployment of cluster resources:
After you adopt GitOps for your own applications, it makes sense to embrace the same principles for your supporting applications that you manage but not necessarily develop. In the case of Kubernetes, these are the applications that take part in the cluster such as metrics, networking agents, service meshes, database, etc.
+ Continuous deployment of infrastructure:
Adopting GitOps for the application is a great step, but you can continue this paradigm to other layers of the infrastructure. If you have a way to define your resources in a declarative way, it should be very easy to adopt the GitOps principles for the infrastructure that powers your applications.
+ Detecting/Avoiding configuration drift:
This is a great way to detect manual (or even unauthorized) changes in your cluster.
Configuration drift between what you think is on a server and what is actually on the server is one of the most common causes of failed deployments.
In the case of Kubernetes, performing manual changes via kubectl is a practice that GitOps can completely eliminate. And even if somebody makes some changes using kubectl, your GitOps agents should notify you immediately. Then it is up to you decide if you want to discard them (and whatever Git has) or accept them (and commit them back to Git)
+ Multi-cluster deployments:
Deploying to a single cluster is already a challenge on its own. If you have multiple clusters at different geographical locations or with different configurations, you need a way to track them and understand how they are different.
This is one of the cases where GitOps really shines. Because the state of all clusters is stored in Git, it is very easy to know what is installed where, who installed it, and when just by looking at the Git history.
A very common question when it comes to multiple environments is finding out what is different between them. If you follow GitOps, the answer is always there just by doing a simple git diff.

# GitOps Pros and Cons:

Some Pros of GitOps:

    Faster deployments
    Safer deployments
    Easier rollbacks
    Straightforward auditing
    Better traceability
    Eliminating configuration drift

# ArgoCD:



There are several questions that you need to answer before making a decision about installation:
    - Will your Argo CD installation manage only the cluster on which it is installed? Will it manage other clusters? Both?
    - Do you want a High Availability (HA) installation or not?
    - Do you want to use plain manifests or are you looking for a Helm chart?
    - How will you manage Argo CD upgrades? With an external system? Or do you wish for Argo CD to manage itself?

## Installation
For quick demos and experimentation, you can deploy ArgoCD by directly using the manifests:
    ```
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
    ```
To get the default password use the command bellow:

    ```
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d > admin-pass.txt
    ```
To expose Argocd
    ```
    kubectl expose -n argocd svc/argocd-server  --port 8080   --name argocd-ext  --type NodePort
    ```


ARGO CLI installation:

    ```
    curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.1.5/argocd-linux-amd64
    chmod +x /usr/local/bin/argocd
    argocd help 
    ```
Example of GitOps using ArgoCD : https://github.com/codefresh-contrib/gitops-pipelines
For a production setup we suggest you use "Autopilot"  or chart helm 

Login with CLI:

We use the insecure option because Argo CD has a self-signed certificate. In a production installation you would have a real certificate and this option should not be used.

    ```
    argocd login localhost:30443 --insecure
    ```

To get app list use the command bellow:

    ```
    argocd app list
    ```

## User management
Once the external URL is ready, you need to decide how users will access the Argo CD UI. There are mainly two approaches:

    - Use a small number of local users. Authentication is handled by Argo CD itself. Best for very small companies (e.g. 2-5 people).
    - Use an SSO provider. Authentication is handled by the provider. Ideal for companies and large organizations


You can read all the details at https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/

## Creating an ArgoCD application
- navigate to the +NEW APP on the left-hand side of the UI. 
- using ArgoCD CLI:

    ```
    argocd app create {APP NAME} --project {PROJECT} --repo {GIT REPO} --path {APP FOLDER} --dest-namespace {NAMESPACE} --dest-server {SERVER URL}
    ```

    PS: Use https://kubernetes.default.svc to reference the same cluster where Argo CD has been deployed
    For a more detailed view of the application configuration, run:

        ```
        argocd app get {APP NAME}
        ```
## Syncing an ArgoCD application
### manual sync:
To sync the app using cli:

```
argocd app sync {APP NAME}
```

Examples:

- The UI starts empty because nothing is deployed on our cluster. Click the "New app" button on the top left and fill the following details:

    application name : demo
    project: default
    repository URL: https://github.com/codefresh-contrib/gitops-certification-examples
    path: ./simple-app
    Cluster: https://kubernetes.default.svc (this is the same cluster where ArgoCD is installed)
    Namespace: default

Leave all the other values empty or with default selections. Finally click the Create button. The application entry will appear in the main dashboard. 

You will see that nothing is deployed yet. The cluster is still empty. The "Out of sync" message means essentially this

- The CLI example:
    ```
    argocd app list
    argocd app get demo
    argocd app history demo
    ```

    delete the previous app to redploy it using cli:

    ```
    argocd app delete demo
    ```
    deploy the app:

    ```
    argocd app create demo2 --project default --repo https://github.com/codefresh-contrib/gitops-certification-examples --path "./simple-app" --dest-namespace default --dest-server https://kubernetes.default.svc
    ```

    sync the app: 

    ```
    argocd app sync demo2
   ```

### auto-sync:

By default, the sync period is 3 minutes. This means that every 3 minutes Argo CD will:

    Gather all applications that are marked to auto-sync.
    Fetch the latest Git state from the respective repositories.
    Compare the Git state with the cluster state.
    Take an action:
        - If both states are the same, do nothing and mark the application as synced.
        - If states are different mark the application as "out-of-sync".
We are able to  change the default period by editing the argocd-cm configmap on the argocd namespace:


In the case of out-of-sync applications, you can further decide if you want Argo CD to discard the cluster state or do nothing at all. More details about this will be discussed later on in the section regarding Git strategies.

PS: We can disable the sync functionality completely by setting the timeout.reconciliation period to 0.

Using Webhooks is very efficient, as now your Argo CD installation will never delay when you commit something to Git. If you only use the default way of polling, then you might have to wait up to 3 minutes (or whatever time you have set as sync period) for Argo CD to detect the changes. With Webhooks, as soon as there is any change in Git, Argo CD will run the sync process.


## Application health:

The possible Status of an application are:

    “Healthy” -> Resource is 100% healthy
    “Progressing” -> Resource is not healthy but still has a chance to reach healthy state
    “Suspended” -> Resource is suspended or paused. The typical example is a cron job
    “Missing” -> Resource is not present in the cluster
    “Degraded” -> Resource status indicates failure or resource could not reach healthy state in time
    “Unknown” -> Health assessment failed and actual health status is unknown

## Sync strategies:


There are 3 parameters that you can change when defining the sync strategy:
   1. Manual or automatic sync.
   2. Auto-pruning of resources - this is only applicable for automatic sync.
   3. Self-Heal of cluster - this is only applicable for automatic sync.
### Auto-pruning:
Auto-pruning defines what Argo CD does when you remove/delete files from Git. If it is enabled, Argo CD will also remove the respective resources in the cluster as well. If disabled, Argo CD will never delete anything from the cluster.
### Self-heal:
Self-heal defines what Argo CD does when you make changes directly to the cluster (via kubectl or any other way). Note that doing manual changes in the cluster is not recommended if you want to follow GitOps principles (as all changes should pass from Git). If enabled, then Argo CD will discard the extra changes and bring the cluster back to the state described in Git.

## Managing secrets:

We Can Vault or Bitnami Sealed
Sealed secrets: https://learning.codefresh.io/path-player?courseid=gitops-with-argo&unit=gitops-with-argo_6177d8794fc59Unit

### installation of sealed secret controller:

```
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
helm install sealed-secrets-controller sealed-secrets/sealed-secrets 
```
### installation of sealed cli:

```
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.19.2/kubeseal-0.19.2-linux-amd64.tar.gz
tar -xvzf kubeseal-<version>-linux-amd64.tar.gz kubeseal
install -m 755 kubeseal /usr/local/bin/kubeseal
```
### example of enryption 
This link contain  an example of encryption with Sealed: https://github.com/samiamerni/gitops-certification-examples/tree/main/secret-app

The folder never-commit-to-git/unsealed_secrets/ under secret-app containt the secrets yaml file that we should encrypt.

To encrypt the secrets use: 

kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
    kubeseal \
      --controller-name=sealed-secrets-controller \
      --controller-namespace=sealed-secrets \
      --format yaml > mysealedsecret.[json|yaml]


```
kubeseal < unsealed_secrets/db-creds.yml > sealed_secrets/db-creds-encrypted.yaml -o yaml
kubeseal < unsealed_secrets/paypal-cert.yml > sealed_secrets/paypal-cert-encrypted.yaml -o yaml
```

Copy the generated files into TOu repo git, then deploy your application using ArgoCD UI (The Sealed secrets controller will automatically decrypt secrets and pass them to your application)

## Declarative Setup:

Using the Argo CD UI for creating applications is great for experimentation and onboarding. However, in a production environment, you should use Git for all operations instead.

To achieve this goal, Argo CD comes with its own custom resources that can be stored in Git, and applied in a cluster using kubectl or even better Argo CD itself.

For the full details of all Argo CD objects, you should consult: https://argo-cd.readthedocs.io/en/stable/operator-manual/declarative-setup/ 

### Example declarative setup:
ArgoCD applications must be deployed in the same namespace as ArgoCD itself.

See the example under the link:  https://github.com/samiamerni/gitops-certification-examples/blob/main/declarative

## deploying with Helm /kustomize

Something important to note is that Argo CD provides native support for Helm, meaning you can directly connect a packaged Helm chart and Argo CD will monitor it for new versions. When this takes place, the Helm chart will no longer function as a Helm chart and instead, is rendered with the Helm template when Argo is installed, using the Argo CD application manifest.

Argo CD then deploys and monitors the application components until both states are identical. The application is no longer a Helm application and is now recognized as an Argo app that can only operated by Argo CD. Hence if you execute the helm list command, you should no longer be able to view your helm release because the Helm metadata no longer exists.

Note that if you use any of the helm commands such as helm list you will not see anything at all. A Helm chart deployed by ArgoCD no longer registers as a Helm deployment. This is because ArgoCD doesn't include the Helm payload information. When deploying a Helm application, Argo CD runs "helm template" and deploys the resulting manifests.

Using CLI:
```
argocd app list
argocd app get helm-gitops-example
argocd app history helm-gitops-example
argocd app delete helm-gitops-example
argocd app create demo --project default --repo https://github.com/codefresh-contrib/gitops-certification-examples --path "./helm-app/" --sync-policy auto --dest-namespace default --dest-server https://kubernetes.default.svc
```

## Progressive Delivery:

Progressive Delivery is the practice of deploying an application in a gradual manner allowing for minimum downtime and easy rollbacks.

### Blue Grenn deployments:

Blue/Green deployments are one of the simplest ways to minimize deployment downtime. Blue/Green deployments are not specific to Kubernetes and can be used even for traditional applications that reside on Virtual Machines.

An exmaple of rollout blue/green: https://github.com/samiamerni/gitops-certification-examples/tree/main/blue-green-app

```
kubectl argo rollouts list rollouts
kubectl argo rollouts status simple-rollout
kubectl argo rollouts get rollout simple-rollout
kubectl argo rollouts set image simple-rollout webserver-simple=docker.io/kostiscodefresh/gitops-canary-app:v2.0
kubectl argo rollouts get rollout simple-rollout
kubectl argo rollouts promote simple-rollout
kubectl argo rollouts get rollout simple-rollout --watch
```

### Canary deployment:

Blue/Green deployments are great for minimizing downtime after a deployment, but they are not perfect. If your new version has a hidden issue that manifests itself only after some time (i.e. it is not detected by your smoke tests), then all your users will be affected because the traffic switch is all or nothing.

An improved deployment method is canary deployments. This functions similar to blue/green, but instead of switching 100% of live traffic all at once to the new version, you can instead move only a subset of users.

Canry rollout gives 3 kubernetes serveices:
- rollout-canary-all-traffic: service is capturing all live traffic from actual users of the application
- rollout-canary-active: this will always point to the stable/previous version of the software.
- rollout-canary-preview: this will only route traffic to the canary/new versions


#  commands to remember: 

```
argocd app list
argocd app get demo
argocd app history demo
argocd app delete demo
```




1. Install the client-side tool (kubeseal) as explained in the docs below:

    https://github.com/bitnami-labs/sealed-secrets#installation-from-source

2. Create a sealed secret file running the command below:

    kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
    kubeseal \
      --controller-name=sealed-secrets-controller \
      --controller-namespace=default \
      --format yaml > mysealedsecret.[json|yaml]

The file mysealedsecret.[json|yaml] is a commitable file.

If you would rather not need access to the cluster to generate the sealed secret you can run:

    kubeseal \
      --controller-name=sealed-secrets-controller \
      --controller-namespace=default \
      --fetch-cert > mycert.pem

to retrieve the public cert used for encryption and store it locally. You can then run 'kubeseal --cert mycert.pem' instead to use the local cert e.g.

    kubectl create secret generic secret-name --dry-run=client --from-literal=foo=bar -o [json|yaml] | \
    kubeseal \
      --controller-name=sealed-secrets-controller \
      --controller-namespace=default \
      --format [json|yaml] --cert mycert.pem > mysealedsecret.[json|yaml]
