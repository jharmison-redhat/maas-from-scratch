# What's been done

- trusted TLS via cert-manager/letsencrypt
- htpasswd oauth configs
- machinesets configured with 1 replicas each for CPU and GPU workers
- NFD and GPU operator are already installed and configured with defaults

# Install prereqs

- cert-manager (RHDP already installed the operator, we just need to add one thing)
  - create a selfsigned issuer for our internal usage
    ```
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: selfsigned-issuer
    spec:
      selfSigned: {}
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: selfsigned-ca
      namespace: cert-manager
    spec:
      commonName: selfsigned-ca
      isCA: true
      issuerRef:
        group: cert-manager.io
        kind: ClusterIssuer
        name: selfsigned-issuer
      privateKey:
        algorithm: ECDSA
        size: 256
      secretName: cert-manager-ca
    ---
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: ca-issuer
    spec:
      ca:
        secretName: cert-manager-ca
    ```
  - configure that selfsigned issuer to be trusted internally to the cluster
    - get the `tls.crt` key from the `cert-manager-ca` secret in the `cert-manager` namespace
    - create a configmap in `openshift-config` named `user-ca-bundle` with a key named `ca-bundle.crt` with the contents
      of the above
    - update the `Proxy` object named `cluster` with `spec.trustedCA.name=user-ca-bundle`
- Leader Worker Set
  - and operand, default is fine
- Cluster Observability Operator
  - pin to 1.4.0
- Red Hat build of OpenTelemetry
- CNPG
  - not required, easier for self-hosted Postgres
  - create a CNPG Cluster
    ```
    apiVersion: v1
    kind: Namespace
    metadata:
      name: maas-db
    ---
    apiVersion: cert-manager.io/v1
    kind: Certificate
    metadata:
      name: openshift-ai-maas
      namespace: maas-db
    spec:
      dnsNames:
        - openshift-ai-maas-rw
        - openshift-ai-maas-rw.maas-db
        - openshift-ai-maas-rw.maas-db.svc
        - openshift-ai-maas-rw.maas-db.svc.cluster.local
        - openshift-ai-maas-ro
        - openshift-ai-maas-ro.maas-db
        - openshift-ai-maas-ro.maas-db.svc
        - openshift-ai-maas-ro.maas-db.svc.cluster.local
        - openshift-ai-maas-r
        - openshift-ai-maas-r.maas-db
        - openshift-ai-maas-r.maas-db.svc
        - openshift-ai-maas-r.maas-db.svc.cluster.local
      issuerRef:
        group: cert-manager.io
        kind: ClusterIssuer
        name: ca-issuer
      secretName: openshift-ai-maas-certificate
      usages:
        - server auth
    ---
    apiVersion: postgresql.cnpg.io/v1
    kind: Cluster
    metadata:
      name: openshift-ai-maas
      namespace: maas-db
    spec:
      bootstrap:
        initdb:
          database: maas
          owner: maas
      certificates:
        serverCASecret: openshift-ai-maas-certificate
        serverTLSSecret: openshift-ai-maas-certificate
      instances: 1
      managed:
        roles:
          - login: true
            name: maas
      resources:
        limits:
          cpu: '1'
          memory: 2Gi
        requests:
          cpu: 200m
          memory: 256Mi
      storage:
        size: 2Gi
        storageClass: gp3-csi
    ```
- enable user-workload-monitoring
  ```
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: cluster-monitoring-config
    namespace: openshift-monitoring
  data:
    config.yaml: |
      enableUserWorkload: true
  ```
  - there's more we could do, such as enabling persistence for cluster monitoring or user workload monitoring. we're
    skipping these for brevity and simplicity.
- create the maas-default-gateway
  ```
  apiVersion: gateway.networking.k8s.io/v1
  kind: GatewayClass
  metadata:
    name: openshift-default
  spec:
    controllerName: "openshift.io/gateway-controller/v1"
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: maas-default-gateway-config
    namespace: openshift-ingress
  data:
    deployment: |
      spec:
        template:
          spec:
            containers:
            - name: istio-proxy
              resources:
                  limits:
                    cpu: "2"
                    memory: 2Gi
                  requests:
                    cpu: 100m
                    memory: 256Mi
  ---
  apiVersion: gateway.networking.k8s.io/v1
  kind: Gateway
  metadata:
    annotations:
      opendatahub.io/managed: 'false'
      security.opendatahub.io/authorino-tls-bootstrap: 'true'
    labels:
      opendatahub.io/managed: 'false'
    name: maas-default-gateway
    namespace: openshift-ingress
  spec:
    gatewayClassName: openshift-default
    infrastructure:
      parametersRef:
        group: ''
        kind: ConfigMap
        name: maas-default-gateway-config
    listeners:
      - allowedRoutes:
          namespaces:
            from: All
        hostname: maas.apps.<INSERT YOUR CLUSTER ROUTE HERE> # Make sure you use the router route. this is not a hard requirement, this is to simplify loadbalancer and tls config
        name: https
        port: 443
        protocol: HTTPS
        tls:
          certificateRefs:
            - group: ''
              kind: Secret
              name: cert-manager-ingress-cert # this is the default for RHDP-provisioned cert-manager wildcard cert
          mode: Terminate
  ```
- Red Hat Connectivity Link
  - Pin to 1.3.4
  - install into kuadrant-system namespace
  - create Kuadrant
    ```
    apiVersion: kuadrant.io/v1beta1
    kind: Kuadrant
    metadata:
      name: kuadrant
      namespace: kuadrant-system
    spec:
      observability:
        enable: true
    ```
  - modify Authorino service to use a service-serving certificate
    ```
    kind: Service
    apiVersion: v1
    metadata:
      name: authorino-authorino-authorization
      namespace: kuadrant-system
      annotations:
        service.beta.openshift.io/serving-cert-secret-name: authorino-server-cert
    ```
  - modify Authorino object to use the certificate
    ```
    apiVersion: operator.authorino.kuadrant.io/v1beta1
    kind: Authorino
    metadata:
      name: authorino
      namespace: kuadrant-system
    spec:
      listener:
        tls:
          certSecretRef:
            name: authorino-server-cert
          enabled: true
    ```

# Install RHOAI

- OpenShift AI
  - create the DSC when prompted, only need a few things explicitly
    ```
    apiVersion: datasciencecluster.opendatahub.io/v2
    kind: DataScienceCluster
    metadata:
      name: default-dsc
    spec:
      components:
        dashboard:
          managementState: Managed
        kserve:
          managementState: Managed
          modelsAsService:
            managementState: Managed
          rawDeploymentServiceConfig: Headed
        llamastackoperator:
          managementState: Managed
        modelregistry:
          managementState: Managed
          registriesNamespace: rhoai-model-registries
    ```
  - modify the DSCInitialization to add monitoring
    ```
    apiVersion: dscinitialization.opendatahub.io/v1
    kind: DSCInitialization
    metadata:
      name: default-dsci
    spec:
      monitoring:
        managementState: Managed
        metrics:
          storage:
            retention: 1d
            size: 5Gi
        namespace: redhat-ods-monitoring
    ```
  - recover the CNPG-generated connection URI for the MaaS DB from the `uri` key of the `openshift-ai-maas-app` secret
    in the `maas-db` namespace
  - create the secret for the MaaS db
    ```
    apiVersion: v1
    kind: Secret
    metadata:
      name: maas-db-config
      namespace: redhat-ods-applications
    type: Opaque
    stringData:
      DB_CONNECTION_URL: <INSERT THE URI FROM THE ABOVE SECRET>
    ```
  - confirm maas-controller and maas-api pods in redhat-ods-applications are ready
  - enable MaaS features in ODHDashboardConfig
    ```
    apiVersion: opendatahub.io/v1alpha
    kind: OdhDashboardConfig
    metadata:
      name: odh-dashboard-config
      namespace: redhat-ods-applications
    spec:
      dashboardConfig:
        genAiStudio: true
        modelAsService: true
        observabilityDashboard: true
    ```

# Use it

- deploy a model (nemotron-3-nano-30b-a3b is already downloaded onto your nodes)
- configure a subscription
- log in as your non-admin user and use it :)
