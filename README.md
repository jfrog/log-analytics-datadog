# Artifactory and Xray Logging Analytics with Fluentd and Datadog

The following document describes how to configure Datadog to gather metrics from Artifactory and Xray through the use of FluentD.

## Table of Contents

1. [Environment Configuration](#environment-configuration)
2. [Fluentd Installation](#fluentd-installation)
    * [OS / Virtual Machine](#os--virtual-machine)
    * [Docker](#docker)
    * [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
    * [Kubernetes Deployment without Helm](#kubernetes-deployment-without-helm)
3. [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog)
    * [Configuration steps for Artifactory](#configuration-steps-for-artifactory)
    * [Configuration steps for Xray](#configuration-steps-for-xray)
    * [Configuration steps for Mission Control](#configuration-steps-for-mission-control)
    * [Configuration steps for Distribution](#configuration-steps-for-distribution)
    * [Configuration steps for Pipelines](#configuration-steps-for-pipelines)
4. [Datadog Setup](#datadog-setup)
    * [Dashboards](#dashboards)
5. [Demo Requirements](#demo-requirements)
6. [Generating data for Testing](#generating-data-for-testing)
7. [References](#references)

## Environment Configuration

We rely heavily on environment variables so that the correct log files are streamed to your observalibity dashboards. Ensure that you set the JF_PRODUCT_DATA_INTERNAL environment variable to the correct path for your product

The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location.

Helm based installs will already have this defined based upon the underlying docker images.

For non-k8s based installations below is a reference to the Docker image locations per product. Note these locations may be different based upon the installation location chosen.

````text
Artifactory: 
export JF_PRODUCT_DATA_INTERNAL=/var/opt/jfrog/artifactory/
````

````text
Xray:
export JF_PRODUCT_DATA_INTERNAL=/var/opt/jfrog/xray/
````

````text
Nginx:
export JF_PRODUCT_DATA_INTERNAL=/var/opt/jfrog/nginx/
````

````text
Mission Control:
export JF_PRODUCT_DATA_INTERNAL=/var/opt/jfrog/mc/
````

````text
Distribution:
export JF_PRODUCT_DATA_INTERNAL=/var/opt/jfrog/distribution/
````

````text
Pipelines:
export JF_PRODUCT_DATA_INTERNAL=/opt/jfrog/pipelines/var/
````

## Fluentd Installation

### OS / Virtual Machine

Ensure you have access to the Internet from VM. Recommended install is through fluentd's native OS based package installs:

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| CentOS/RHEL   | RPM (YUM)       | https://docs.fluentd.org/installation/install-by-rpm |
| Debian/Ubuntu | APT             | https://docs.fluentd.org/installation/install-by-deb |
| MacOS/Darwin  | DMG             | https://docs.fluentd.org/installation/install-by-dmg |
| Windows       | MSI             | https://docs.fluentd.org/installation/install-by-msi |

User installs can utilize the zip installer for Linux

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz
````

Untar to create the folder:
````text
tar -xvf fluentd-1.11.0-linux-x86_64.tar.gz
````
Move into the new folder:

````text
cd fluentd-1.11.0-linux-x86_64
````

Configure `fluent.conf.*` according to the instructions mentioned in [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section and then run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured.

````text
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
````

### Docker

Recommended install for Docker is to utilize the zip installer for Linux

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz
````

Untar to create the folder:
````text
tar -xvf fluentd-1.11.0-linux-x86_64.tar.gz
````
Move into the new folder:

````text
cd fluentd-1.11.0-linux-x86_64
````

Configure `fluent.conf.*` according to the instructions mentioned in [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section and then run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured.

````text
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
````

### Kubernetes Deployment with Helm

Recommended installation for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product        | Example Values File |
|----------------|-------------|
| Platform       | helm/jfrog-platform-values.yaml |
| Artifactory    | helm/artifactory-values.yaml |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray           | helm/xray-values.yaml |

Update the values.yaml associated to the product you want to deploy with your Datadog settings.

Then deploy the helm chart as described below:

Add JFrog Helm repository:

```kubernetes helm
helm repo add jfrog https://charts.jfrog.io
helm repo update
```

JFrog Platform ⎈:

```textmate
!!!Important Note!!!: Platform Chart Deployment shown here is for reference purpose only, 
Kindly apply these charts only after reviewing the options and make necessary changes that suits your deployment
```

Pre-requisite for Observability integration with JFrog Platform charts

First, install the JFrog Platform chart by configuring intended replicaCount for Artifactory, Xray and enable or disable the solutions

Refer the sample yaml for reference, download [here](https://github.com/jfrog/log-analytics-datadog/blob/master/helm/jfrog-platform-values-without-datadog.yaml)

```yaml

installerInfo: '{ "productId": "Helm_datadog_artifactory/{{ .Chart.Version }}", "features": [ { "featureId": "ArtifactoryVersion/{{ default .Chart.AppVersion .Values.artifactory.image.version }}" }, { "featureId": "{{ if .Values.postgresql.enabled }}postgresql{{ else }}{{ .Values.database.type }}{{ end }}/0.0.0" }, { "featureId": "Platform/{{ default "kubernetes" .Values.installer.platform }}" },  { "featureId": "Channel/Helm_datadog_artifactory" } ] }'
artifactory:
  artifactory:
    openMetrics:
      enabled: true
xray:
  enabled: true
  replicaCount: 1  
insight:
  enabled: false
distribution:
  enabled: false
pipelines:
  enabled: false
rabbitmq:
  enabled: true
redis:
  enabled: false

```
To install the JFrog Platform with the above said configurations run the following command, (note the namespaces and adjust as needed by your deployment requirement)

```kubernetes helm
helm upgrade --install jfrog-platform --namespace jfrog-platform jfrog/jfrog-platform -f jfrog-platform-values-without-datadog.yaml
```

Once the platform is accessible, login to the platform and perform the setup as directed on the UI.

Second, when your JFrog Platform is ready and accessible, the following should be noted

1. Access Token - click [here](https://www.jfrog.com/confluence/display/JFROG/Access+Tokens#AccessTokens-GeneratingScopedTokens) to know how to generate a admin scoped access token
2. API Key - click [here](https://www.jfrog.com/confluence/display/RTF6X/Updating+Your+Profile#UpdatingYourProfile-APIKey) to generate an API Key with profile update process
3. User - admin
   Refer [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section for configuring the below parameters
4. Datadog API Key - click [here](https://docs.datadoghq.com/account_management/api-app-keys/), to get the API Key which should be used to send data to Datadog

Once the values are noted, download the file to apply the JFrog Platform Upgrade for Datadog from [here](https://github.com/jfrog/log-analytics-datadog/blob/master/helm/jfrog-platform-values.yaml)

Replace the respective values for the following in the global segment of chart values, review the chart values and apply them accordingly

```yaml
global:
   datadog:
      apikey: datadog_api_key
   jfrog:
      observability:
         metrics:
            jpd_url: http://localhost:8082
            jpd_url_nginx: http://jfrog-platform-artifactory-nginx
            username: jfrog_user
            apikey: jfrog_api_key
            token: jfrog_token
         branch: master
```

Once the values are replaced, apply the upgrade as mentioned

1. Get the JFrog Platform Postgres Password and store it to a variable
```shell
export POSTGRES_PASSWORD=kubectl get secret -n jfrog-platform  jfrog-platform-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode
```
2. Using the password obtained, run the following upgrade command
```kubernetes helm
helm upgrade jfrog-platform --namespace jfrog-platform jfrog/jfrog-platform --set databaseUpgradeReady=true --set postgresql.postgresqlPassword=$POSTGRES_PASSWORD -f jfrog-platform-values.yaml
```
3. For getting metrics data kindly follow the documentation on Datadog agent based setup [here](https://github.com/jfrog/metrics/blob/main/datadog/README.md) 

```textmate
!!!Important Note!!!: Platform Chart Deployment shown here is for reference purpose only, 
Kindly apply these charts only after reviewing the options and make necessary changes that suits your deployment
```



Replace placeholders with your ``masterKey`` and ``joinKey``. To generate each of them, use the command
``openssl rand -hex 32``

Artifactory ⎈:

Replace the `datadog_api_key` at the end of the yaml file with apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/) and then run the following helm command:

```kubernetes helm
helm upgrade --install artifactory  jfrog/artifactory \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-values.yaml
```

Artifactory-HA ⎈:

For HA installation, please create a license secret on your cluster prior to installation.

```text
kubectl create secret generic artifactory-license --from-file=<path_to_license_file>artifactory.cluster.license 
```

Note: Replace placeholders with your ``masterKey`` and ``joinKey``. To generate each of them, use the command
``openssl rand -hex 32``

Replace the `datadog_api_key` at the end of the yaml file with apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/) and then run the following helm command

```kubernetes helm
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-ha-values.yaml
```

Xray ⎈:

Update the following fields in `/helm/xray-values.yaml`:

Replace `datadog_api_key` in `datadog.apiKey` with apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

Replace `jfrog_user` in `jfrog.siem.username` with Artifactory username 

Replace `jfrog_api_key` in `jfrog.siem.apikey` with [Artifactory API Key](https://www.jfrog.com/confluence/display/JFROG/User+Profile#UserProfile-APIKey)

Replace `jfrog_jpd_url` in `jfrog.siem.jpdUrl` with Artifactory JPD URL of the format `http://<ip_address>` 

Use the same `joinKey` as you used in Artifactory installation to allow Xray node to successfully connect to Artifactory.

```kubernetes helm
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/xray-values.yaml
```

### Kubernetes Deployment without Helm

To modify existing Kubernetes based deployments without using Helm, users can use the zip installer for Linux:

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz
````

Untar to create the folder:
````text
tar -xvf fluentd-1.11.0-linux-x86_64.tar.gz
````
Move into the new folder:

````text
cd fluentd-1.11.0-linux-x86_64
````
Configure `fluent.conf.*` according to the instructions mentioned in [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section and then run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured.

````text
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
````

## Fluentd Configuration for Datadog

Download and configure the relevant fluentd.conf files for Datadog

### Configuration steps for Artifactory

Download the artifactory fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.rt
````

Override the match directive(last section) of the downloaded `fluent.conf.rt` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_artifactory
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true

Authenticate with the Artifactory API by replacing `<TOKEN>` with your bearer token in the downloaded `fluent.conf.rt` file.
There should be two spots listed below:

```
command "curl --request GET 'http://localhost:8081/artifactory/api/system/version' -H 'Authorization: Bearer <TOKEN>'"
headers {"Authorization":"Bearer <TOKEN>"}
```

For information on authentication with a bearer token with artifactory, please visit [Bearer Token Authentication](https://www.jfrog.com/confluence/display/JFROG/Access+Tokens#AccessTokens)

### Configuration steps for Xray

Download the Xray fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.xray
````

Fill in the JPD_URL, USER, JFROG_API_KEY fields in the source directive of the downloaded `fluent.conf.xray` with the details given below

```text
<source>
  @type jfrog_siem
  tag jfrog.xray.siem.vulnerabilities
  jpd_url JPD_URL
  username USER
  apikey JFROG_API_KEY
  pos_file_path "#{ENV['JF_PRODUCT_DATA_INTERNAL']}/log/jfrog_siem.log.pos"
  from_date "2016-01-01"
</source>
```

_**required**_: ```JPD_URL``` is the Artifactory JPD URL of the format `http://<ip_address>` with is used to pull Xray Violations

_**required**_: ```USER``` is the Artifactory username for authentication

_**required**_: ```JFROG_API_KEY``` is the [Artifactory API Key](https://www.jfrog.com/confluence/display/JFROG/User+Profile#UserProfile-APIKey) for authentication

_**optional**_: If not specified, value is set to current date. Setting from_date value will result in violations from the specified date

Override the match directive (last section) of the downloaded `fluent.conf.xray` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_xray
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true


### Configuration steps for Nginx

Download the Mission Control fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.nginx
````

Override the match directive(last section) of the downloaded `fluent.conf.nginx` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_nginx
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true


### Configuration steps for Mission Control

Download the Mission Control fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.missioncontrol
````

Override the match directive(last section) of the downloaded `fluent.conf.missioncontrol` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_missioncontrol
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true

### Configuration steps for Distribution

Download the distribution fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.distribution
````

Override the match directive(last section) of the downloaded `fluent.conf.distribution` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_distribution
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true

### Configuration steps for Pipelines

Download the pipelines fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the [Environment Configuration](#environment-configuration) section.

````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.pipelines
````

Override the match directive(last section) of the downloaded `fluent.conf.pipelines` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_pipelines
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true


## Datadog Setup

Datadog setup can be done by going through the below onboarding steps or by using apiKey directly if one exists. If an apiKey exists, use the Datadog Fluentd plugin to forward logs directly from Fluentd to your datadog account. 

* Create an account in Datadog
* Run the datadog agent in your kubernetes cluster by deploying it with a helm chart
* To enable log collection, update datadog-values.yaml file given in the onboarding steps of datadog
* Once the agent starts reporting, you'll get an apiKey which we'll be using to send formatted logs through fluentd

Once datadog is setup, we can access logs via Logs > Search. We can also select the specific source that we want to get logs from. Adding proper metadata is the key to unlocking the full potential of your logs in datadog. By default, the hostname and timestamp fields should be remapped.

* Add all attributes as facets from Facets > Add on the left side of the screen in Logs > search

### Dashboards
JFrog Artifactory Dashboard is divided into three sections Application, Audit and Requests

* **Application** - This section tracks Log Volume(information about different log sources) and Artifactory Errors over time(bursts of application errors that may otherwise go undetected)
* **Audit** - This section tracks audit logs help you determine who is accessing your Artifactory instance and from where. These can help you track potentially malicious requests or processes (such as CI jobs) using expired credentials.
* **Requests** - This section tracks HTTP response codes, Top 10 IP addresses for uploads and downloads

JFrog Artifactory and Xray Metrics dashboards:

JFrog Artifactory’s/Xray's Metrics API integration with Datadog allows you to send metrics from the Artifactory’s/Xray's Open Metrics API endpoint to Datadog. With this integration, you can gain insights into the system performance, storage consumption, and connection statistics associated with JFrog Artifactory/Xray, as well as, insights into the count and type of artifacts and components scanned by Xray. Upon setting up the configuration, these metrics are made available as out-of-the-box dashboards within the Datadog UI and may be used to enhance existing dashboards within Datadog.
## Demo Requirements

* Kubernetes Cluster
* Artifactory and/or Xray installed via [JFrog Helm Charts](https://github.com/jfrog/charts)
* Helm 3

## Generating Data for Testing
[Partner Integration Test Framework](https://github.com/jfrog/partner-integration-tests) can be used to generate data for metrics.

## References
* [Datadog](https://docs.datadoghq.com/getting_started/) - Cloud monitoring as a service
