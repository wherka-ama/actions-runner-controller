# Autoscaling Runner Scale Sets mode

This new autoscaling mode brings numerous enhancements (described in the following sections) that will make your experience more reliable and secure.

## How it works

![arc_hld_v1 drawio (1)](https://user-images.githubusercontent.com/568794/212665433-2d1f3d6e-0ba8-4f02-9d1b-27d00c49abd1.png)

In addition to the increased reliability of the automatic scaling, we have worked on these improvements:

- No longer require cert-manager as a prerequisite for installing actions-runner-controller
- Reliable scale-up based on job demands and scale-down to zero runner pods
- Reduce API requests to `api.github.com`, no more API rate-limiting problems
- The GitHub Personal Access Token (PAT) or the GitHub App installation token is no longer passed to the runner pod for runner registration
- Maximum flexibility for customizing your runner pod template

### Demo

https://user-images.githubusercontent.com/568794/212668313-8946ddc5-60c1-461f-a73e-27f5e8c75720.mp4

## Setup

### Prerequisites

1. Create a K8s cluster, if not available.
    - If you don't have a K8s cluster, you can install a local environment using minikube. See [installing minikube](https://minikube.sigs.k8s.io/docs/start/).
1. Install helm 3, if not available. See [installing Helm](https://helm.sh/docs/intro/install/).

### Install actions-runner-controller

1. Install actions-runner-controller using helm 3. For additional configuration options, see [values.yaml](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set-controller/values.yaml)

    ```bash
    NAMESPACE="arc-systems"
    helm install arc \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
        --version 0.3.0
    ```

1. Generate a Personal Access Token (PAT) or create and install a GitHub App. See [Creating a personal access token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) and [Creating a GitHub App](https://docs.github.com/en/developers/apps/creating-a-github-app).
    - ℹ For the list of required permissions, see [Authenticating to the GitHub API](https://github.com/actions/actions-runner-controller/blob/master/docs/authenticating-to-the-github-api.md#authenticating-to-the-github-api).

1. You're ready to install the autoscaling runner set. For additional configuration options, see [values.yaml](https://github.com/actions/actions-runner-controller/blob/master/charts/gha-runner-scale-set/values.yaml)
    - ℹ **Choose your installation name carefully**, you will use it as the value of `runs-on` in your workflow.
    - ℹ **We recommend you choose a unique namespace in the following steps**. As a good security measure, it's best to have your runner pods created in a different namespace than the one containing the manager and listener pods.

    ```bash
    # Using a Personal Access Token (PAT)
    INSTALLATION_NAME="arc-runner-set"
    NAMESPACE="arc-runners"
    GITHUB_CONFIG_URL="https://github.com/<your_enterprise/org/repo>"
    GITHUB_PAT="<PAT>"
    helm install "${INSTALLATION_NAME}" \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
        --set githubConfigSecret.github_token="${GITHUB_PAT}" \
        oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set --version 0.3.0
    ```

    ```bash
    # Using a GitHub App
    INSTALLATION_NAME="arc-runner-set"
    NAMESPACE="arc-runners"
    GITHUB_CONFIG_URL="https://github.com/<your_enterprise/org/repo>"
    GITHUB_APP_ID="<GITHUB_APP_ID>"
    GITHUB_APP_INSTALLATION_ID="<GITHUB_APP_INSTALLATION_ID>"
    GITHUB_APP_PRIVATE_KEY="<GITHUB_APP_PRIVATE_KEY>"
    helm install arc-runner-set \
        --namespace "${NAMESPACE}" \
        --create-namespace \
        --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
        --set githubConfigSecret.github_app_id="${GITHUB_APP_ID}" \
        --set githubConfigSecret.github_app_installation_id="${GITHUB_APP_INSTALLATION_ID}" \
        --set githubConfigSecret.github_app_private_key="${GITHUB_APP_PRIVATE_KEY}" \
        oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set --version 0.3.0
    ```

1. Check your installation. If everything went well, you should see the following:

    ```bash
    $ helm list -n "${NAMESPACE}"

    NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                    APP VERSION
    arc             arc-systems     1               2023-01-18 10:03:36.610534934 +0000 UTC deployed        gha-runner-scale-set-controller-0.3.0        preview
    arc-runner-set  arc-systems     1               2023-01-18 10:20:14.795285645 +0000 UTC deployed        gha-runner-scale-set-0.3.0            0.3.0
    ```

    ```bash
    $ kubectl get pods -n "${NAMESPACE}"

    NAME                                              READY   STATUS    RESTARTS   AGE
    arc-gha-runner-scale-set-controller-8c74b6f95-gr7zr   1/1     Running   0          20m
    arc-runner-set-6cd58d58-listener                  1/1     Running   0          21s
    ```

1. In a repository, create a simple test workflow as follows. The `runs-on` value should match the helm installation name you used in the previous step.

    ```yaml
    name: Test workflow
    on:
        workflow_dispatch:

    jobs:
    test:
        runs-on: arc-runner-set
        steps:
        - name: Hello world
            run: echo "Hello world"
    ```

1. Run the workflow. You should see the runner pod being created and the workflow being executed.

    ```bash
    $ kubectl get pods -A

    NAMESPACE     NAME                                                  READY   STATUS    RESTARTS      AGE
    arc-systems   arc-gha-runner-scale-set-controller-8c74b6f95-gr7zr   1/1     Running   0             27m
    arc-systems   arc-runner-set-6cd58d58-listener                      1/1     Running   0             7m52s
    arc-runners   arc-runner-set-rmrgw-runner-p9p5n                     1/1     Running   0             21s
    ```

### Upgrade to newer versions

Upgrading actions-runner-controller requires a few extra steps because CRDs will not be automatically upgraded (this is a helm limitation).

1. Uninstall the autoscaling runner set first

    ```bash
    INSTALLATION_NAME="arc-runner-set"
    NAMESPACE="arc-runners"
    helm uninstall "${INSTALLATION_NAME}" --namespace "${NAMESPACE}"
    ```

1. Wait for all the pods to drain

1. Pull the new helm chart, unpack it and update the CRDs. When applying this step, don't forget to replace `<PATH>` with the path of the `gha-runner-scale-set-controller` helm chart:

    ```bash
    helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
        --version 0.3.0 \
        --untar && \
        kubectl replace -f <PATH>/gha-runner-scale-set-controller/crds/
    ```

1. Reinstall actions-runner-controller using the steps from the previous section

## Troubleshooting

### Check the logs

You can check the logs of the controller pod using the following command:

```bash
# Controller logs
kubectl logs -n "${NAMESPACE}" -l app.kubernetes.io/name=gha-runner-scale-set-controller
```

```bash
# Runner set listener logs
kubectl logs -n "${NAMESPACE}" -l actions.github.com/scale-set-namespace=arc-systems -l actions.github.com/scale-set-name=arc-runner-set
```

### Naming error: `Name must have up to characters`

We are using some of the resources generated names as labels for other resources. Resource names have a max length of `263 characters` while labels are limited to `63 characters`. Given this constraint, we have to limit the resource names to `63 characters`.

Since part of the resource name is defined by you, we have to impose a limit on the amount of characters you can use for the installation and namespace names.

If you see these errors, you have to use shorter installation or namespace names.

```bash
Error: INSTALLATION FAILED: execution error at (gha-runner-scale-set/templates/autoscalingrunnerset.yaml:5:5): Name must have up to 45 characters

Error: INSTALLATION FAILED: execution error at (gha-runner-scale-set/templates/autoscalingrunnerset.yaml:8:5): Namespace must have up to 63 characters
```

### If you installed the autoscaling runner set, but the listener pod is not created

Verify that the secret you provided is correct and that the `githubConfigUrl` you provided is accurate.

### Access to the path `/home/runner/_work/_tool` is denied error

You might see this error if you're using kubernetes mode with persistent volumes. This is because the runner container is running with a non-root user and is causing a permissions mismatch with the mounted volume.

To fix this, you can either:

1. Use a volume type that supports `securityContext.fsGroup` (`hostPath` volumes don't support it, `local` volumes do as well as other types). Update the `fsGroup` of your runner pod to match the GID of the runner. You can do that by updating the `gha-runner-scale-set` helm chart values to include the following:

    ```yaml
    spec:
        securityContext:
            fsGroup: 123
        containers:
        - name: runner
        image: ghcr.io/actions/actions-runner:<VERSION> # Replace <VERSION> with the version you want to use
        command: ["/home/runner/run.sh"]
    ```

1. If updating the `securityContext` of your runner pod is not a viable solution, you can workaround the issue by using `initContainers` to change the mounted volume's ownership, as follows:

    ```yaml
    template:
    spec:
        initContainers:
        - name: kube-init
        image: ghcr.io/actions/actions-runner:latest
        command: ["sudo", "chown", "-R", "1001:123", "/home/runner/_work"]
        volumeMounts:
            - name: work
            mountPath: /home/runner/_work
        containers:
        - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
    ```

## Changelog

### v0.3.0

#### Major changes

1. Runner pods are more similar to hosted runners [#2348](https://github.com/actions/actions-runner-controller/pull/2348)
1. Add support for self-signed CA certificates [#2268](https://github.com/actions/actions-runner-controller/pull/2268)
1. Fixed trailing slashes in config URLs breaking installations [#2381](https://github.com/actions/actions-runner-controller/pull/2381)
1. Fixed a bug where the listener pod would ignore proxy settings from env [#2366](https://github.com/actions/actions-runner-controller/pull/2366)
1. Added runner set name field making it optionally configurable [#2279](https://github.com/actions/actions-runner-controller/pull/2279)
1. Name and namespace labels of listener pod have been split [#2341](https://github.com/actions/actions-runner-controller/pull/2341)
1. Added chart name constraints validation on AutoscalingRunnerSet install [#2347](https://github.com/actions/actions-runner-controller/pull/2347)

### v0.2.0

#### Major changes

1. Added proxy support for the controller and the runner pods, see the new helm chart fields [#2286](https://github.com/actions/actions-runner-controller/pull/2286)
1. Added the abiilty to provide a pre-defined kubernetes secret for the auto scaling runner set helm chart [#2234](https://github.com/actions/actions-runner-controller/pull/2234)
1. Enhanced security posture by removing un-required permissions for the manager-role [#2260](https://github.com/actions/actions-runner-controller/pull/2260)
1. Enhanced our logging by returning an error when a runner group is defined in the values file but it's not created in GitHub [#2215](https://github.com/actions/actions-runner-controller/pull/2215)
1. Fixed helm charts issues that were preventing the use of DinD [#2291](https://github.com/actions/actions-runner-controller/pull/2291)
1. Fixed a bug that was preventing runner scale from being removed from the backend when they were deleted from the cluster [#2255](https://github.com/actions/actions-runner-controller/pull/2255) [#2223](https://github.com/actions/actions-runner-controller/pull/2223)
1. Fixed bugs with the helm chart definitions preventing certain values from being set [#2222](https://github.com/actions/actions-runner-controller/pull/2222)
1. Fixed a bug that prevented the configuration of a runner group for a runner scale set [#2216](https://github.com/actions/actions-runner-controller/pull/2216)
