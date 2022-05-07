# EKS Anywhere GitOps
#### Sources
Install EKS Anywhere:   
https://anywhere.eks.amazonaws.com/docs/getting-started/install/    
Create local cluster:    
https://anywhere.eks.amazonaws.com/docs/getting-started/local-environment/  
Manage cluster with GitOps:     
https://anywhere.eks.amazonaws.com/docs/tasks/cluster/cluster-flux/

### Install EKS Anywhere CLI tools

#### Via Homebrew (macOS and Linux)
```bash
brew install aws/tap/eks-anywhere
```

#### Manually (macOS and Linux)
```bash
curl "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
    --silent --location \
    | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
```
Install the `eksctl-anywhere` plugin.
```bash
export EKSA_RELEASE="0.8.2" OS="$(uname -s | tr A-Z a-z)" RELEASE_NUMBER=10
curl "https://anywhere-assets.eks.amazonaws.com/releases/eks-a/${RELEASE_NUMBER}/artifacts/eks-a/v${EKSA_RELEASE}/${OS}/amd64/eksctl-anywhere-v${EKSA_RELEASE}-${OS}-amd64.tar.gz" \
    --silent --location \
    | tar xz ./eksctl-anywhere
sudo mv ./eksctl-anywhere /usr/local/bin/
```
#### Upgrade eksctl-anywhere
```bash
brew update
brew upgrade eks-anywhere
```
If you installed `eksctl-anywhere` manually you should follow the installation steps to download the latest release.    
You can verify your installed version with:
```bash
eksctl anywhere version
```
### Create a `local` GitOps cluster
In order to use GitOps to manage cluster scaling, you need a couple of things:
- A GitHub account
- A cluster configuration file with a `GitOpsConfig`, referenced with a `gitOpsRef` in your Cluster spec
- A Personal Access Token (PAT) for the GitHub account , with permissions to create, clone, and push to a repo

#### Steps
1. Create a GitHub Personal Access Token    
    Create a Personal Access Token (PAT) to access your provided GitHub repository.     
    > It must be scoped for all repo permissions.
    ```bash
    export EKSA_GITHUB_TOKEN=ghp_MyValidPersonalAccessTokenWithRepoPermissions
    ```
1. Generate a cluster config
    ```bash
    CLUSTER_NAME=dev-cluster
    eksctl anywhere generate clusterconfig $CLUSTER_NAME \
    ```
1. Add gitops to the cluster config
    ```yaml
    apiVersion: anywhere.eks.amazonaws.com/v1alpha1
    kind: Cluster
    metadata:
    name: mynewgitopscluster
    spec:
    ... # collapsed cluster spec fields
    # Below added for gitops support
    gitOpsRef:
        kind: GitOpsConfig
        name: my-cluster-name
    ---
    apiVersion: anywhere.eks.amazonaws.com/v1alpha1
    kind: GitOpsConfig
    metadata:
    name: my-cluster-name
    spec:
    flux:
        github:
        personal: true 
        repository: mygithubrepository # Change Me
        owner: mygithubusername # Change Me
    ```
1. Create a GitOps enabled cluster  
    >NOTE: After your cluster has been created the cluster configuration will automatically be committed to your git repo.
    ```bash
    eksctl anywhere create cluster -f $CLUSTER_NAME.yaml
    ```
    Example command output  
    ```
    Performing setup and validations
    ✅ validation succeeded {"validation": "docker Provider setup is valid"}
    Creating new bootstrap cluster
    Installing cluster-api providers on bootstrap cluster
    Provider specific setup
    Creating new workload cluster
    Installing networking on workload cluster
    Installing cluster-api providers on workload cluster
    Moving cluster management from bootstrap to workload cluster
    Installing EKS-A custom components (CRD and controller) on workload cluster
    Creating EKS-A CRDs instances on workload cluster
    Installing AddonManager and GitOps Toolkit on workload cluster
    GitOps field not specified, bootstrap flux skipped
    Deleting bootstrap cluster
    🎉 Cluster created!
    ```
1. Use the cluster  
Once the cluster is created you can use it with the generated `KUBECONFIG` file in your local directory
    ```bash
    export KUBECONFIG=${PWD}/${CLUSTER_NAME}/${CLUSTER_NAME}-eks-a-cluster.kubeconfig
    kubectl get ns
    ```

    Example command output
    ```
    NAME                                STATUS   AGE
    capd-system                         Active   48m
    capi-kubeadm-bootstrap-system       Active   48m
    capi-kubeadm-control-plane-system   Active   48m
    capi-system                         Active   48m
    cert-manager                        Active   49m
    default                             Active   50m
    eksa-system                         Active   48m
    etcdadm-bootstrap-provider-system   Active   48m
    etcdadm-controller-system           Active   48m
    flux-system                         Active   47m
    kube-node-lease                     Active   50m
    kube-public                         Active   50m
    kube-system                         Active   50m
    ```

#### Test GitOps controller   
After your cluster has been created, you can test the GitOps controller by modifying the cluster specification.     
1.  Navigate to the git Repository
    > The repo directory is located in your `pwd`
    ```bash
    cd CLUSTER_NAME
    ```
1. Modify the `workerNodeGroupsConfigurations[0].count` field with your desired changes.

1. Commit the file to your git repository
    ```bash
    git add eksa-cluster.yaml
    git commit -m 'Scaling nodes for test'
    git push origin main
    ```
1. The flux controller will automatically make the required changes.    
If you updated your node count, you can use this command to see the current node state.
```bash
kubectl get nodes 
```
