# JFrog Artifactory and Xray Log Analytics with Fluentd and Datadog

The following document describes how to configure Datadog to gather logs, metrics and violations from Artifactory and Xray through the use of FluentD.

## Versions Supported

This integration is last tested with Artifactory 7.71.11 and Xray 3.88.12 versions.

## Table of Contents

`Note! You must follow the order of the steps throughout Datadog Configuration`
1. [Datadog Setup](#datadog-setup)
2. [JFrog Metrics Setup](#metrics-setup)
3. [Fluentd Installation](#fluentd-installation)
   * [OS / Virtual Machine](#os--virtual-machine)
   * [Docker](#docker)
   * [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
4. [Dashboards](#dashboards)
5. [References](#references)

## Datadog Setup

Datadog setup can be done by going through the below onboarding steps or by using apiKey directly if one exists. If an apiKey exists, skip the steps below and move on to [Fluentd Installation](#fluentd-installation) to forward logs directly to your datadog account.

* Create an account in Datadog
* Run the datadog agent in your kubernetes cluster by deploying it with a helm chart
* To enable log collection, update datadog-values.yaml file given in the onboarding steps of datadog
* Once the agent starts reporting, you'll get an apiKey which we'll be using to send formatted logs through fluentd

Once datadog is setup, we can access logs via Logs > Search. We can also select the specific source that we want to get logs from. Adding proper metadata is the key to unlocking the full potential of your logs in datadog. By default, the hostname and timestamp fields should be remapped.

* Add all attributes as facets from Facets > Add on the left side of the screen in Logs > search

## JFrog Metrics Setup
To enable metrics in Artifactory, make the following configuration changes to the [Artifactory System YAML](https://www.jfrog.com/confluence/display/JFROG/Artifactory+System+YAML)
```yaml
artifactory:
    metrics:
        enabled: true
    openMetrics:
        enabled: true
```
Once this configuration is done and the application is restarted, metrics will be available in Open Metrics Format

Metrics are enabled by default in Xray. 
For kubernetes based installs, openMetrics are enabled in the helm install commands listed below

## Fluentd Installation

### OS / Virtual Machine
Ensure you have access to the Internet from a virtual machine (VM). We recommend installation through FluentD's native OS based package installs:

| OS             | Package Manager        | Link                                                 |
|----------------|------------------------|------------------------------------------------------|
| CentOS/RHEL    | Linux - RPM (YUM)      | https://docs.fluentd.org/installation/install-by-rpm |
| Debian/Ubuntu  | Linux - APT            | https://docs.fluentd.org/installation/install-by-deb |
| MacOS/Darwin   | MacOS - DMG            | https://docs.fluentd.org/installation/install-by-dmg |
| Windows        | Windows - MSI          | https://docs.fluentd.org/installation/install-by-msi |
| Gem Install**	 | MacOS & Linux - Gem			 | https://docs.fluentd.org/installation/install-by-gem | 

##### Gem based install
For a Gem-based install, the Ruby Interpreter must be setup first. You can install the Ruby Interpreter by doing the following:

1. Install Ruby Version Manager (RVM) outlined in the [RVM documentation](https://rvm.io/rvm/install#installation-explained).
   * Use the `SUDO` command  for multi-user installation. For more information, see the [RVM troubleshooting documentation](https://rvm.io/support/troubleshooting#sudo).

2. After the RVM installation is complete, execute the command 'rvm -v' to verify.

3. Install Ruby v2.7.0 or above with the command `rvm install <ver_num>`, (for example, `rvm install 2.7.5`).

4. Verify the Ruby installation, execute `ruby -v`, gem installation `gem -v` and `bundler -v` to ensure all the components are intact.

5. Install the FluentD gem with the command `gem install fluentd`.

6. After FluentD is successfully installed, install the following plugins.

```shell
gem install fluent-plugin-concat
gem install fluent-plugin-datadog
gem install fluent-plugin-jfrog-siem
gem install fluent-plugin-jfrog-metrics
gem install fluent-plugin-jfrog-send-metrics
```

##### Configure Fluentd
We rely on environment variables to stream log files to your observability dashboards. Ensure that you fill in the `.env` file with the correct values. You can download the `.env` file [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/jfrog.env).

* **JF_PRODUCT_DATA_INTERNAL**: The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location. For each JFrog service, you can find its active log files in the `$JFROG_HOME/<product>/var/log` directory
* **DATADOG_API_KEY**: API Key from [Datadog](https://app.datadoghq.com/organization-settings/api-keys)
* **JPD_URL**: Artifactory JPD URL with the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
* **COMMON_JPD**: This flag should be set as true only for non-Kubernetes installations or installations where the JPD base URL is the same to access both Artifactory and Xray (for example, `https://sample_base_url/artifactory` or `https://sample_base_url/xray`)

Apply the `.env` files and run the fluentd wrapper with the following command, and note that the argument points to the `fluent.conf.*` file previously configured:

```shell
source jfrog.env
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
```

### Docker
In order to run FluentD as a docker image to send the logs, violations, and metrics data to Datadog, execute the following commands on the host that runs the docker.

1. Execute the `docker version` and `docker ps` commands to verify that the Docker installation is functional.

2. If the version and process are listed successfully, build the intended docker image for Datadog using the docker file. You can download [this Dockerfile]https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/Dockerfile to any directory that has write permissions.

3. Download the `docker.env` file needed to run `Jfrog/FluentD` Docker Images for Datadog. You can download [this docker.env]https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/docker.env to the directory where the docker file was downloaded.

4. Execute the following command to build the docker image: `docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t <image_name>`. For example:

    ```shell
     docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t jfrog/fluentd-datadog-rt .'
    ```

5. Fill out the necessary information in the docker.env file:

   * **JF_PRODUCT_DATA_INTERNAL**: The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location. For each JFrog service you will find its active log files in the `$JFROG_HOME/<product>/var/log` directory
   * **DATADOG_API_KEY**: API Key from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
   * **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
   * **JPD_ADMIN_USERNAME**: Artifactory username for authentication
   * **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
   * **COMMON_JPD**: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

6. Execute 'docker run -it --name jfrog-fluentd-datadog-rt -v <path_to_logs>:/var/opt/jfrog/artifactory --env-file docker.env <image_name>'

   The `<path_to_logs>` should be an absolute path where the Jfrog Artifactory Logs folder resides, such as a Docker based Artifactory Installation like`/var/opt/jfrog/artifactory/var/logs` on the docker host. For example:

    ```shell
     docker run -it --name jfrog-fluentd-datadog-rt -v $JFROG_HOME/artifactory/var/:/var/opt/jfrog/artifactory --env-file docker.env jfrog/fluentd-datadog-rt
    ```


### Kubernetes Deployment with Helm
The recommended installation method for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product        | Example Values File             |
|----------------|---------------------------------|
| Artifactory    | helm/artifactory-values.yaml    |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray           | helm/xray-values.yaml           |

Add JFrog Helm repository:

```shell
helm repo add jfrog https://charts.jfrog.io
helm repo update
```
Replace placeholders with your ``masterKey`` and ``joinKey``. To generate each of them, use the command
``openssl rand -hex 32``

#### Artifactory ⎈:

1. Skip this step if you already have Artifactory installed. Else, install Artifactory using the command below
    ```shell
    helm upgrade --install artifactory  jfrog/artifactory \
           --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
           --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
           --set artifactory.license.secret=artifactory-license \
           --set artifactory.license.dataKey=artifactory.cluster.license \
           --set artifactory.metrics.enabled=true \
           --set artifactory.openMetrics.enabled=true
    ```

2. Create a secret for JFrog's admin token - [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) using any of the following methods
    ```shell
    kubectl create secret generic jfrog-admin-token --from-file=token=<path_to_token_file>
    
    OR
    
    kubectl create secret generic jfrog-admin-token --from-literal=token=<JFROG_ADMN_TOKEN>
    ```
3. For Artifactory installation, download the .env file from [here](https://github.com/jfrog/log-analytics-datadog/raw/master/helm/jfrog_helm.env). Fill in the jfrog_helm.env file with correct values.

   * **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying Docker images. Not a required field for k8s installation
   * **DATADOG_API_KEY**: API Key from [Datadog](https://app.datadoghq.com/organization-settings/api-keys)
   * **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
   * **JPD_ADMIN_USERNAME**: Artifactory username for authentication
   * **COMMON_JPD**: This flag should be set as true only for non-Kubernetes installations or installations where the JPD base URL is the same to access both Artifactory and Xray (for example, `https://sample_base_url/artifactory` or `https://sample_base_url/xray`)

   Apply the .env files using the helm command below

   ```shell
   source jfrog_helm.env
   ```
4. Postgres password is required to upgrade Artifactory. Run the following command to get the current password
   ```shell
   POSTGRES_PASSWORD=$(kubectl get secret artifactory-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
   ```
5. Upgrade Artifactory installation using the command below
    ```shell
    helm upgrade --install artifactory jfrog/artifactory \
           --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
           --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
           --set artifactory.metrics.enabled=true --set artifactory.openMetrics.enabled=true \
           --set databaseUpgradeReady=true --set postgresql.postgresqlPassword=$POSTGRES_PASSWORD --set nginx.service.ssloffload=true \
           --set datadog.api_key=$DATADOG_API_KEY  \
           --set jfrog.observability.jpd_url=$JPD_URL \
           --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
           --set jfrog.observability.common_jpd=$COMMON_JPD \
           -f helm/artifactory-values.yaml
    ```

#### Artifactory-HA ⎈:

1. For HA installation, please create a license secret on your cluster prior to installation.
   ```shell
   kubectl create secret generic artifactory-license --from-file=<path_to_license_file>artifactory.cluster.license 
   ```
2. Skip this step if you already have Artifactory installed. Else, install Artifactory using the command below
    ```shell
    helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set artifactory.license.secret=artifactory-license \
       --set artifactory.license.dataKey=artifactory.cluster.license \
       --set artifactory.metrics.enabled=true \
       --set artifactory.openMetrics.enabled=true
    ```
3. Create a secret for JFrog's admin token - [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) using any of the following methods
   ```shell
   kubectl create secret generic jfrog-admin-token --from-file=token=<path_to_token_file>
   
   OR
   
   kubectl create secret generic jfrog-admin-token --from-literal=token=<JFROG_ADMN_TOKEN>
   ```
4. Download the .env file from [here](https://github.com/jfrog/log-analytics-datadog/raw/master/helm/jfrog_helm.env). Fill in the jfrog_helm.env file with correct values.

   * **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying Docker images. Not a required field for k8s installation
   * **DATADOG_API_KEY**: API Key from [Datadog](https://app.datadoghq.com/organization-settings/api-keys)
   * **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
   * **JPD_ADMIN_USERNAME**: Artifactory username for authentication
   * **COMMON_JPD**: This flag should be set as true only for non-Kubernetes installations or installations where the JPD base URL is the same to access both Artifactory and Xray (for example, `https://sample_base_url/artifactory` or `https://sample_base_url/xray`)

   Apply the .env files and then run the helm command below

   ```shell
   source jfrog_helm.env
   ```
5. Postgres password is required to upgrade Artifactory. Run the following command to get the current password
   ```shell
   POSTGRES_PASSWORD=$(kubectl get secret artifactory-ha-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)
   ```
6. Upgrade Artifactory HA installation using the command below
   ```text
   helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE --set artifactory.replicaCount=0 \
       --set artifactory.metrics.enabled=true --set artifactory.openMetrics.enabled=true \
       --set databaseUpgradeReady=true --set postgresql.postgresqlPassword=$POSTGRES_PASSWORD --set nginx.service.ssloffload=true \
       --set datadog.api_key=$DATADOG_API_KEY  \
       --set jfrog.observability.jpd_url=$JPD_URL \
       --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
       --set jfrog.observability.common_jpd=$COMMON_JPD \
       -f helm/artifactory-ha-values.yaml
   ```

#### Xray ⎈:

Create a secret for JFrog's admin token - [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) using any of the following methods if it doesn't exist
```shell
kubectl create secret generic jfrog-admin-token --from-file=token=<path_to_token_file>

OR

kubectl create secret generic jfrog-admin-token --from-literal=token=<JFROG_ADMN_TOKEN>
```

For Xray installation, download the .env file from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/jfrog_helm.env). Fill in the jfrog_helm.env file with correct values.

* **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying Docker images. Not a required field for k8s installation
* **DATADOG_API_KEY**: API Key from [Datadog](https://app.datadoghq.com/organization-settings/api-keys)
* **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **COMMON_JPD**: This flag should be set as true only for non-Kubernetes installations or installations where the JPD base URL is the same to access both Artifactory and Xray (for example, `https://sample_base_url/artifactory` or `https://sample_base_url/xray`)

Apply the .env files and then run the helm command below

```shell
source jfrog_helm.env
```

Use the same `joinKey` as you used in Artifactory installation to allow Xray node to successfully connect to Artifactory.

```shell
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set datadog.api_key=$DATADOG_API_KEY  \
       --set jfrog.observability.jpd_url=$JPD_URL \
       --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
       --set jfrog.observability.common_jpd=$COMMON_JPD \
       -f helm/xray-values.yaml
```

### Dashboards

#### JFrog Artifactory Dashboard 
This dashboard is divided into three sections Application, Audit and Requests
* **Application** - This section tracks Log Volume(information about different log sources) and Artifactory Errors over time(bursts of application errors that may otherwise go undetected)
* **Audit** - This section tracks audit logs help you determine who is accessing your Artifactory instance and from where. These can help you track potentially malicious requests or processes (such as CI jobs) using expired credentials.
* **Requests** - This section tracks HTTP response codes, Top 10 IP addresses for uploads and downloads

#### JFrog Artifactory Metrics dashboard
This dashboard tracks Artifactory System Metrics, JVM memory, Garbabe Collection, Database Connections, and HTTP Connections metrics

#### JFrog Xray Logs dashboard
This dashboard provides a summary of access, service and traffic log volumes associated with Xray. Additionally, customers are also able to track various HTTP response codes, HTTP 500 errors, and log errors for greater operational insight 

#### JFrog Xray Violations Dashboard
This dashboard provides an aggregated summary of all the license violations and security vulnerabilities found by Xray. Information is segment by watch policies and rules. Trending information is provided on the type and severity of violations over time, as well as, insights on most frequently occurring CVEs, top impacted artifacts and components.

#### JFrog Xray Metrics Dashboard
This dashboard tracks System Metrics, and data metrics about Scanned Artifacts and Scanned Components


## Demo Requirements

* Kubernetes Cluster
* Artifactory and/or Xray installed via [JFrog Helm Charts](https://github.com/jfrog/charts)
* Helm 3

## Generating Data for Testing

[Partner Integration Test Framework](https://github.com/jfrog/partner-integration-tests) can be used to generate data for metrics.

## References

* [Datadog](https://docs.datadoghq.com/getting_started/) - Cloud monitoring as a service
