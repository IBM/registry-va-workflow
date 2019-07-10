# Tutorial: IBM Cloud Container Registry & Vulnerability Advisor Workflow

To complete this tutorial, follow along here or view the tutorial in the [documentation](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_tutorial_workflow) for IBM Cloud Container Registry.

In this tutorial we will walk through the basic functionality of both IBM Cloud Container Registry and Vulnerability Advisor. These two services are pre-integrated and work together seamlessly in IBM Cloud, and their features provide a robust but straightforward workflow for users of containers. With these two services you can store your container images, ensure the security of both your images and your Kubernetes clusters, control which images can be used in deployments to your clusters, and more.

While much of the information provided in this tutorial is available in greater detail in our "How To" sections, this tutorial will synthesize all of those capabilities into a coherent workflow that facilitates the simple use of IBM Cloud Container Registry and Vulnerability Advisor. View the links throughout to learn more about each capability.

## Objectives

* Understand the core features of IBM Cloud Container Registry and Vulnerability Advisor
* Compose the functionality of these two services into a robust workflow

## Services used

This tutorial uses the following IBM Cloud services:

* [IBM Cloud Container Registry](https://cloud.ibm.com/kubernetes/registry/main/private)
* [IBM Cloud Kubernetes Service](https://cloud.ibm.com/kubernetes/catalog/cluster)

## Before you begin

* [Install Git](https://git-scm.com/)
* [IBM Cloud Developer Tools](https://github.com/IBM-Cloud/ibm-cloud-developer-tools) - Script to install `docker`, `kubectl`, `helm`, `ibmcloud` CLI and required plug-ins
* [Create a free Kubernetes cluster](https://cloud.ibm.com/docs/containers?topic=containers-clusters#clusters_free)

## From code to a running container

Using [IBM Cloud Container Registry](https://www.ibm.com/cloud/container-registry) to store your container images is the easiest way to get an application up and running with IBM Cloud Kubernetes Service. In this section you will build a container image, store it in IBM Cloud Container Registry, and create a Kubernetes deployment using that image.

### Create a namespace

To store your container images in IBM Cloud Container Registry, you first need to create a [namespace](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_setup_cli_namespace#registry_namespace_setup). Throughout this tutorial, replace `<my_namespace>` with your chosen namespace.

1. Log in to IBM Cloud:

    ```
    ibmcloud login [--sso]
    ```

    If you have a federated ID, use `ibmcloud login --sso` to log in. Enter your user name and use the provided URL in your CLI output to retrieve your one-time passcode. You know you have a federated ID when the login fails without the `--sso` and succeeds with the `--sso` option.

2. Create a namespace:

    ```
    ibmcloud cr namespace-add <my_namespace>
    ```

    The namespace must be unique across all IBM Cloud accounts in the same region. Namespaces must have 4 - 30 characters and contain lowercase letters, numbers, hyphens (-), and underscores (_) only. Namespaces must start and end with a letter or number.

### Build and push an image

To [build a container image and push it to IBM Cloud Container Registry](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_images_#registry_images_creating), you need a simple application and a Dockerfile. These and other artifacts that you'll need for this tutorial are in this GitHub repository, so go ahead and clone it if you haven't already.

1. To build the image, run the following command from the directory of this repo:

    ```
    docker build -t us.icr.io/<my_namespace>/hello-world:1 .
    ```

1. Log your local Docker daemon into IBM Cloud Container Registry:

    ```
    ibmcloud cr login
    ```

1. Push the image:

    ```
    docker push us.icr.io/<my_namespace>/hello-world:1
    ```

    You can build your image directly in IBM Cloud instead of building it locally and pushing it separately. Try running `ibmcloud cr build -t us.icr.io/<my_namespace>/hello-world:1`.

1. Confirm that your image uploaded successfully:

    ```
    ibmcloud cr images
    ```

    You can [automate access to IBM Cloud Container Registry](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_access) with API keys and [grant access to IBM Cloud Container Registry resources](https://cloud.ibm.com/docs/services/Registry?topic=registry-iam_access) using IAM.

### Deploy a container using your image

Now that you have an image stored in IBM Cloud Container Registry, you can [run a container on IBM Cloud Kubernetes Service](https://cloud.ibm.com/docs/containers?topic=containers-app#app_cli) using that image.

For the rest of this tutorial, replace `<my_cluster>` with the name of your free cluster.

1. Run the following command and set the environment variable displayed in the output:

    ```
    ibmcloud ks cluster-config <my_cluster> --export
    ```

1. Run your image as a deployment and create a service, accessed through the IP of the worker node, to expose it:

    ```
    kubectl apply -f hello-world.yaml
    ```

1. Find the port used on the worker node by examining your new service:

    ```
    kubectl describe service hello-world
    ```

    Note the number on the `NodePort:` line as `<nodeport>`.

1. In the output of the following command, note the public IP as `<public_IP>`:

    ```
    ibmcloud ks workers <my_cluster>
    ```

1. Access your container/service. You may also use a web browser. If you see "Hello, world!", you're good to go:

    ```
    curl <public_IP>:<nodeport>
    ```

## Secure your images and clusters

When you push an image to a namespace, the image is automatically scanned by [Vulnerability Advisor](https://cloud.ibm.com/docs/services/va?topic=va-va_index) to detect [potential vulnerabilities](https://cloud.ibm.com/docs/services/va?topic=va-va_index#types). If vulnerabilities are found, instructions are provided to help fix the reported vulnerabilities.

To demonstrate these features, we need to push an intentionally vulnerable image.

At the time of writing, the vulnerability information described below was accurate. However, images are continually updated and new CVEs are discovered. You may see new vulnerability information that was not available at the time of writing. If so, this is a further opportunity to practice by following the steps provided by Vulnerability Advisor to resolve any issues you see.

### View the vulnerability report for your image

When a vulnerability is detected in one of your images, a [report](https://cloud.ibm.com/docs/services/va?topic=va-va_index#va_reviewing) is provided that gives more information about the vulnerability as well as remediation steps.

1. Build and push a vulnerable image:

    ```
    docker build -t us.icr.io/<my_namespace>/hello-world:2 -f Dockerfile-vulnerable .
    docker push us.icr.io/<my_namespace>/hello-world:2
    ```

    You can read the Dockerfile to better understand how this image was made vulnerable. In short, a Debian base image is used, and the `apt` package is downgraded to a version that is vulnerable to CVE-2019-3462.

1. List your images, and take note of the `SECURITY STATUS` column:

    ```
    ibmcloud cr images
    ```

    This column conveys the number of issues present in your image. Since the number is nonzero, this image is vulnerable!

1. Use the `vulnerability-assessment` (alias `va`) command to get more information about the vulnerability:

    ```
    ibmcloud cr va us.icr.io/<my_namespace>/hello-world:1
    ```

    Among other things, this output contains the ID of the vulnerability (if applicable), the affected package, and steps to resolve the issue.

### Enforce security in your cluster

Despite the vulnerability present in your image, you are still able to deploy a container to your cluster using this image, which may be undesirable. [Container Image Security Enforcement](https://cloud.ibm.com/docs/services/Registry?topic=registry-security_enforce) allows you to enforce security in a variety of ways. For one, you can prevent vulnerable images from being using in deployments to your cluster.

1. Follow the steps [here](https://cloud.ibm.com/docs/services/Registry?topic=registry-security_enforce#sec_enforce_install) to install Container Image Security Enforcement. This should entail installing `helm` if it's not already installed, adding the appropriate chart repository, and installing Container Image Security Enforcement.

1. The [default policies](https://cloud.ibm.com/docs/services/Registry?topic=registry-security_enforce#default_policies) are actually too restrictive for this tutorial as they involve [image signing](https://cloud.ibm.com/docs/services/Registry?topic=registry-registry_trustedcontent), which you can learn about on your own. Therefore, we have to create custom policies. View the `security.yaml` file, and read about [customizing policies](https://cloud.ibm.com/docs/services/Registry?topic=registry-security_enforce#customize_policies) in order to understand this file's contents. In short, this policy requires all images in your namespace to have no issues reported by Vulnerability Advisor.

1. Update the `security.yaml` file to include your namespace. Look for the following line:

    ```yaml
    - name: us.icr.io/<my_namespace>/*
    ```

1. Apply the custom policies:

    ```
    kubectl apply -f security.yaml
    ```

1. Update `hello-world.yaml` so that it pulls your new, vulnerable image. To do this, simply change the tag:

    ```yaml
    image: us.icr.io/<my_namespace>/hello-world:2
    ```

1. Patch the existing deployment:

    ```
    kubectl apply -f hello-world.yaml
    ```

1. You should see the following error message:

    ```console
    Deny "us.icr.io/<my_namespace>/hello-world:2", the Vulnerability Advisor image scan assessment found issues with the container image that are not exempted. Refer to your image vulnerability report for more details by using the `bx cr va` command.
    ```

    The Vulnerability Advisor verdict is subject to any [exemption policies](https://cloud.ibm.com/docs/services/Registry?topic=va-va_index#va_managing_policy) that you create. If you want to use an image that Vulnerability Advisor considers vulnerable, you can exempt one or more vulnerabilities so that Vulnerability Advisor will not consider them in its verdict. You can see whether an issue is exempted by looking at the `Policy Status` column in the output of the `ibmcloud cr va` command, and you can also list your exemptions with `ibmcloud cr exemptions`.

### Resolve vulnerabilities in your image

As noted previously, Vulnerability Advisor provides remediation steps for each vulnerability present in an image. These are found in the `How To Resolve` column of the `ibmcloud cr va` command output. If you follow the provided steps, you can resolve any issues in your image.

Since CVEs are frequently discovered and patched, this Dockerfile included a contrived vulnerability introduced by downgrading a package to a known vulnerable version. Therefore, to fix our vulnerability we can simply comment out the line that downgrades `apt`.

1. Comment out the following line (place `#` marker at the beginning as shown):

    ```dockerfile
    # RUN apt-get install --allow-downgrades -y apt=1.4.8
    ```

1. Build and push the image again:

    ```
    docker build -t us.icr.io/<my_namespace>/hello-world:2 -f Dockerfile-vulnerable .
    docker push us.icr.io/<my_namespace>/hello-world:2
    ```

1. Wait for the scan to complete to ensure that no issues are present in this image:

    ```
    ibmcloud cr images
    ```

1. Patch the deployment:

    ```
    kubectl apply -f hello-world.yaml
    ```

1. Wait for the deployment to complete:

    ```
    kubectl rollout status deployment hello-world
    ```

    This deployment should now succeed, and you should again be able to access your service and see "Hello, world!" displayed.

## Deploying to non-default Kubernetes namespaces

Once you've created a cluster with IBM Cloud Kubernetes Service, you can pull images from IBM Cloud Container Registry to the `default` Kubernetes namespace out of the box. But what if you want to deploy to other non-default namespaces? Let's give it a try.

1. Create a namespace in Kubernetes. Let's call it `test`:

    ```
    kubectl create namespace test
    ```

1. Set the `metadata.namespace` field to `test` for the deployment and the service in `hello-world.yaml`. You'll need to add this field:

    ```yaml
    metadata:
      name: hello-world
      namespace: test
    ```

1. Apply the configuration:

    ```
    kubectl apply -f hello-world.yaml
    ```

    Since Container Image Security Enforcement is still enabled in your cluster, your deployment will fail immediately with this message:

    ```console
    Error from server: error when creating "hello-world.yaml": admission webhook "va.hooks.securityenforcement.admission.cloud.ibm.com" denied the request:
    Deny "us.icr.io/<my_namespace>/hello-world:2", no valid ImagePullSecret defined for us.icr.io
    ```

    This is because Container Image Security Enforcement knows that this deployment cannot succeed since the `test` namespace is unable to pull images from your IBM Cloud Container Registry namespace. The `default` Kubernetes namespace in an IBM Cloud Kubernetes Service cluster comes [preconfigured with image pull secrets](https://cloud.ibm.com/docs/containers?topic=containers-images#cluster_registry_auth) to pull images from IBM Cloud Container Registry. However, these secrets are not present in your new namespace.

    If you first [remove Container Image Security Enforcement](https://cloud.ibm.com/docs/services/Registry?topic=registry-security_enforce#remove), the `apply` command will complete successfully, but upon inspecting the deployment's sole pod with `kubectl describe po <pod_name> -n test`, the events log will indicate that the cluster is unauthorized to pull the image.

    You need to [set up an image pull secret](https://cloud.ibm.com/docs/containers?topic=containers-images#other) in your new namespace in order to deploy containers to that namespace, which we will do next.

1. Follow the steps [here](https://cloud.ibm.com/docs/containers?topic=containers-images#copy_imagePullSecret) to copy an image pull secret to the `test` namespace and add that secret to the default service account. You can copy all the `icr.io` secrets or just the `us.icr.io` secret since your image is in that local registry.

1. Delete your deployment and reapply the configuration:

    ```
    kubectl delete -f hello-world.yaml
    kubectl apply -f hello-world.yaml
    ```

    This time it should succeed, and you should once again be able to access your container with `curl` or a web browser.

 Well done! You've successfully completed this tutorial.
</staging>
