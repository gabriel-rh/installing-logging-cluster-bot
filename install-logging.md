# Intalling logging on ClusterBot 

## Prerequisites

-   You should have the OC CLI installed

## Create cluster

In the Cluster Bot app:

```
launch 4.16.10 aws,no-spot
```

This takes about 50 minutes and returns the URL and login details:

```
Your cluster is ready, it will be shut down automatically in ~135 minutes.
https://console-openshift-console.apps.ci-ln-hw2p3kk-76ef8.aws-2.ci.openshift.org

Log in to the console with user kubeadmin and password astwp-WUaaS-BnLF8-rU8vw
```

It also supplies the `kubconfig` file if you want to use that. Here we will login via the web browser with the username and password.


By default, cluster bot creates a cluster with 6 nodes: 3 control plane (masters), 3 workers. In the Admin perspective, navigate to `Compute` -> `NOdes`:

![Default cluster nodes](images/cluster-nodes-default.png)

We need to extend the cluster to install logging,and we also need some more powerful nodes.

## Add new nodes to cluster

-   View the default machine sets:

    ![Default machine sets](images/machine-sets/01-machine-sets-default.png)

-   Click on the actions button on the first machine set and choose edit machine count:

    ![Edit machine count](images/machine-sets/02-machine-set1-edit-machine-count.png)

-   Increase count from two to six:

    ![Increase machine count to six](images/machine-sets/03-machine-set1-edit-machine-count-6.png)

-   The machine set increases from two to six in summary view:

    ![Machine set one machine count increrases to 6](images/machine-sets/04-machine-set1-2-to-6.png)

-   Click on machine set one, and select the Machines tab, to see new machines provisioned:

    ![Machine set one shows new machines provisioned](images/machine-sets/05-machine-set1-details-machines.png)

-   Wait until the new machines are provisioned as nodes:

    ![Machine set one shows new machines provisioned as nodes](images/machine-sets/06-machine-set1-provisioned-as-nodes.png)


## Add bigger nodes

-   The machine set summary shows the default instance type for the machines - `m6a.xlarge`:

    ![Default machine sets](images/machine-sets/01-machine-sets-default.png)

-   On the machine set summary, click on the second machine set and access the YAML tab:

    ![Machine set 2 shows YAML](images/machine-sets/10-machine-set2-details-yaml.png)

-   Scroll down to the `spec` section to see the `instanceType`:

    ```lang=yaml
        spec:
        lifecycleHooks: {}
        metadata: {}
        providerSpec:
            value:
            userDataSecret:
                name: worker-user-data
            placement:
                availabilityZone: us-east-1f
                region: us-east-1
            credentialsSecret:
                name: aws-cloud-credentials
            instanceType: m6a.xlarge
            metadata:
                creationTimestamp: null
    ```
-   Change the `instanceType: m6a.xlarge` to `instanceType: m6a.4xlarge` and press `Save`.

-   View the machine sets overivew and observe that the instance type for machine set two has changed. 

    ![View new instance type in machine set overview](images/machine-sets/11-machine-sets-larger-instance-type.png)

-   Expand the second machine set by editing the machine count:

    ![Edit machine count for machine set 2](images/machine-sets/12-machine-set2-edit-machine-count.png)

-   Increase the number of machines from one to four

    ![Increase the number of machines from one to four](images/machine-sets/13-machine-set2-1-to-4.png)

-   Click on machine set two, and select the Machines tab, to see new machines provisioned:

    ![Machine set two shows new machines provisioned](images/machine-sets/14-machine-set2-details-machine.png)

-   Wait until the new machines are provisioned as nodes:

    ![Machine set two shows new machines provisioned as nodes](images/machine-sets/15-machine-set2-provisioned-as-nodes.png)


## Install the Loki Operator

-   Go to the Operator Hub in the web console and search for "Loki"

    ![Operator Hub](images/operator-loki/operator-hub-install-loki.png)

-   Install the Red Hat Loki Operator, selecting version 6.0

    ![Installing Loki Operator version 6.0](images/operator-loki/install-loki-operator-60.png)

- Select `Enable Operator recommended cluster monitoring on this namespace.`

    ![Enable Operator recommended cluster monitoring](images/operator-loki/loki-enable-cluster-monitoring.png)

- Click `Install` and wait until it completes:

    ![Installing Loki Operator](images/operator-loki/installing-loki.png)


## Install the Red Hat OpenShift Logging Operator

-   Go to `Operators` -> `Operator Hub` in the web console and search for "openshift logging"

    ![Operator Hub](images/operator-logging/operator-hub-install-logging.png)

-   Install the Red Hat OpenShift Logging Operator, selecting version 6.0

    ![Installing Red Hat OpenShift Logging Operator version 6.0](images/operator-logging/install-logging-operator-60.png)

-   Select:

    -   `A specific namespace on the cluster is selected under Installation Mode.`
    -   `Enable Operator recommended cluster monitoring on this namespace.`

    ![Select specific namespace and enable Operator recommended cluster monitoring](images/operator-logging/logging-specific-namespace-enable-cluster-monitoring.png)  

- Click `Install` and wait until it completes. Navigate to `Operators` -> `Installed Operators`:

    ![Installing Red Hat OpenShift Logging Operator](images/operator-logging/installed-operators.png)



## Log in to cluster from command line

-   In the top right-hand corner of the web console UI, click on `kube:admin` and select `Copy login command` to retrieve the login command for the OC CLI:

    ![Web console UI](images/web-console-login-command.png)

-   Click `Display token` on the following screen and the login command will appear:

    ```
    oc login --token=sha256~37D0xSwYlcqD5Pf8nV1m9gl4zzTh3iVBZbSQyGbHqvQ --server=https://api.ci-ln-hw2p3kk-76ef8.aws-2.ci.openshift.org:6443
    ```


-   Run the command in your terminal:

    ```lang=shell {title="Login to cluster"}
    $ oc login --token=sha256~37D0xSwYlcqD5Pf8nV1m9gl4zzTh3iVBZbSQyGbHqvQ --server=https://api.ci-ln-hw2p3kk-76ef8.aws-2.ci.openshift.org:6443
    ```

-   Accept the request to use incecure connections:

    ```
    The server uses a certificate signed by an unknown authority.
    You can bypass the certificate check, but any data you send to the server could be intercepted by others.
    Use insecure connections? (y/n): y

    WARNING: Using insecure TLS client config. Setting this option is not supported!

    Logged into "https://api.ci-ln-hw2p3kk-76ef8.aws-2.ci.openshift.org:6443" as "kube:admin" using the token provided.

    You have access to 70 projects, the list has been suppressed. You can list all projects with 'oc projects'

    Using project "default".
    ```

-   Set your project to `openshift-logging`.

    ```
    $ oc project openshift-logging
    ```


## Configure S3 storage 

- Default storage classes on Cluster Bot AWS

    ![Default storage classes on Cluster Bot AWS](images/storage-classes.png)


-   Retrieve the credentials for the storage:

    ```
    $ oc get secret -n openshift-cluster-csi-drivers ebs-cloud-credentials -o yaml
    ```

- The output shows the `aws_access_key_id` and the `aws_secret_access_key`, encoded in base64:

    ```
    apiVersion: v1
    data:
      aws_access_key_id: QUtJQTQ3T05SVTU1RzZRM0ZRTTY=
      aws_secret_access_key: MFB0Tmw1VVdBVTNHZEVEMlIvRTZaUmJuMXFmaWJuUVZlaG55Q1lhVQ==
      credentials: W2RlZmF1bHRdCmF3c19hY2Nlc3Nfa2V5X2lkID0gQUtJQTQ3T05SVTU1RzZRM0ZRTTYKYXdzX3NlY3JldF9hY2Nlc3Nfa2V5ID0gMFB0Tmw1VVdBVTNHZEVEMlIvRTZaUmJuMXFmaWJuUVZlaG55Q1lhVQ==
    kind: Secret
    metadata:
      annotations:
    ...
    ```

-   Decode the access key:

    ```
    $ echo "QUtJQTQ3T05SVTU1RzZRM0ZRTTY=" | base64 -d
    ```
-   The output should look like:

    ```
    AKIA47ONRU55G6Q3FQM6
    ```

- Decode the secret access key

    ```
    $ echo "MFB0Tmw1VVdBVTNHZEVEMlIvRTZaUmJuMXFmaWJuUVZlaG55Q1lhVQ==" | base64 -d
    ```

-   The output looks like:

    ```
    0PtNl5UWAU3GdED2R/E6ZRbn1qfibnQVehnyCYaU
    ```

-   Create a secret in the `openshift-logging` namespace. In the example below, the secret is named `test-bucket`, and requires:

    -   The decoded access key
    -   The decoded access key secret
    -   An arbitrary bucket name
    -   An endpoint in AWS
    -   A region in AWS

    ```
    $ oc create secret generic test-bucket --from-literal=access_key_id=AKIA47ONRU55G6Q3FQM6  --from-literal=access_key_secret=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 --from-literal=bucketnames=gabrielsbucket --from-literal=endpoint=https://s3.eu-west-3.amazonaws.com --from-literal=region=eu-west-3 --namespace=openshift-logging
    ```

## Create LokiStack using S3 bucket

-   Set your default project in the web console to `openshift-logging`

    ![Set your default project to openshift-logging](images/lokistack/set-project-openshift-logging.png)

-   Go to Installed Operators

    ![Installed Operators](images/lokistack/installed-operators-openshift-logging.png)

-   Navigate to `Loki Operator` -> `LokiStack` and click `Create LokiStack`

    ![Create LokiStack](images/lokistack/loki-operator-create-lokistack.png)

-   Navigate to the Create LokiStack YAML view and see the default YAML:

    ![LokiStack YAML default](images/lokistack/create-lokistack-yaml-default.png)

-   Override the default YAML:

    ```
    apiVersion: loki.grafana.com/v1
    kind: LokiStack
    metadata:
      name: logging-loki
      namespace: openshift-logging
    spec:
      managementState: Managed
      rules:
        enabled: true
        namespaceSelector:
          matchLabels:
            openshift.io/log-alerting: 'true'
        selector:
          matchLabels:
            openshift.io/log-alerting: 'true'
      size: 1x.extra-small
      storage:
        schemas:
        - version: v13
          effectiveDate: "2024-03-01"
        secret:
          name: test-bucket
          type: s3
      storageClassName: gp2-csi
      tenants:
        mode: openshift-logging
    ```

    1.   `storageClassName: gp2-csi`
    1.   `name: test-bucket`
    1.   `size: 1x.extra-small`

    

-   Click `Create`:

    ![Create LokiStack using YAML](images/lokistack/create-lokistack-yaml.png)

-   View the LokiStack instance, initially in the `pending` state:

    ![LokiStack instance pending](images/lokistack/lokistack-instance-pending.png)

-   View the LokiStack details

    ![LokiStack instance details](images/lokistack/lokistack-details.png)

-   View the LokiStack resources being created

    ![LokiStack instance details](images/lokistack/lokistack-details-resources.png)

-   Inspect the LokiStack `conditions` in the YAML view, and wait until all components are ready. If you see any errors here, you need to fix them.

    ![LokiStack instance conditions in YAML](images/lokistack/lokistack-conditions-yaml.png)

-   The LokiStack instance should eventually show the status as `Ready`:

    ![LokiStack instance ready](images/lokistack/lokistack-instance.png)

## Create service account and cluster roles for log writer

-   Create a service account

    ```
    $ oc create sa collector -n openshift-logging
    ```

-   Create `cluster-role.yaml`:

    ```lang=yaml {title="cluster-role.yaml"}
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: logging-collector-logs-writer
      namespace: openshift-logging
    rules:
    - apiGroups:
      - loki.grafana.com
      resourceNames:
      - logs
      resources:
      - application
      - audit
      - infrastructure
      verbs:
      - create
    ```

-   Create cluster role named `logging-collector-logs-writer`:

    ```
    $ oc create -f cluster-role.yaml 
    ```

-   Ignore the error if it already exists:

    ```
    Error from server (AlreadyExists): error when creating "cluster-role.yaml": clusterroles.rbac.authorization.k8s.io "logging-collector-logs-writer" already exists
    ```

-   Add cluster role to service account

    ```
    $ oc adm policy add-cluster-role-to-user logging-collector-logs-writer -z collector -n openshift-logging
    ```

## Install COO and the logging UIPlugin

-   Locate the COO Operator in the Operator Hub

    ![COO in Operator Hub](images/coo/operator-hub-coo.png)

-   Install the latest version of COO

    ![Install the latest version of COO](images/coo/coo-install-041.png)

-   Wait while the operator is installed:

    ![Installing COO](images/coo/coo-installing.png)

-   When the Operator is ready for use, click `View Operator`

    ![COO ready for use](images/coo/coo-installed-ready-for-use.png)

-   There are a lot of tabs on the COO page - scroll to the right to find the UIPlugin tab:

    ![COO tabs](images/coo/coo-tabs.png)

-   Select the `UIPlugin` tab and click `Create UIPlugin`

    ![Create COO UIPlugin](images/coo/coo-create-uiplugin.png)

-   Select the YAML tab on the `Create UIPlugin` screen:

    ![Create UIPlugin logging YAML](images/coo/create-uiplugin-logging-yaml.png)

-   Paste the appropriate YAML for the logging plugin, and click `Create`:

    ```
    apiVersion: observability.openshift.io/v1alpha1
    kind: UIPlugin
    metadata:
      name: logging
    spec:
      type: Logging
      logging:
        lokiStack:
          name: logging-loki
    ```

-   You are notified that the web console has been updated and that you need to refresh:

    ![Refresh web console](images/coo/uiplugin-logging-refresh-web-console.png)

-   After refreshing, you will see that Logs has been added to the Observe menu list

    ![Observe Logs added](images/coo/observe-logs-added.png)

## Add cluster roles for collection of logs for infrastructure, application and audit:

```
$ oc adm policy add-cluster-role-to-user collect-infrastructure-logs -z collector -n openshift-logging
```

```
$ oc adm policy add-cluster-role-to-user collect-application-logs -z collector -n openshift-logging
```

```
$ oc adm policy add-cluster-role-to-user collect-audit-logs -z collector -n openshift-logging
```

## Create log forwarder

-   Paste the following YAML from the docs into the web console (or alternatively, create a YAML file and apply):

    ```
    apiVersion: observability.openshift.io/v1
    kind: ClusterLogForwarder
    metadata:
      name: collector
      namespace: openshift-logging
    spec:
      serviceAccount:
        name: collector
      outputs:
      - name: default-lokistack
        type: lokiStack
        lokiStack:
          target:
            name: logging-loki
            namespace: openshift-logging
        authentication:
          token:
            from: serviceAccount
        tls:
          ca:
            key: service-ca.crt
            configMapName: openshift-service-ca.crt
      pipelines:
      - name: default-logstore
        inputRefs:
        - application
        - infrastructure
        outputRefs:
        - default-lokistack
    ```

-   The UI will signal an error with the authentication stanza:

    ![Cluster log forwarder YAML error](images/cluster-log-forwarder/cluster-log-forwarder-yaml-wrong.png)

-   Correct the YAML by indenting the authentication stanza:

    ```
    apiVersion: observability.openshift.io/v1
    kind: ClusterLogForwarder
    metadata:
      name: collector
      namespace: openshift-logging
    spec:
      serviceAccount:
        name: collector
      outputs:
      - name: default-lokistack
        type: lokiStack
        lokiStack:
          target:
            name: logging-loki
            namespace: openshift-logging
          authentication:
            token:
              from: serviceAccount
        tls:
          ca:
            key: service-ca.crt
            configMapName: openshift-service-ca.crt
      pipelines:
      - name: default-logstore
        inputRefs:
        - application
        - infrastructure
        outputRefs:
        - default-lokistack
    ```

-   The reported error in the UI is resolved - click `Create`:

    ![Cluster log forwarder YAML fixed](images/cluster-log-forwarder/cluster-log-forwarder-fixed.png)

-   Wait until the log forwarder status is `Ready`:

    ![Cluster log forwarder conditions](images/cluster-log-forwarder/cluster-log-forwarder-conditions.png)

## View collected logs

-   Navigate to `Observe` -> `Logs`:

    ![Observer logs blank](images/observe-logs/observe-logs-blank.png)

-   Choose all the log severities:

    ![Choose all the log severities](images/observe-logs/choose-all-severities.png)

-   Choose to view infrastructure logs:

    ![Choose infrastructure logs](images/observe-logs/choose-infrastructure-logs.png)

-   For convenience, you can set the history to something like `Last 15 minutes` and refresh period to `15 seconds`. Click `Run Query` and the logs should soon appear:

    ![Infrastructure logs output](images/observe-logs/infrastructure-logs-output.png)
