# Kyma Runtime end-to-end provisioning test

## Overview

Kyma Runtime end-to-end provisioning test checks if [Runtime provisioning](https://github.com/kyma-project/control-plane/blob/main/docs/kyma-environment-broker/02-01-architecture.md) works as expected. The test is based on the Kyma Environment Broker (KEB), Runtime Provisioner and Director implementation. External dependencies relevant for this scenario are mocked.

The test is executed on a dev cluster. It is executed after every merge to the `kyma` repository that changes the `compass` chart.

## Prerequisites

To run this test, you must have the following Secrets inside your cluster:
- Gardener Secret per provider
- Service Manager Secret

You must also have the Kyma Environment Broker [configured](https://github.com/kyma-project/control-plane/tree/main/components/kyma-environment-broker#configuration) to use these Secrets in order to successfully create a Runtime.

## Details

### End-to-end provisioning test
The provisioning end-to-end test contains a broker client implementation which mocks Registry. It is an external dependency that calls the broker in the regular scenario. The test is divided into two phases:

1. Provisioning

    During the provisioning phase, the test executes the following steps:

    a. Sends a call to KEB to provision a Runtime. KEB creates an operation and sends a request to the Runtime Provisioner. The test waits until the operation is successful. It takes about 30 minutes on GCP and a few hours on Azure. You can configure the timeout using the environment variable.

    b. Creates a ConfigMap with **instanceId** specified.

    c. Fetches the DashboardURL from KEB. To do so, the Runtime must be successfully provisioned and registered in the Director.

    d. Updates the ConfigMap with **dashboardUrl** field.

    e. Creates a Secret with a kubeconfig of the provisioned Runtime.

    f. Ensures that the DashboardURL redirects to the UUA login page. It means that the Kyma Runtime is accessible.

2. Cleanup

    The cleanup logic is executed at the end of the end-to-end test or when the provisioning phase fails. During this phase, the test executes the following steps:

    a. Gets **instanceId** from the ConfigMap.

    b. Removes the test's Secret and ConfigMap.

    c. Fetches the Runtime kubeconfig from the Runtime Provisioner and uses it to clean resources which block the cluster from deprovisioning.

    d. Sends a request to deprovision the Runtime to KEB. The request is passed to the Runtime Provisioner which deprovisions the Runtime.

    e. Waits until the deprovisioning is successful. It takes about 20 minutes to complete. You can configure the timeout using the environment variable.

Between the end-to-end test phases, you can execute your own test directly on the provisioned Runtime. To do so, use a kubeconfig stored in a Secret created in the provisioning phase.

### End-to-end suspension test

The end-to-end suspension test uses the **Trial** Service Plan ID to provision Kyma Runtime. Then, the test suspends and unsuspends the Kyma Runtime and ensures that it is still accessible after the process. The suspension test works similarly to the provisioning test, but it has two additional steps in the `Provisioning` phase:

1. Suspension

    After successfully provisioning a Kyma Runtime, the test sends an update call to KEB to suspend the Runtime. Then, the test waits until the operation is successful.


1. Unsuspension

   After Runtime suspension succeeded, the test sends an update call to KEB to unsuspend the Runtime. Then, the test waits until the operation is successful. After that, the test ensures that the DashboardURL redirects to the UUA login page once again, which means that the Kyma Runtime is accessible.

After successful suspension and unsuspension of the Kyma Runtime, the test proceeds to the `Cleanup` phase.

## Configuration

You can configure the test execution by using the following environment variables:

| Name | Description | Default value |
|-----|---------|:--------:|
| **APP_BROKER_URL** | Specifies the KEB URL. | None |
| **APP_PROVISION_TIMEOUT** | Specifies a timeout for the provisioning operation to succeed. | `3h` |
| **APP_DEPROVISION_TIMEOUT** | Specifies a timeout for the deprovisioning operation to succeed. | `1h` |
| **APP_BROKER_PROVISION_GCP** | Specifies if a Runtime cluster is hosted on GCP. If set to `false`, it provisions on Azure. | `true` |
| **APP_BROKER_AUTH_USERNAME** | Specifies the username for the basic authentication in KEB. | `broker` |
| **APP_BROKER_AUTH_PASSWORD** | Specifies the password for the basic authentication in KEB. | None |
| **APP_RUNTIME_PROVISIONER_URL** | Specifies the Provisioner URL. | None |
| **APP_RUNTIME_UUA_INSTANCE_NAME** | Specifies the name of the UUA instance which is provisioned in the Runtime. | `uua-issuer` |
| **APP_RUNTIME_UUA_INSTANCE_NAMESPACE** | Specifies the Namespace of the UUA instance which is provisioned in the Runtime. | `kyma-system` |
| **APP_TENANT_ID** | Specifies TenantID which is used in the test. | None |
| **APP_DIRECTOR_URL** | Specifies the Director URL. | `http://compass-director.compass-system.svc.cluster.local:3000/graphql` |
| **APP_DIRECTOR_OAUTH_TOKEN_URL** | Specifies the URL for OAuth authentication. | None |
| **APP_DIRECTOR_OAUTH_CLIENT_ID** | Specifies the client ID for OAuth authentication. | None |
| **APP_DIRECTOR_OAUTH_SECRET** | Specifies the client secret for OAuth authentication. | None |
| **APP_DIRECTOR_OAUTH_SCOPE** | Specifies the scopes for OAuth authentication. | `runtime:read runtime:write` |
| **APP_DUMMY_TEST** | Specifies if test should success without any action. | `false` |
| **APP_CLEANUP_PHASE** | Specifies if the test executes the cleanup phase. | `false` |
| **APP_CONFIG_NAME** | Specifies the name of the ConfigMap and Secret created in the test. | `e2e-runtime-config` |
| **APP_DEPLOY_NAMESPACE** | Specifies the Namespace of the ConfigMap and Secret created in the test. | `kcp-system` |
| **APP_BUSOLA_URL** | Specifies the URL to the expected Kyma Dashboard used when asserting redirection to the UI Console.  | `kcp-system` |
