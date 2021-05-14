# Artifactory and Xray Logging Analytics with Fluentd and Datadog

The following document describes how to configure Datadog to gather metrics from Artifactory and Xray through the use of FluentD.

| version | artifactory | xray  | distribution | mission_control | pipelines |
|---------|-------------|-------|--------------|-----------------|-----------|
| 0.8.0   | 7.12.8      | 3.17.3| 2.6.0        | 4.6.3           | 1.12.2    |
| 0.7.0   | 7.10.2      | 3.11.3| 2.4.2        | 4.5.0           | 1.7.2     |
| 0.6.0   | 7.7.8       | 3.8.6 | 2.4.2        | 4.5.0           | 1.7.2     |
| 0.5.0   | 7.7.3       | 3.8.0 | 2.4.2        | 4.5.0           | 1.7.2     |
| 0.4.0   | 7.7.3       | 3.8.0 | 2.4.2        | 4.5.0           | N/A       |
| 0.3.0   | 7.7.3       | 3.8.0 | 2.4.2        | N/A             | N/A       |
| 0.2.0   | 7.7.3       | 3.8.0 | N/A          | N/A             | N/A       |
| 0.1.1   | 7.6.3       | 3.6.2 | N/A          | N/A             | N/A       |

## Table of Contents

1. [Environment Configuration](#environment-configuration)
2. [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog)
    * [Configuration steps for Artifactory](#configuration-steps-for-artifactory)
    * [Configuration steps for Xray](#configuration-steps-for-xray)
    * [Configuration steps for Mission Control](#configuration-steps-for-mission-control)
    * [Configuration steps for Distribution](#configuration-steps-for-distribution)
    * [Configuration steps for Pipelines](#configuration-steps-for-pipelines)
3. [Fluentd Installation](#fluentd-installation)
    * [OS / Virtual Machine](#os--virtual-machine)
    * [Docker](#docker)
    * [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
    * [Kubernetes Deployment without Helm](#kubernetes-deployment-without-helm)
4. [Datadog Setup](#datadog-setup)
    * [Dashboards](#dashboards)
5. [Demo Requirements](#demo-requirements)
6. [Generating data for Testing](#generating-data-for-testing)
7. [References](#references)

## Environment Configuration

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
Mision Control:
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

_**required**_: _API_KEY_ is the apiKey from Datadog

_dd_source_ attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

_include_tag_key_ defaults to false and it will add fluentd tag in the json record if set to true

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
  pos_file "#{ENV['JF_PRODUCT_DATA_INTERNAL']}/log/jfrog_siem.log.pos"
</source>
```

_**required**_: _JPD_URL_ is the Artifactory JPD URL of the format `http://<ip_address>` with is used to pull Xray Violations

_**required**_: _USER_ is the Artifactory username for authentication

_**required**_: _JFROG_API_KEY_ is the [Artifactory API Key](https://www.jfrog.com/confluence/display/JFROG/User+Profile#UserProfile-APIKey) for authentication

Override the match directive(last section) of the downloaded `fluent.conf.xray` with the details given below

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_jfrog_xray
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

_**required**_: _API_KEY_ is the apiKey from Datadog

_dd_source_ attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

_include_tag_key_ defaults to false and it will add fluentd tag in the json record if set to true

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

_**required**_: _API_KEY_ is the apiKey from Datadog

_dd_source_ attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

_include_tag_key_ defaults to false and it will add fluentd tag in the json record if set to true

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

_**required**_: _API_KEY_ is the apiKey from Datadog

_dd_source_ attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

_include_tag_key_ defaults to false and it will add fluentd tag in the json record if set to true

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

_**required**_: _API_KEY_ is the apiKey from Datadog

_dd_source_ attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.

_include_tag_key_ defaults to false and it will add fluentd tag in the json record if set to true


## Fluentd Installation

### OS / Virtual Machine

Recommended install is through fluentd's native OS based package installs:

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
Run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured above in the [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section.

````text
./fluentd test.conf
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
Run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured above in the [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section.

````text
./fluentd test.conf
````

### Kubernetes Deployment with Helm

Recommended install for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product | Example Values File |
|---------|-------------|
| Artifactory | helm/artifactory-values.yaml |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray | helm/xray-values.yaml |

Update the values.yaml associated to the product you want to deploy with your Datadog settings.

Then deploy the helm chart as described below:

Note: To generate ``masterKey`` and ``joinKey``, use the command
``openssl rand -hex 32``

Artifactory ⎈:

Replace the `datadog_api_key` towards the end with apiKey from Datadog and then run the following helm command

```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-values.yaml
```

Artifactory-HA ⎈:

For HA installation, please create a license secret on your cluster prior to installation.

Replace the `datadog_api_key` towards the end with apiKey from Datadog and then run the following helm command

```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set artifactory.license.secret=<license_secret_name> \
       --set artifactory.license.dataKey=artifactory.cluster.license \
       -f helm/artifactory-ha-values.yaml
```

Xray ⎈:

Update the following fields towards the end in `/helm/xray-values.yaml`:

Replace `datadog_api_key` with apiKey from Datadog

Replace `jfrog_user` with Artifactory username

Replace `jfrog_api_key` with [Artifactory API Key](https://www.jfrog.com/confluence/display/JFROG/User+Profile#UserProfile-APIKey)

Replace `jfrog_jpd_url` with Artifactory JPD URL of the format `http://<ip_address>` 

```text
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/xray-values.yaml
```

### Kubernetes Deployment without Helm

To modify existing Kubernetes based deployments without using Helm users can use the zip installer for Linux:

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above  in the [Environment Configuration](#environment-configuration) section.

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
Run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured above in the [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section.

````text
./fluentd test.conf
````


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
