---
title: Enable Admission Controller in TiDB Operator
summary: Learn how to enable the admission controller in TiDB Operator and the functionality of the admission controller.
aliases: ['/docs/tidb-in-kubernetes/dev/enable-admission-webhook/']
---

# Enable Admission Controller in TiDB Operator

Kubernetes v1.9 introduces the [dynamic admission control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) to modify and validate resources. TiDB Operator also supports the dynamic admission control to modify, validate, and maintain resources. This document describes how to enable the admission controller and introduces the functionality of the admission controller.

## Prerequisites

Unlike those of most products on Kubernetes, the admission controller of TiDB Operator consists of two mechanisms: [extension API-server](https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server/) and [Webhook Configuration](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#configure-admission-webhooks-on-the-fly).

To use the admission controller, you need to enable the aggregation layer feature of the Kubernetes cluster. The feature is enabled by default. To check whether it is enabled, see [Enable Kubernetes Apiserver flags](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/#enable-kubernetes-apiserver-flags).

## Enable the admission controller

With a default installation, TiDB Operator disables the admission controller. Take the following steps to manually turn it on.

1. Edit the `values.yaml` file in TiDB Operator.

    Enable the `Operator Webhook` feature:

    ```yaml
    admissionWebhook:
      create: true
    ```

2. Configure the failure policy.

    It is recommended to set the [`failurePolicy`](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/#failure-policy) of TiDB Operator to `Failure`. The exception occurs in `admission webhook` does not affect the whole cluster, because the dynamic admission control supports the label-based filtering mechanism.

    ```yaml
    ......
    failurePolicy:
        validation: Fail
        mutation: Fail
    ```

3. Install or update TiDB Operator.

    To install or update TiDB Operator, see [Deploy TiDB Operator on Kubernetes](deploy-tidb-operator.md).

## Set the TLS certificate for the admission controller

By default, the admission controller and Kubernetes api-server skip the [TLS verification](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/#contacting-the-extension-apiserver). To manually enable and configure the TLS verification between the admission controller and Kubernetes api-server, take the following steps:

1. Install `cert-manager`.

    Refer to [cert-manager installation on Kubernetes](https://cert-manager.io/docs/installation/) for details.

2. Create an Issuer to issue certificates to the TiDB operator.

    To configure `cert-manager`, create the Issuer resources.

    First, create a directory which saves the files that `cert-manager` needs to create certificates:

    ```shell
    mkdir -p cert-manager-operator
    cd cert-manager-operator
    ```

    Then, create a `tidb-operator-issuer.yaml` file with the following content:

    ```yaml
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: tidb-operator-selfsigned-ca-issuer
      namespace: ${namespace}
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: tidb-operator-ca
      namespace: ${namespace}
    spec:
      secretName: tidb-operator-ca-secret
      commonName: "TiDB Operator CA"
      isCA: true
      duration: 87600h # 10yrs
      renewBefore: 720h # 30d
      issuerRef:
        name: tidb-operator-selfsigned-ca-issuer
        kind: Issuer
    ---
    apiVersion: cert-manager.io/v1
    kind: Issuer
    metadata:
      name: tidb-operator-issuer
      namespace: ${namespace}
    spec:
      ca:
        secretName: tidb-operator-ca-secret
    ```

    `${namespace}` is the name of the operator, usually `tidb-admin`, this namespace must exist before creating the objects.

    The above YAML file creates three objects:

    - An Issuer object of the SelfSigned type, used to generate the CA certificate needed by Issuer of the CA type;
    - A Certificate object, whose `isCa` is set to `true`.
    - An Issuer, used to issue TLS certificates.

    Finally, execute the following command to create an Issuer:

    ```shell
    kubectl apply -f tidb-operator-issuer.yaml
    ```

    To check if the issuer and certificate are ready:
    ```
    $ kubectl -n tidb-admin get issuer.cert-manager.io,certificate.cert-manager.io -o wide
    NAME                                                        READY   STATUS                AGE
    issuer.cert-manager.io/tidb-operator-issuer                 True    Signing CA verified   118s
    issuer.cert-manager.io/tidb-operator-selfsigned-ca-issuer   True                          118s
    
    NAME                                           READY   SECRET                    ISSUER                               STATUS                                          AGE
    certificate.cert-manager.io/tidb-operator-ca   True    tidb-operator-ca-secret   tidb-operator-selfsigned-ca-issuer   Certificate is up to date and has not expired   118s
```

3. Generate the certificate for the admission controller.

    Create `webhook-server.yaml`:

    ``` yaml
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
        name: webhook-server-secret
        namespace: ${namespace}
    spec:
        secretName: webhook-server-secret
        duration: 8760h # 365d
        renewBefore: 360h # 15d
        subject:
          organizations:
          - PingCAP
        commonName: "TiDB Operator Webhook"
        usages:
          - server auth
          - client auth
        dnsNames:
          - "tidb-admission-webhook.${namespace}"
          - "tidb-admission-webhook.${namespace}.svc"
          - "tidb-admission-webhook.${namespace}.svc.cluster"
          - "tidb-admission-webhook.${namespace}.svc.cluster.local"
        ipAddresses:
          - 127.0.0.1
          - ::1
        issuerRef:
          name: tidb-operator-issuer
          kind: Issuer
          group: cert-manager.io
    ```

    `${namespace}` is the namespace which TiDB Operator is deployed in.

    ```shell
    kubectl apply -f webhook-server.yaml
    ```

    To check if the certificate is ready:

    ```
    $ kubectl -n tidb-admin get certificate.cert-manager.io/webhook-server-secret -o wide
    NAME                    READY   SECRET                  ISSUER                 STATUS                                          AGE
    webhook-server-secret   True    webhook-server-secret   tidb-operator-issuer   Certificate is up to date and has not expired   70s
    ```

    The secret should have 3 files:

    ```
    $ kubectl -n tidb-admin describe secret/webhook-server-secret 
    Name:         webhook-server-secret
    Namespace:    tidb-admin
    Labels:       controller.cert-manager.io/fao=true
    Annotations:  cert-manager.io/alt-names:
                    tidb-admission-webhook.tidb-admin,tidb-admission-webhook.tidb-admin.svc,tidb-admission-webhook.tidb-admin.svc.cluster,tidb-admission-webho...
                  cert-manager.io/certificate-name: webhook-server-secret
                  cert-manager.io/common-name: TiDB Operator Webhook
                  cert-manager.io/ip-sans: 127.0.0.1,::1
                  cert-manager.io/issuer-group: cert-manager.io
                  cert-manager.io/issuer-kind: Issuer
                  cert-manager.io/issuer-name: tidb-operator-issuer
                  cert-manager.io/subject-organizations: PingCAP
                  cert-manager.io/uri-sans: 
    
    Type:  kubernetes.io/tls
    
    Data
    ====
    ca.crt:   1103 bytes
    tls.crt:  1448 bytes
    tls.key:  1679 bytes
```

4. Modify `values.yaml`, and install or upgrade TiDB Operator.

    Get the value of `ca.crt`:

    ```shell
    kubectl get secret webhook-server-secret --namespace=${namespace} -o=jsonpath='{.data.ca\.crt}'
    ```

    Configure the items in `values.yaml` as described below:

    ```yaml
    admissionWebhook:
      apiservice:
        insecureSkipTLSVerify: false # Enable TLS verification
        tlsSecret: "webhook-server-secret" # The name of the secret belonging to the certificate created in Step 3
        caBundle: "<caBundle>" # The value of `ca.crt` obtained in the above step
    ```

    After configuring the items, install or upgrade TiDB Operator. For installation, see [Deploy TiDB Operator](deploy-tidb-operator.md). For upgrade, see [Upgrade TiDB Operator](upgrade-tidb-operator.md).

## Functionality of the admission controller

TiDB Operator implements many functions using the admission controller. This section introduces the admission controller for each resource and its corresponding functions.

* Admission controller for StatefulSet validation

    The admission controller for StatefulSet validation supports the gated launch of the TiDB/TiKV component in a TiDB cluster. The component is disabled by default if the admission controller is enabled.

    ```yaml
    admissionWebhook:
      validation:
        statefulSets: false
    ```

    You can control the gated launch of the TiDB/TiKV component in a TiDB cluster through two annotations, `tidb.pingcap.com/tikv-partition` and `tidb.pingcap.com/tidb-partition`. To set the gated launch of the TiKV component in a TiDB cluster, execute the following commands. The effect of `partition=2` is the same as that of [StatefulSet Partitions](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#partitions).

    ```shell
    kubectl annotate tidbcluster ${name} -n ${namespace} tidb.pingcap.com/tikv-partition=2 &&
    tidbcluster.pingcap.com/${name} annotated
    ```

    Execute the following commands to unset the gated launch:

    ```shell
    kubectl annotate tidbcluster ${name} -n ${namespace} tidb.pingcap.com/tikv-partition- &&
    tidbcluster.pingcap.com/${name} annotated
    ```

    This also applies to the TiDB component.

* Admission controller for TiDB Operator resources validation

    The admission controller for TiDB Operator resources validation supports validating customized resources such as `TidbCluster` and `TidbMonitor` in TiDB Operator. The component is disabled by default if the admission controller is enabled.

    ```yaml
    admissionWebhook:
      validation:
        pingcapResources: false
    ```

    For example, regarding `TidbCluster` resources, the admission controller for TiDB Operator resources validation checks the required fields of the `spec` field. When you create or update `TidbCluster`, if the check is not passed (for example, neither of the `spec.pd.image` filed and the `spec.pd.baseImage` field is defined), this admission controller refuses the request.

* Admission controller for TiDB Operator resources modification

    The admission controller for TiDB Operator resources modification supports filling in the default values of customized resources, such as `TidbCluster` and `TidbMonitor` in TiDB Operator. The component is enabled by default if the admission controller is enabled.

    ```yaml
    admissionWebhook:
      mutation:
        pingcapResources: true
    ```
