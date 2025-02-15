The [Operator Lifecycle Manager (OLM)][olm] provides a declarative way to install and upgrade operators and their
dependencies.

You can install the Starboard operator from [OperatorHub.io](https://operatorhub.io/operator/starboard-operator)
or [ArtifactHUB](https://artifacthub.io/) by creating the OperatorGroup, which defines the operator's
multitenancy, and Subscription that links everything together to run the operator's pod.

1. Install the Operator Lifecycle Manager:

        curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/0.16.1/install.sh | bash -s 0.16.1

2. Create the namespace to install the operator in:

        kubectl create ns starboard-operator

3. Declare the target namespaces by creating the OperatorGroup:


        cat << EOF | kubectl apply -f -
        apiVersion: operators.coreos.com/v1alpha2
        kind: OperatorGroup
        metadata:
          name: starboard-operator
          namespace: starboard-operator
        spec:
          targetNamespaces:
          - default
        EOF

4. (Optional) Configure Starboard by creating the `starboard` ConfigMap and the `starboard` secret in
   the `starboard-operator` namespace. For example, you can use Trivy
   in [ClientServer](./../../integrations/vulnerability-scanners/trivy.md#clientserver) mode or
   [Aqua Enterprise](./../../integrations/vulnerability-scanners/aqua-enterprise.md) as an active vulnerability scanner.
   If you skip this step, the operator will ensure [configuration objects](./../../settings.md)
   on startup with the default settings.

        kubectl apply -f https://raw.githubusercontent.com/aquasecurity/starboard/main/deploy/static/05-starboard-operator.config.yaml
   Review the default values and makes sure the operator is configured properly:

        kubectl describe cm starboard -n starboard-operator
        kubectl describe secret starboard -n starboard-operator

5. Install the operator by creating the Subscription:

        cat << EOF | kubectl apply -f -
        apiVersion: operators.coreos.com/v1alpha1
        kind: Subscription
        metadata:
          name: starboard-operator
          namespace: starboard-operator
        spec:
          channel: alpha
          name: starboard-operator
          source: operatorhubio-catalog
          sourceNamespace: olm
          config:
            env:
            - name: OPERATOR_SCAN_JOB_TIMEOUT
              value: "60s"
            - name: OPERATOR_CONCURRENT_SCAN_JOBS_LIMIT
              value: "10"
            - name: OPERATOR_LOG_DEV_MODE
              value: "true"
        EOF
   The operator will be installed in the `starboard-operator` namespace and will be usable from the `default` namespace.
   Note that the `spec.config` property allows you to override the default [configuration](./../configuration.md) of
   the operator's Deployment.

6. After install, watch the operator come up using the following command:

        kubectl get clusterserviceversions -n starboard-operator
        NAME                        DISPLAY              VERSION   REPLACES                    PHASE
        starboard-operator.v0.8.0   Starboard Operator   0.8.0     starboard-operator.v0.7.0   Succeeded
   If the above command succeeds and the ClusterServiceVersion has transitioned from `Installing` to `Succeeded` phase
   you will also find the operator's Deployment in the same namespace where the Subscription is:

        kubectl get deployments -n starboard-operator
        NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
        starboard-operator   1/1     1            1           11m
   If for some reason it's not ready yet, check the logs of the Deployment for errors:

        kubectl logs deployment/starboard-operator -n starboard-operator

## Uninstall

To uninstall the operator delete the Subscription, the ClusterServiceVersion, and the OperatorGroup:

    kubectl delete subscription starboard-operator -n starboard-operator
    kubectl delete clusterserviceversion starboard-operator.v0.8.0 -n starboard-operator
    kubectl delete operatorgroup starboard-operator -n starboard-operator

[olm]: https://github.com/operator-framework/operator-lifecycle-manager/
