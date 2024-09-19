![Argo CD Diff Preview](./images/title_dark.png)

Argo CD Diff Preview is a tool that renders the diff between two branches in a Git repository. It is designed to render manifests generated by Argo CD, providing a clear and concise view of the changes between two branches. It operates similarly to Atlantis for Terraform, creating a plan that outlines the proposed changes.

### 3 Example Pull Requests:
- [Helm Example | Internal Chart](https://github.com/dag-andersen/argocd-diff-preview/pull/16)
- [Helm example | External Chart: Nginx](https://github.com/dag-andersen/argocd-diff-preview/pull/15)
- [Kustomize Example](https://github.com/dag-andersen/argocd-diff-preview/pull/12)

![](./images/example-1.png)

## ArgoCon 2024 Talk

<img align="right" src="./images/ArgoConLogoOrange.svg" width="30%"> `argocd-diff-preview` will be presented at ArgoCon 2024 in Utah, US. The talk will cover the current tools and methods for visualizing code changes in GitOps workflows and introduce this new approach, which uses ephemeral clusters to render accurate diffs directly on your pull requests.

- [GitOps Safety: Rendering Accurate ArgoCD Diffs Directly on Pull Requests](
https://colocatedeventsna2024.sched.com/event/1izsL/gitops-safety-rendering-accurate-argocd-diffs-directly-on-pull-requests-dag-bjerre-andersen-visma-regina-voloshin-octopus-deploy)


## Why do we need this?

In the Kubernetes world, we often use templating tools like Kustomize and Helm to generate our Kubernetes manifests. These tools make maintaining and streamlining configuration easier across applications and environments. However, they also make it harder to visualize the application's actual configuration in the cluster.

Mentally parsing Helm templates and Kustomize patches is hard without rendering the actual output. Thus, making mistakes while modifying an application's configuration is relatively easy.

In the field of GitOps and infrastructure as code, all configurations are checked into Git and modified through PRs. The code changes in the PR are reviewed by a human, who needs to understand the changes made to the configuration. This is hard when the configuration is generated through templating tools like Kustomize and Helm.

## Overview

![](./images/flow_dark.png)

The safest way to make changes to you Helm Charts and Kustomize Overlays in your GitOps repository is to let Argo CD render them for you. This can be done by spinning up an ephemeral cluster in your automated pipelines. Since the diff is rendered by Argo CD itself, it is as accurate as possible.

The implementation is actually quite simple. It just follows the steps below:

#### 10 Steps
1. Start a local cluster
2. Install Argo CD
3. Add the required credentials (git credentials, image pull secrets, etc.)
4. Fetch all Argo CD applications from your PR branch
   - Point their `targetRevision` to the Pull Request branch
   - Remove the `syncPolicy` from the applications (to avoid the applications syncing locally)
5. Apply the modified applications to the cluster
6. Let Argo CD do its magic
7. Extract the rendered manifests from the Argo CD server
8. Repeat steps 4–7 for the base branch (main branch)
9. Create a diff between the manifests rendered from each branch
10. Display the diff in the PR

## Features

- Renders manifests generated by Argo CD
- Does not require access to your real cluster or Argo CD instance. The tool runs in complete isolation.
- Can be run locally before you open the pull request
- Works with private repositories and Helm charts
- Provides a clear and concise view of the changes
- Render resources from external sources (e.g., Helm charts). For example, when you update the chart version of Nginx, you can get a render of the new output. For example, this is useful to spot changes in default values. [PR example](https://github.com/dag-andersen/argocd-diff-preview/pull/15). 

#### Not supported
- [Argo CD Config Management Plugins](https://argo-cd.readthedocs.io/en/stable/operator-manual/config-management-plugins/)

---

> [!TIP]
> 
> ## Try demo locally with 3 simple commands!
> 
> First, make sure Docker is running. E.g., run `docker ps` to see if it's running.
> 
> Second, run the following 3 commands:
> 
> ```bash
> git clone https://github.com/dag-andersen/argocd-diff-preview base-branch --depth 1 -q 
> git clone https://github.com/dag-andersen/argocd-diff-preview target-branch --depth 1 -q -b helm-example-3
> docker run \
>    --network host \
>    -v /var/run/docker.sock:/var/run/docker.sock \
>    -v $(pwd)/output:/output \
>    -v $(pwd)/base-branch:/base-branch \
>    -v $(pwd)/target-branch:/target-branch \
>    -e TARGET_BRANCH=helm-example-3 \
>    -e REPO=dag-andersen/argocd-diff-preview \
>    dagandersen/argocd-diff-preview:v0.0.16
> ```
> 
> and the output would be something like this:
> 
> ```
> ...
> 🚀 Creating cluster...
> 🦑 Installing Argo CD...
> ...
> 🌚 Getting resources for base-branch
> 🌚 Getting resources for target-branch
> ...
> 🔮 Generating diff between main and helm-example-3
> 🙏 Please check the ./output/diff.md file for differences
> ```
> 
> Finally, you can view the diff by running `cat ./output/diff.md`. The diff should look something like [this](https://github.com/dag-andersen/argocd-diff-preview/pull/16)

## Installation and Usage

### Run as container

Pre-requisites:
- Install: [Docker](https://docs.docker.com/get-docker/)

```bash
docker run \
   --network host \                                   # This is required so the container can access the local cluster on the host's docker daemon.
   -v /var/run/docker.sock:/var/run/docker.sock \     # This is required to access the host's docker daemon.
   -v <path-to-main-branch>:/base-branch \
   -v <path-to-pr-branch>:/target-branch \
   -v $(pwd)/output:/output \
   -e BASE_BRANCH=main \
   -e TARGET_BRANCH=<name-of-the-target-branch> \
   -e REPO=<owner/repo-name> \
   dagandersen/argocd-diff-preview:v0.0.16
```

Example on how to use it: ["Try demo locally with 3 simple commands!"](./README.md#try-demo-locally-with-3-simple-commands)

### Run as binary

Pre-requisites:
- Install: [Git](https://git-scm.com/downloads), [Docker](https://docs.docker.com/get-docker/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) OR [minikube](https://minikube.sigs.k8s.io/docs/start/), [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

Check the [releases](https://github.com/dag-andersen/argocd-diff-preview/releases) and find the correct binary for your operating system.

Example for downloading and running on macOS:

```bash
curl -LJO https://github.com/dag-andersen/argocd-diff-preview/releases/download/v0.0.16/argocd-diff-preview-Darwin-x86_64.tar.gz
tar -xvf argocd-diff-preview-Darwin-x86_64.tar.gz
sudo mv argocd-diff-preview /usr/local/bin
argocd-diff-preview --help
```

<details>
  <summary>Expand section - Example of how to use it </summary>

Pull down the two branches you want to compare from your repository.

The main branch will be cloned into the `base-branch` folder, and the target branch will be cloned into the `target-branch` folder.

```bash
git clone https://github.com/<owner>/<repo-name> base-branch --depth 1 -q 
git clone https://github.com/<owner>/<repo-name> target-branch --depth 1 -q -b <your-branch>
```

And now you are ready to run the tool:
  
```bash
argocd-diff-preview \
   --repo <owner>/<repo-name> \
   --base-branch main \
   --target-branch <your-branch>
```

</details>

### Run from source

Pre-requisites:
- Install: [Git](https://git-scm.com/downloads), [Docker](https://docs.docker.com/get-docker/), [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/), [kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) OR [minikube](https://minikube.sigs.k8s.io/docs/start/), [Argo CD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/), [Rust](https://www.rust-lang.org/tools/install)

```bash
git clone https://github.com/dag-andersen/argocd-diff-preview
cd argocd-diff-preview
cargo run -- --help
```

<details>
  <summary>Expand section - Example of how to use it </summary>

Pull down the two branches you want to compare from your repository.

The main branch will be cloned into the `base-branch` folder, and the target branch will be cloned into the `target-branch` folder.

```bash
git clone https://github.com/<owner>/<repo-name> base-branch --depth 1 -q 
git clone https://github.com/<owner>/<repo-name> target-branch --depth 1 -q -b <your-branch>
```

And now you are ready to run the tool:
  
```bash
cargo run -- \
   --repo <owner>/<repo-name> \
   --base-branch main \
   --target-branch <your-branch>
```

</details>


### Usage in a GitHub Actions Workflow

```yaml
name: Argo CD Diff Preview

on:
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          path: pull-request

      - uses: actions/checkout@v4
        with:
          ref: main
          path: main

      - name: Generate Diff
        run: |
          docker run \
            --network=host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v $(pwd)/main:/base-branch \
            -v $(pwd)/pull-request:/target-branch \
            -v $(pwd)/output:/output \
            -e TARGET_BRANCH=${{ github.head_ref }} \
            -e REPO=${{ github.repository }} \
            dagandersen/argocd-diff-preview:v0.0.16

      - name: Post diff as comment
        run: |
          gh pr comment ${{ github.event.number }} --repo ${{ github.repository }} --body-file output/diff.md --edit-last || \
          gh pr comment ${{ github.event.number }} --repo ${{ github.repository }} --body-file output/diff.md
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Private repositories and Helm charts

In the simple code examples above, we do not provide the cluster with any credentials, which only works if the image/Helm Chart registry and the Git repository are public. Since your repository might not be public you need to provide the tool with the necessary read-access credentials for the repository. This can be done by placing the Argo CD repo secrets in folder mounted at `/secrets`. When the tool starts, it will simply run `kubectl apply -f /secrets` to apply every resource to the cluster, before starting the rendering process.

Example of accessing a private repository with a GitHub token: 

```yaml
jobs:
  build:
    ...
    steps:
      ...
    - name: Prepare secrets
      run: |
        mkdir secrets
        cat > secrets/secret.yaml << "EOF"
        apiVersion: v1
        kind: Secret
        metadata:
          name: private-repo
          namespace: argocd
          labels:
            argocd.argoproj.io/secret-type: repo-creds
        stringData:
          url: https://github.com/${{ github.repository }}
          password: ${{ secrets.GITHUB_TOKEN }}      ⬅️ Short-lived GitHub Token
          username: not-used
        EOF

    - name: Generate Diff
      run: |
        docker run \
          --network=host \
          -v /var/run/docker.sock:/var/run/docker.sock \
          -v $(pwd)/main:/base-branch \
          -v $(pwd)/pull-request:/target-branch \
          -v $(pwd)/output:/output \
          -v $(pwd)/secrets:/secrets \               ⬅️ Mount the secrets folder
          -e TARGET_BRANCH=${{ github.head_ref }} \
          -e REPO=${{ github.repository }} \
          dagandersen/argocd-diff-preview:v0.0.16
```

For more info, see the [Argo CD docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/argocd-repo-creds-yaml/)

### Helm/Kustomize generated ArgoCD applications

`argocd-diff-preview` will only look for YAML files in the repository with `kind: Application` or `kind: ApplicationSet`. If your applications are generated from a Helm chart or Kustomize template, you will have to add a step in the pipeline that renders the chart/template.

Helm and Kustomize examples:

```yaml
jobs:
  build:
    ...
    steps:
      ...
    - uses: actions/checkout@v4
      with:
        path: pull-request

    - name: Generate with helm chart
      run: helm template pull-request/some/path/my-chart > pull-request/rendered-apps.yaml

    - name: Generate with kustomize
      run: kustomize build pull-request/some/path/my-kustomize > pull-request/rendered-apps.yaml
      
    - name: Generate Diff
      run: |
        docker run \
          --network=host \
          -v /var/run/docker.sock:/var/run/docker.sock \
          -v $(pwd)/main:/base-branch \
          ...
```
This will place the rendered manifests inside the `pull-request` folder, and the tool will pick them up.

### Selecting which applications to render

Rendering the manifests generated by all applications in the repository on each pull request is slow. Limiting the number of applications rendered can speed up the rendering process significantly. 

By default, `argocd-diff-preview` will render all applications in the repository.

Limiting the number of applications rendered can be done in three ways:

#### Label selector

Run the tool with the `--selector` flag to filter applications based on labels. The flag supports `=`, `==`, and `!=`.

*Example:*
```bash
argocd-diff-preview --selector "team=a"
```

will target the following application :arrow_down:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  labels:
    team: a
spec:
  ...
```

#### Ignoring applications

You can instruct the tool to ignore certain applications by adding the annotation `argocd-diff-preview/ignore: "true"` to the application manifest.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  annotations:
    argocd-diff-preview/ignore: "true"
spec:
  ...
```

#### File Regex

Alternatively, you can filter applications using regex. Running the tool with `--file-regex` allows it to run only on manifest's which filepaths match a particular regex.

For example, if someone in your organization from *Team A* changes to one of their applications, the tool can be run with `--file-regex=/Team-A/` so it only renders changes in folders matching `*/Team-A/*`.


## Options
```
USAGE:
    argocd-diff-preview [FLAGS] [OPTIONS] --repo <repo> --target-branch <target-branch>

FLAGS:
    -d, --debug      Activate debug mode
    -h, --help       Prints help information
    -V, --version    Prints version information

OPTIONS:
        --argocd-version <argocd-version>     Argo CD version [env: ARGOCD_VERSION=]  [default: stable]
    -b, --base-branch <base-branch>           Base branch name [env: BASE_BRANCH=]  [default: main]
        --base-branch-folder <folder>         Base branch folder [env: BASE_BRANCH_FOLDER=]  [default: base-branch]
    -i, --diff-ignore <diff-ignore>           Ignore lines in diff. Example: use 'v[1,9]+.[1,9]+.[1,9]+' for ignoring changes caused by version changes following semver [env: DIFF_IGNORE=]
    -r, --file-regex <file-regex>             Regex to filter files. Example: "/apps_.*\.yaml" [env: FILE_REGEX=]
        --kustomize-build-options <options>   kustomize.buildOptions for argocd-cm ConfigMap [env: KUSTOMIZE_BUILD_OPTIONS=]
    -c, --line-count <line-count>             Generate diffs with <n> lines above and below the highlighted changes in the diff. [env: LINE_COUNT=]  [Default: 10] 
        --local-cluster-tool <tool>           Local cluster tool. Options: kind, minikube [env: LOCAL_CLUSTER_TOOL=] [default: auto]
        --max-diff-length <length>            Max diff message character count. [env: MAX_DIFF_LENGTH=]  [Default: 65536] (GitHub comment limit)
    -o, --output-folder <output-folder>       Output folder where the diff will be saved [env: OUTPUT_FOLDER=]  [default: ./output]
        --repo <repo>                         Git Repository. Format: OWNER/REPO [env: REPO=]
    -s, --secrets-folder <secrets-folder>     Secrets folder where the secrets are read from [env: SECRETS_FOLDER=]  [default: ./secrets]
    -l, --selector <selector>                 Label selector to filter on, supports '=', '==', and '!='. (e.g. -l key1=value1,key2=value2) [env: SELECTOR=]
    -t, --target-branch <target-branch>       Target branch name [env: TARGET_BRANCH=]
        --target-branch-folder <folder>       Target branch folder [env: TARGET_BRANCH_FOLDER=]  [default: target-branch]
        --timeout <timeout>                   Set timeout [env: TIMEOUT=]  [default: 180]
```

## All contributors

Contributions are welcome!

<a href="https://github.com/dag-andersen/argocd-diff-preview/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=dag-andersen/argocd-diff-preview" />
</a>

## Roadmap
- Make a dedicated GitHub Action that wraps the Docker container, so the tool becomes more user-friendly.  
- Delete Argo CD Applications, when they have been parsed by the tool, so Argo CD can focus on the remaining applications, which hopefully speeds up the rendering process.

> [!IMPORTANT]
> ## Questions, issues, or suggestions
> If you experience issues or have any questions or suggestions, please open an issue in this repository! 🚀

