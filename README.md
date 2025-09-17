
# Disclosure

Opinions expressed in this blog are my own and do not necessarily reflect that of the company I work for.

# Introduction

This repository demonstrates how to use Red Hat Advanced Cluster Management (ACM) together with GitOps (Argo CD ApplicationSet push model) and PolicyGenerator to deploy policies to selected spoke clusters.

- To reduce the administrative work, a ApplicationSet using the Git directory generator was configured to deploy the spoke clusters Policies who dont have order requirements, this Plolisies are configured under the policies/spoke-platform/*. For each policy, the ApplicationSet automatically creates an Application that ensures the policy is applied to the target clusters.

- For the Policies requiring an order of deployment, an App-of-Applications has been configured, this way we can configure sync waves to control the order each resource is created and configured. For example the Keycloak CR has a requirement for the router certificate to be created by Cert-Manager operator. All operators have been created under the app-of-applications, though not all operator have the requirement to be created in a specific order.

Repository structure example:
- boostrap/app directory: contains the ApplicationSet manifests
- boostrap/clustergroups directory: contains the MCE .....
- policies/*: Contains the Polcies that will be enforced, each child directory is one policy.

```
.
├── bootstrap
│   ├── app
│   │   └── 40-applicationset-governance.yaml
│   └── clustergroups
│       ├── 00-namespace.yaml
│       ├── 10-rbac.yaml
│       ├── 30-mce-mceprod.yaml
│       ├── 31-mce-mcedev.yaml
│       ├── 40-mc-mcprod.yaml
│       ├── 41-mc-mcdev.yaml
│       ├── 50-mcsb-mceprod.yaml
│       └── 51-mcsb-mcedev.yaml
...
...
├── policies
│   ├── compliance-operator
│   │   ├── kustomization.yaml
│   │   ├── manifests
│   │   │   ├── 00-namespace.yaml
│   │   │   ├── 10-operatorgroup.yaml
│   │   │   └── 20-subscription.yaml
│   │   └── policy-generator.yaml
│   ├── create-namespace-mynamespace
│   │   ├── kustomization.yaml
│   │   ├── manifests
│   │   │   └── namespace.yaml
│   │   └── policy-generator.yaml
```

The following diagram provides a high-level overview of the components involved, from the creation of the ApplicationSet to the enforcement of policies:

![Diagram flow: creation of the manifests](images/diagram_flow.jpg "creation of the manifests")

For more details on ACM policy architecture, see the community [documentation](https://open-cluster-management.io/docs/getting-started/integration/policy-controllers/policy-framework/).


# Prerequisites

Before using this repository, make sure you have:
- A running OpenShift cluster with Red Hat ACM installed.
- Two or more spoke clusters already imported into ACM. This repo expects that one cluster is named prod-cluster and other named dev-cluster, besides the local-cluster/ACM HUB cluster

# LAB Architecture 

The enviremont has 3 clusters, with the following naming: 
- local-cluster: this is the ACM HUB cluster (cluster where ACM is installed)
- prod-cluster: spoke cluster. For placement porpuses will be labeled with environment: prod
- dev-cluster: spoke cluster. For placement porpuses will be labeled with environment: dev

# How to use this Repository

## Summary of Configuration:
??1. Configure ArgoCD instance to use the PolicyGenerator plugin.
??1. Create the manifests of the bootstrap

??1. Label the target managed clusters with `environment=prod` (or adjust in policy-generator.yaml).

## Part1: Detailed Configuration

1. Login to ACM HUB cluster

    ```bash
    oc login -u <user> -p <password> <API_ENDPOINT>
    ```

2. Clone Git
    
    ```
    git clone https://github.com/luisevm/acm-policies-gitops.git
    ```

3. Install Openshift-Gitops in ACM HUB cluster

    ```bash
    oc create -f bootstrap/gitops/00-namespace.yaml
    oc create -f bootstrap/gitops/10-operatorgroup.yaml
    oc create -f bootstrap/gitops/20-subscription.yaml
    ```
    #Check that the installation was successful
    ```bash
    oc -n openshift-gitops-operator get csv
    ```

4. Give RBAC to allow the user you login to OpenShift or ArgoCD, to see in ArgoCD the applications created in ACM HUB OpenShift cluster. Replace the user "Admin" with your user.

    ```bash
    oc create -f - <<EOF
    apiVersion: user.openshift.io/v1
    kind: Group
    metadata:
      name: cluster-admins
    users:
    - admin
    EOF
    ```

5. Configure ArgoCD instance to use the PolicyGenerator plugin.

    Documentation reference link: [Integrating the Policy Generator with OpenShift GitOps](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/gitops/gitops-overview#integrate-pol-gen-ocp-gitops) and [chapter](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.13/html/gitops/gitops-overview#gitops-policy-definitions).

    a. Find the imageContainer version for your ACM version:
    - Open https://catalog.redhat.com
    - Search by image multicluster-operators-subscription
    - Check the image versions available and select the image name that match your ACM version, in my case ACM version is 2.14 and the correspondent image is registry.redhat.io/rhacm2/multicluster-operators-subscription-rhel9:2.14.0-1752502331.

    b. Patch the ArgoCD adding the following configuration to the existing ArgoCD manifest:
    - Edit ArgoCD instance - in my case Im using the instance running in openshift-gitops namespace
        ```bash
        oc -n openshift-gitops edit argocd openshift-gitops
        ```
    - Patch ArgoCD

        ```
        apiVersion: argoproj.io/v1beta1
        kind: ArgoCD
        metadata:
          name: openshift-gitops
          namespace: openshift-gitops
        spec:
          kustomizeBuildOptions: --enable-alpha-plugins
          repo:
            env:
            - name: KUSTOMIZE_PLUGIN_HOME
              value: /etc/kustomize/plugin
            initContainers:
            - args:
              - -c
              - cp /policy-generator/PolicyGenerator-not-fips-compliant /policy-generator-tmp/PolicyGenerator
              command:
              - /bin/bash
              image: registry.redhat.io/rhacm2/multicluster-operators-subscription-rhel9:2.14.0-1752502331
              name: policy-generator-install
              volumeMounts:
              - mountPath: /policy-generator-tmp
                name: policy-generator
            volumeMounts:
            - mountPath: /etc/kustomize/plugin/policy.open-cluster-management.io/v1/policygenerator
              name: policy-generator
            volumes:
            - emptyDir: {}
              name: policy-generator
        ```

    c. Check that the ArgoCD instance restarts and that is goes running again

    ```
    oc -n openshift-gitops get pods
    ```

6. Bootstrap required Objects

    a. Create in ACM HUB the namespace where the Policyes will be saved 

    ```
    oc create -f bootstrap/clustergroups/00-namespace.yaml
    ```

    b.Configure the RBAC

    ```
    oc create -f bootstrap/clustergroups/10-rbac.yaml
    ```

    c.

    ```
    oc create -f bootstrap/clustergroups/30-mce-mceprod.yaml
    ```

    d.
    ```
    oc create -f bootstrap/clustergroups/31-mce-mcedev.yaml
    ```

    e.
    ```
    oc create -f bootstrap/clustergroups/40-mc-mcprod.yaml
    oc label managedcluster prod-cluster cluster.open-cluster-management.io/clusterset=mceprod --overwrite
    oc label ManagedCluster prod-cluster environment=prod
    ```

    f.
    ```
    oc create -f bootstrap/clustergroups/41-mc-mcdev.yaml 
    oc label managedcluster dev-cluster cluster.open-cluster-management.io/clusterset=mcedev --overwrite
    oc label ManagedCluster dev-cluster environment=dev
    ```

    g.
    ```
    oc create -f bootstrap/clustergroups/50-mcsb-mceprod.yaml 
    ```

    h.
    ```
    oc create -f bootstrap/clustergroups/51-mcsb-mcedev.yaml
    ```

7. Create ApplicationSet

    a.
    ```
    oc create -f bootstrap/app/40-applicationset-spoke-config.yaml
    ```

    b. Check that the AplicationSet was created that it created one Application per each Policy

    ```
    oc -n openshift-gitops get applications.argoproj.io
    ```

## Part2: Detailed Configuration - Setup Sealed Secrets

Sealed Secrets team has developed a Helm Chart for installing the solution automatically. This automatism is customizable with multiple variables depending on the client requirements.

It is important to bear in mind that The kubeseal utility uses asymmetric crypto to encrypt secrets that only the controller can decrypt. Please visit the following [link](https://github.com/bitnami-labs/sealed-secrets/blob/main/docs/developer/crypto.md) for more information about security protocols and cryptographic tools used.

In the following process, a Sealed Secrets controller will be installed using a custom certificate that was generated in the respective namespace previously. This installation model is designed for multi-cluster environments where it is required to use the same certificate in multiple Kubernetes clusters in order to facilitate operations and maintainability.

- Create Namespace where Sealed Secrets Controller will be deployed

```$bash
oc new-project sealedsecrets
```

- Assign permissions to the default service account in order to be able to deploy the respective controller pod

```$bash
oc adm policy add-scc-to-user anyuid -z sealed-secrets -n sealedsecrets
```

- Generate the respective certificates and create a secret

```$bash
sh bootstrap/sealed-secrets/createcertificate/generate-cert.sh
```

- Deploy Sealed Secrets using the respective Helm Chart and the secret generated by the previous script execution

```$bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets -n sealedsecrets --set-string secretName=cert-encryption sealed-secrets/sealed-secrets
```

- Install the command line tool kubeseal
https://github.com/bitnami-labs/sealed-secrets/releases

```$bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.2/kubeseal-0.24.2-linux-amd64.tar.gz -O kubeseal.tar.gz 
tar xzf kubeseal.tar.gz
rm kubeseal.tar.gz
chmod 755 kubeseal 
sudo mv kubeseal /usr/local/bin/
```

- To authorize certain calls to the bitnami sealed secrets API Group from the projects. To configure the final namespace to host the respective *SealedSecret* objects and the respective ArgoCD application that handles the creation of the secret, configure RBAC:

```$bash
oc apply -f bootstrap/sealed-secrets/manifests/ClusterRole_namespaceauth.yaml

oc apply -f bootstrap/sealed-secrets/manifests/rolebinding-sealedsecret.yaml
```

- Create SealedSecret Object - to be used by cert-manager: This object will contain the encrypted key used to access CloudFlare APU authentication key.

    - Export Vars

         ```$bash
         export CLOUDFLARE_API_TOKEN=?????
         ```

    - Locally create a secret for Kubernetes to use to access CloudFlare and label it.

        ```$bash
        CLOUDFLARE_API_TOKEN_B64=`echo -n ${CLOUDFLARE_API_TOKEN} | base64`
        ```

        ```$bash
        cat <<EOF > secret_azv.yaml
        apiVersion: v1
        data:
          api-token: ${CLOUDFLARE_API_TOKEN_B64}
        kind: Secret
        metadata:
          labels:
            secrets-store.csi.k8s.io/used: "true"  
          name: secrets-store-cmcreds
          namespace: ${NAMESPACE}
        EOF
        ```

    - locally create the K8s manifest of the Sealed Secret wich embeds the encryption of the secret to have access to the AZV.

        ```$bash
        kubeseal -f secret_azv.yaml -n ${NAMESPACE} --name secrets-store-cmcreds \
         --controller-namespace=sealedsecrets \
         --controller-name=sealed-secrets \
         --format yaml > bootstrap/sealed-secrets/secrets-store-cmcreds.yaml
         ```

    > **NOTE**
    > controller-namespace: define the namespace where the operator is installed, 
    > controller-name: is a combination of the SealedSecretController object name and the name of the namespace

    - Create namespace and add label to allow GitOps to manage the namespace

        ```$bash
        oc new-project ${NAMESPACE}
        ```

        ```$bash
        oc label namespace ${NAMESPACE} argocd.argoproj.io/managed-by=openshift-gitops
        ```

- Before creating the application it is necessary to make a commit and push to the forked repository. 

```$bash
git add *
git commit -m "argocd sealed secrets"
git push
```








## Part3: Detailed Configuration - Deploy App of Applications
???????????


# Troubleshoot
Example to troubleshoot the Policy to audit the presence of the OpenShift-Gitops operator.

1. On the ACM HUB cluster

    ```bash
    oc -n acm-policy get policy
    oc -n acm-policy describe policy policy-audit-gitops-operator
    ```

    ```bash
    oc -n acm-policies get policy,placement,placementbinding
    ```

    #Verify that ApplicationSet was deployed
    ```bash
    oc -n openshift-gitops describe applicationset 
    ```

    #Open ArgoCD UI in a browser and verify all the Applications deployed

    #Verify Applications are created for each policy:

    ```bash
    oc -n openshift-gitops get applications.argoproj.io
    ```

    ```bash
    oc -n acm-policy get placement
    oc -n acm-policy describe placement gitops-targets
    ```

    ```bash
    oc -n acm-policy get placementdecision
    ```

    ```bash
    oc -n acm-policy describe policy <your-policy-name>
    oc -n prod-cluster get policy
    ```

2. On the spoke cluster

    #Verify Policy was propagated to the spoke cluster
    ```bash
    oc -n prod-cluster get policy
    oc -n prod-cluster describe policy acm-policies.policy-audit-gitops-operator
    ```

    #Look at policy-controller logs on the spoke
    ```bash
    oc -n open-cluster-management-agent-addon get pods | grep governance-policy-framework
    oc -n open-cluster-management-agent-addon logs <policy-framework-pod>
    ```


NOTE: For a ROSA or OSD cluster this procedure doesnt work and would require some tweeking, for example importing the LetsEncript CA certificate have to be done usind the `ocm` cli command or via the OpenShift console UI (https://console.redhat.com/openshift)




Additional notes https://blog.stderr.at/openshift/2023/03/operator-installation-with-argo-cd/

