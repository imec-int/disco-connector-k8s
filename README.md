# DISCO Sovity Connector

Includes a simple deployment of a sovity Connector that uses https://daps.disco.ai-data.imec.be/ as the authority
server.

## Docker Compose

The docker-compose file is provided for local development and testing. Create a `.env` file from the `.env.template` and
start the services.

## Prerequisites

- A Managed Kubernetes cluster
- [Helm](https://helm.sh/docs/intro/install/)
- DNS record for the domain you want to use
- Keystore issued by the authority server

## Installation

Connect to the Kubernetes cluster

   ```shell
   kubectl config use-context <your-cluster-name>
   ```

### Infrastructure

1. Install Ingress-Nginx controller (or any other ingress controller you prefer)
   ```shell
   helm upgrade --install ingress-nginx ingress-nginx \
     --repo https://kubernetes.github.io/ingress-nginx \
     --namespace ingress-nginx --create-namespace
   ```

   Confirm the installation by checking external IP of the LoadBalancer service

   ```shell
   kubectl get svc -n ingress-nginx
   ```

2. Add DNS record for the LoadBalancer service.

3. Install cert-manager
   ```shell
   helm repo add jetstack https://charts.jetstack.io --force-update

   helm install \
     cert-manager jetstack/cert-manager \
     --namespace cert-manager \
     --create-namespace \
     --version v1.16.3 \
     --set crds.enabled=true
   ```

4. Apply a ClusterIssuer [cluster-issuer.yaml](cluster/infrastructure/cluster-issuer.yaml)

   ```shell
   kubectl apply -f cluster/infrastructure/cluster-issuer.yaml
   ```

### Application

1. Install Postgres Operator

   ```shell
   # add repo for postgres-operator
   helm repo add postgres-operator-charts https://opensource.zalando.com/postgres-operator/charts/postgres-operator
   
   # install the postgres-operator
   helm install postgres-operator postgres-operator-charts/postgres-operator --namespace postgres-operator --create-namespace
   ```

2. Create a [connector.sec.yaml](cluster/connector/secrets/connector.sec.yaml) from the
   template [connector.sec.template.yaml](cluster/connector/secrets/connector.sec.template.yaml). Update
   `keystore-password` with the password provided by the authority server. Adjust the `api-auth-key` if needed.

3. Copy your keystore file `keystore.p12` to the `cluster/connector/secrets` directory.

4. Update the [connector.env](cluster/connector/config/connector/connector.env)
   and [connector-ui.env](cluster/connector/config/connector/connector-ui.env) files with the correct values.
    - `${client}` - your client id
    - `${domain}` - the domain you have set up

5. Adjust the [ingress.yaml](cluster/connector/ingress.yaml) file with the correct domain.

6. Apply the connector resources

   ```shell
   kubectl apply -k cluster
   ```

## Registration

1. Register the data plane of the connector with the control plane - update the `${domain}` and `${api-auth-key}`:

   ```shell
   curl --location 'https://${domain}/api/management/v2/dataplanes' \
   --header 'Content-Type: application/json' \
   --header 'x-api-key: ${api-auth-key}' \
   --data-raw '{
       "@context": {
           "edc": "https://w3id.org/edc/v0.0.1/ns/"
       },
       "edctype": "dataspaceconnector:dataplaneinstance",
       "id": "http-pull-dataplane",
       "url": "https://${domain}/api/management/transfer",
       "allowedSourceTypes": [
           "HttpData"
       ],
       "allowedDestTypes": [
           "HttpData",
           "HttpProxy"
       ],
       "properties": {
           "edc:publicApiUrl": "https://${domain}/public"
       }
   }'
   ```
