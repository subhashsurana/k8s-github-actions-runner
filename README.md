This Repository is to install Actions Runner Controller for self hosting your Github Actions Runner on AWS EKS Cluster

Below are the Steps required for installing the necessary Kubernetes resources

Reference to original Repository: https://github.com/actions-runner-controller/actions-runner-controller

1. If Helm is not installed then follow below steps

     curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

    chmod 700 get_helm.sh

    ./get_helm.sh

2. To Install cert-manager 
    for latest version check: 
    https://cert-manager.io/docs/installation/supported-releases/ 
    
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    helm install \
    --wait --create-namespace \
    cert-manager jetstack/cert-manager \
    --namespace cert-manager \
    --version v1.6.1 \
    --set installCRDs=true

3. Create Namespace using below command 
    kubectl create ns actions-runner-system
                    OR
    kubectl apply -f k8s/namespace.yaml

There are two ways for actions-runner-controller to authenticate with the GitHub API (only 1 can be configured at a time however):

    Using a GitHub App (not supported for enterprise level runners due to lack of support from GitHub)
    Using a PAT

Functionality wise, there isn't much of a difference between the 2 authentication methods. The primarily benefit of authenticating via a GitHub App is an increased API quota.

4. Deploying using Github App Authentication (Replace with the actual values obtained)

You can create a GitHub App for either your user account or any organization, below are the app permissions required for each supported type of runner:

Note: Links are provided further down to create an app for your logged in user account or an organization with the permissions for all runner types set in each link's query string

Required Permissions for Repository Runners:
Repository Permissions

    Actions (read)
    Administration (read / write)
    Checks (read) (if you are going to use Webhook Driven Scaling)
    Metadata (read)

Required Permissions for Organization Runners:
Repository Permissions

    Actions (read)
    Metadata (read)

Organization Permissions

    Self-hosted runners (read / write)

Subscribe to events

At this point you have a choice of configuring a webhook, a webhook is needed if you are going to use webhook driven scaling. The webhook can be configured centrally in the GitHub app itself or separately. In either case the event details are:

    Check run (required for all webhook driven scaling events)
    Workflow job (optionally) (required for webhook driven scaling with workflow_job events

    
    Check: https://github.com/actions-runner-controller/actions-runner-controller#deploying-using-github-app-authentication

    export APP_ID=XXXXXX
    export INSTALLATION_ID=XXXXXXXX
    export PRIVATE_KEY_FILE_PATH=~/.ssh/*.pem

Kubectl Deployment:


    kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_app_id=${APP_ID} \
    --from-literal=github_app_installation_id=${INSTALLATION_ID} \
    --from-file=github_app_private_key=${PRIVATE_KEY_FILE_PATH}


5. Deploying Using PAT Authentication 
Personal Access Tokens can be used to register a self-hosted runner by actions-runner-controller.
og-in to a GitHub account that has admin privileges for the repository, and create a personal access token with the appropriate scopes listed below:

Required Scopes for Repository Runners

    repo (Full control)

Required Scopes for Organization Runners

    repo (Full control)
    admin:org (Full control)
    admin:public_key (read:public_key)
    admin:repo_hook (read:repo_hook)
    admin:org_hook (Full control)
    notifications (Full control)
    workflow (Full control)

Once you have created the appropriate token, deploy it as a secret to your Kubernetes cluster that you are going to deploy the solution on:

Kubectl Deployment:


    export GITHUB_TOKEN='your pat token here'
    kubectl create secret generic controller-manager \
    -n actions-runner-system \
    --from-literal=github_token=${GITHUB_TOKEN}

Helm Deployment:

Also, you can Configure necessary values.yaml for deploying the secret via Helm

6. Installation of Actions Runner Controller via Helm Chart

Install using below helm command:

    helm upgrade actions-runner-controller \
    actions-runner-controller/actions-runner-controller  \
    --install --namespace actions-runner-system \
    --values k8s/Charts/values.yaml

Kubectl Deployment:

REPLACE "v0.21.1" with the version you wish to deploy

    kubectl replace -f https://github.com/actions-runner-controller/actions-runner-controller/releases/download/v0.21.1/actions-runner-controller.yaml

Usage:

GitHub self-hosted runners can be deployed at various levels in a management hierarchy:

    The repository level
    The organization level
    The enterprise level

There are two ways to use this controller:

    Manage runners one by one with Runner.
    Manage a set of runners with RunnerDeployment.

Repository Runners:

To launch a single self-hosted runner, check the YAML manifest file which includes Runner resource. After applying via Kubectl, it will launch a self-hosted runner with name dm-next-api-runner for the actions-runner-controller/actions-runner-controller repository.

$ kubectl apply -f k8s/runner.yaml

runner.actions.summerwind.dev/example-runner created

There will be Runner resouce and a runner pod that will get created

$ kubectl get runners -n actions-runner-system

$ kubectl get pods -n actions-runner-system

Organization Runners: 

To add the runner to an organization, you only need to replace the repository field with organization, so the runner will register itself to the organization.

RunnerDeployments:

You can manage sets of runners instead of individually through the RunnerDeployment kind and its replicas: attribute. This kind is required for many of the advanced features.

There are RunnerReplicaSet and RunnerDeployment kinds that corresponds to the ReplicaSet and Deployment kinds but for the Runner kind.

$ kubectl apply -f k8s/runner-deployment.yaml

Autoscaling:

Check horizontal-runner-autoscaler.yaml

For Pull Driven scaling, metrics spec TotalNumberOfQueuedAndInProgressWorkflowRuns will be resposible for the relevant repositories added.

For Webhook driven scaling, scaleUpTriggers spec will be handling the necessary github events like webhook_job or check_run events.

