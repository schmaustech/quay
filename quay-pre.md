# Red Hat Quay Operator 3.6 Pre-Release Instructions

## Intended use and support scope

This document describes how to get access to a private beta release of the Red Hat Quay Operator v3.6. The intention is to give users a chance to test this release of the Operator in regards to particular gaps in functionality of previous Operator versions or to upgrade directly from earlier versions of Quay to 3.6. 
The intention is to test this in non-production environments. The Operator and the deployed Quay version are unreleased as of yet and therefore unsupported for production deployments. 

**Do not** use this Operator to manage production data!

**Do not** use these instructions or the Operator on production clusters!

**Do not** try to upgrade this Operator to a later GA release of the Operator!

There is no support SLA provided for this Operator distribution. The goal is to get early feedback about behaviors observed by this particular version of the Operator. Red Hat is not liable for any data loss, outage or service degradation caused by the Operator distribution.

## Known issues

- `mirror` component is changed to `unmanaged` after an configuration update through the config editor ([PROJQUAY-2490](https://issues.redhat.com/browse/PROJQUAY-2490))
- `tls` component is changed to unmanaged after an configuration update through the config editor ([PROJQUAY-2488](https://issues.redhat.com/browse/PROJQUAY-2488))
- reconciliation error when changing the Quay server name and uploading custom certificates through the config editor ([PROJQUAY-2428](https://issues.redhat.com/browse/PROJQUAY-2428))
- multi-attach volume errors with the postgresql database after reconciliation ([PROJQUAY-2436](https://issues.redhat.com/browse/PROJQUAY-2436))
- unable to set `mirror` component to managed after enabling mirroring through config editor when `tls` is unmanaged ([PROJQUAY-2489](https://issues.redhat.com/browse/PROJQUAY-2489))
- Quay notifications not working when `tls` is unamanged ([PROJQUAY-2424](https://issues.redhat.com/browse/PROJQUAY-2424))
- `objectstorage` unexpectedly changed to `managed` after adding OIDC provider through the config editor ([PROJQUAY-2460](https://issues.redhat.com/browse/PROJQUAY-2460))

## Testing the Red Hat Quay Operator 3.6 Pre-Release

### Target cluster preparations

The OpenShift cluster running the pre-release version of the Red Hat Quay 3.6 operator needs to get an additional pull-secret and an image content source policy configured for the deployment to work. All instructions need to be carried out with an account that has the `cluster-admin` role. The cluster needs to be able to reach quay.io.

1. Add following `ImageContentSourcePolicy` to the target cluster

    ```yaml
    ---
    apiVersion: operator.openshift.io/v1alpha1
    kind: ImageContentSourcePolicy
    metadata:
      labels:
        operators.openshift.org/catalog: "true"
      name: quay-operator-catalog-mirror
    spec:
      repositoryDigestMirrors:
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-builder-qemu-rhcos-rhel8
        source: registry.redhat.io/quay/quay-builder-qemu-rhcos-rhel8
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-rhel8
        source: registry.redhat.io/quay/quay-rhel8
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-builder-rhel8
        source: registry.redhat.io/quay/quay-builder-rhel8
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-operator-rhel8
        source: registry.redhat.io/quay/quay-operator-rhel8
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-operator-bundle
        source: registry.stage.redhat.io/quay/quay-operator-bundle
      - mirrors:
        - quay.io:443/quay-operator-mirror/rhel8-redis-5
        source: registry.redhat.io/rhel8/redis-5
      - mirrors:
        - quay.io:443/quay-operator-mirror/rhel8-postgresql-10
        source: registry.redhat.io/rhel8/postgresql-10
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-clair-rhel8
        source: registry.redhat.io/quay/clair-rhel8
      - mirrors:
        - quay.io:443/quay-operator-mirror/rh-osbs-quay-quay-operator-bundle
        source: registry-proxy.engineering.redhat.com/rh-osbs/quay-quay-operator-bundle
      - mirrors:
        - quay.io:443/quay-operator-mirror/quay-quay-rhel8-operator
        source: registry.redhat.io/quay/quay-rhel8-operator
    ```

2. Extract the global pull secret of the target cluster to a file on disk

    `oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > global-pull-secret.json`

3. Add a second set of login credentials to quay.io to be able to pull the images associated with the Quay Operator 3.6 Pre-Release:

    `oc registry login --registry quay.io:443 --auth-basic="quay-operator-mirror+robot:NL52E5UB3L9XZ5TQNNJJ4I76WDNDMEMDKPCOIAQEDRKIOJAOEUE7RBS9XHDZ8OW4" --to=global-pull-secret.json`

4. Update the global pull secret with the new set of credentials:

    `oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=global-pull-secret.json`

### Installing the Red Hat Quay Operator 3.6 Pre-Release in a cluster without a previous version of the operator installed

1. Create a new namespace to test the Operator and Quay

    `oc new-project quay-registry`

2. Add the catalog containing the Red Hat Quay Operator 3.6 Pre-Release

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: quay-operator-catalog
      namespace: quay-registry
    spec:
      sourceType: grpc
      image: quay.io:443/quay-operator-mirror/catalog:v3.6.0-45
      displayName: Quay Operator 3.6 Pre-Release Catalog
      publisher: Red Hat Quay Engineering
    ```

3. Observe the catalog becoming healthy by waiting for `CatalogSource` to transition into the connection state `READY` and the associated pod running healthy:

    `oc describe catsrc quay-operator-catalog`

    `oc get pods --selector olm.catalogSource=quay-operator-catalog`

4. Create an `OperatorGroup` configuration that instructs the operator to watch the current namespace, e.g. `quay-registry`


    ```yaml
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: quay-operator-group
      namespace: quay-registry
    spec:
      targetNamespaces:
        - quay-registry
    ```
5. Create a `Subscription` that triggers the installation of the Red Hat Quay 3.6 Operator Pre-Release from the mirror catalog

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: quay-operator
      namespace: quay-registry
    spec:
      channel: "stable-3.6"
      installPlanApproval: Automatic
      name: quay-operator
      source: quay-operator-catalog
      sourceNamespace: quay-registry
    ```

6. Observe the installation of the operator through the status of the `ClusterServiceVersion` object

    `oc get csv -n quay-registry -w`

7. After 30-60 seconds the `ClusterServiceVersion` should transition into the `Suceeded` state and the operator pod should run in healthy state

    `oc get deployment --selector olm.owner=quay-operator.v3.6.0`

8. At this point the deployment of the operator is complete and you can the pre-release [documentation](https://gabriel-rh.github.io/quay-docs/downstream/deploy_quay_on_openshift_op_tng/html-singler) for the Quay Operator to deploy a Quay registry.

### Updating an existing Quay Operator deployment to the Red Hat Quay Operator 3.6 Pre-Release

If you already have a Quay Operator deployment, you can test the update to the Red Hat Quay Operator 3.6 Pre-Release. Remember, that the Pre-Release is not meant for production deployments, **so don't upgrade your production operator deployment**.

Note that updating existing Quay Operator deployments to the Red Hat Quay Operator 3.6 Pre-Release **will also update all Quay deployments to Red Hat Quay 3.6**.

1. Locate the namespace in which the current Quay Operator is deployed, frequently that is `openshift-operators`

    `oc get subscriptions -A`

    For example would yield:

    ```
    NAMESPACE             NAME            PACKAGE         SOURCE             CHANNEL
    openshift-operators   quay-operator   quay-operator   redhat-operators   quay-v3.5
    openshift-storage     ocs-operator    ocs-operator    redhat-operators   stable-4.8
    ```

2. Ensure the current Operator deployment is healthy before proceeding with the update:

    `oc get csv -n openshift-operators`

    For example should yield:

    ```
    NAME                   DISPLAY        VERSION   REPLACES               PHASE
    quay-operator.v3.5.6   Red Hat Quay   3.5.6     quay-operator.v3.5.5   Succeeded
    ```

3. Add the catalog containing the Red Hat Quay Operator 3.6 Pre-Release **into the same namespace** where the `Subscription` for the current Quay Operator is, e.g.

    ```yaml
    apiVersion: operators.coreos.com/v1alpha1
    kind: CatalogSource
    metadata:
      name: quay-operator-catalog
      namespace: openshift-operators
    spec:
      sourceType: grpc
      image: quay.io:443/quay-operator-mirror/catalog:v3.6.0-30
      displayName: Quay Operator 3.6 Pre-Release Catalog
      publisher: Red Hat Quay Engineering
    ```

4. Patch the existing `Subscription` for the current Quay Operator so that it points to the catalog above and the new channel `stable-3.6`, adjust the name and namespace of the `Subscription`, as well as the name namespace of the `CatalogSource` above to your enviroment:

    `oc patch subscription quay-operator -n openshift-operators --patch '{"spec":{"source":"quay-operator-catalog", "sourceNamespace":"openshift-operators", "channel": "stable-3.6"}}' --type merge`

5. If the existing `Subscription` has the `spec.installPlanApproval` property set to `Automatic` the update will commence automatically. Observe the `ClusterServiceVersion` object in the same namespace reflect the update:

    `oc get csv -n openshift-operators -w`

6. If the existing `Subscription` has the `spec.installPlanApproval` property set to `Manual` you will need to approve the update by approving the `InstallPlan` that is references in the `status` section of the `Subscription`. Set the `spec.approved` property to `true` of the referenced `InstallPlan` object.

7. The update should successfully replace the current Quay Operator with the Red Hat Quay 3.6 Operator Pre-Release:

    ```
    NAME                   DISPLAY        VERSION   REPLACES               PHASE
    quay-operator.v3.6.0   Red Hat Quay   3.6.0     quay-operator.v3.5.6   Succeeded
    ```

8. Existing `QuayRegistry` instances will automatically get updated. Observe the reported `status.currentVersion` property transition to `3.6.0`, for example:

   `oc get quayregistry example -o jsonpath='{.status.currentVersion}'`
   

### Updating to a newer version of the Red Hat Quay Operator 3.6 Pre-Release

Occassionally there will be a new version of the Red Hat Quay Operator 3.6 Pre-Release. It is shipped in the form of a new catalog image, where a higher build suffix depicts a newer version, e.g. `quay.io:443/quay-operator-mirror/catalog:v3.6.0-33` vs. `quay.io:443/quay-operator-mirror/catalog:v3.6.0-36`. The operator version will remain at `3.6.0`. 

To use the new release, please uninstall the old release by following these steps:

1. Delete any deployed `QuayRegistry` instance
2. Delete the `Subscription` and then the `ClusterServiceVersion` object related to the operator
3. Delete the `CatalogSource` instance, serving the previous version of the operator

Follow the instructions "Installing the Red Hat Quay Operator 3.6 Pre-Release in a cluster without a previous version of the operator installed" to start testing the new version.
