# Guide to Configuring Datadog with various Jfrog Products

The following document describes how to configure Datadog to gather metrics from Artifactory and Xray through the use of FluentD.

## Table of Contents

1. [Environment Configuration](#environment-configuration)
2. [FluentD Configuration with Kubernetes with Helm](#kubernetes-deployment-with-helm)

## Environment Configuration

We rely heavily on environment variables so that the correct log files are streamed to your observalibity dashboards. Ensure that you set the JF_PRODUCT_DATA_INTERNAL environment variable to the correct path for your product

The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location.

Helm based installs will already have this defined based upon the underlying docker images.

For non-k8s based installations below is a reference to the Docker image locations per product. Note these locations may be different based upon the installation location chosen.

Note: These must be done one at a time for each product.

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
Please notice that this last one does not follow the same pattern as above.
````text
Pipelines:
export JF_PRODUCT_DATA_INTERNAL=/opt/jfrog/pipelines/var/
````

# For Kubernetes Installations with Helm 

Recommended installation for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product | Example Values File |
|---------|-------------|
| Artifactory | helm/artifactory-values.yaml |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray | helm/xray-values.yaml |

Update the values.yaml associated to the product you want to deploy with your Datadog settings.

Then deploy the helm chart as described below:

Add JFrog Helm repository:

```text
helm repo add jfrog https://charts.jfrog.io
helm repo update
```

Replace placeholders with your ``masterKey`` and ``joinKey``. To generate each of them, use the command
``openssl rand -hex 32``

Replace the `datadog_api_key` at the end of the yaml file for the product you want with apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/) and then run a helm upgrade that looks like this:

```text
helm upgrade --install <product_name>  jfrog/<product_name> \
       --set <product_name>.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/<product_name>-values.yaml
```

Note: Some products have slightly different steps listed below.

#### Per Product Instructions

Artifactory ⎈:

Run:
```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
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

Then run:
```text
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

```text
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/xray-values.yaml
```

Nginx:

Currently nginx can not send logs for helm based installations

# For Everything Else
## Downloading and Install FluentD

Follow the instructions found in the [fluentd docs](https://docs.fluentd.org/installation) to download and install fluentd or
Users can use the zip installer for Linux:

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

## Configuring Fluentd

Configure `fluent.conf.*` for your specified product according to these steps:

Download your fluentd configuration file to a directory the user has permissions to write, such as the $JF_PRODUCT_DATA_INTERNAL locations discussed above in the Environment Configuration section.

Artifactory Example:
````text
cd $JF_PRODUCT_DATA_INTERNAL
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.rt
````
Override the match directive(last section) of the downloaded `fluent.conf.*` with the details given below

Artifactory Example:
```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_artifactory
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

```@id``` tags the outgoing logs. This can be whatever you want but we recommend a name that helps indicate the logs origin.

_**required**_: ```API_KEY``` is the apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

```include_tag_key``` defaults to false and it will add fluentd tag in the json record if set to true

### Configuration steps for Xray

Xray configuration requires the following extra steps.

Fill in the JPD_URL, USER, JFROG_API_KEY fields in the source directive of the downloaded `fluent.conf.xray` with the details given below

```text
<source>
  @type jfrog_siem
  tag jfrog.xray.siem.vulnerabilities
  jpd_url JPD_URL
  username USER
  apikey JFROG_API_KEY
  pos_file "#{ENV['JF_PRODUCT_DATA_INTERNAL']}/log/jfrog_siem.log.pos"
</source>
```

_**required**_: ```JPD_URL``` is the Artifactory JPD URL of the format `http://<ip_address>` with is used to pull Xray Violations

_**required**_: ```USER``` is the Artifactory username for authentication

_**required**_: ```JFROG_API_KEY``` is the [Artifactory API Key](https://www.jfrog.com/confluence/display/JFROG/User+Profile#UserProfile-APIKey) for authentication

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
