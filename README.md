# Artifactory and Xray Logging Analytics with Fluentd and Datadog

The following document describes how to configure Datadog to gather metrics from Artifactory and Xray through the use of FluentD.

## Table of Contents

`Note! You must follow the order of the steps throughout the Datadog Configuration`

1. [Environment Configuration](#environment-configuration)
2. [Fluentd Installation](#fluentd-installation)
    * [OS / Virtual Machine](#os--virtual-machine)
    * [Docker](#docker)
    * [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
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
| Gem Install**	 | MacOS & Linux - Gem			 | https://docs.fluentd.org/installation/install-by-gem | 

```text
** For Gem based install, Ruby Interpreter has to be setup first, following is the recommended process to install Ruby

1. Install Ruby Version Manager (RVM) as described in https://rvm.io/rvm/install#installation-explained, ensure to follow all the onscreen instructions provided to complete the rvm installation
	* For installation across users a SUDO based install is recommended, the installation is as described in https://rvm.io/support/troubleshooting#sudo

2. Once rvm installation is complete, verify the RVM installation executing the command 'rvm -v'

3. Now install ruby v2.7.0 or above executing the command 'rvm install <ver_num>', ex: 'rvm install 2.7.5'

4. Verify the ruby installation, execute 'ruby -v', gem installation 'gem -v' and 'bundler -v' to ensure all the components are intact

5. Post completion of Ruby, Gems installation, the environment is ready to further install new gems, execute the following gem install commands one after other to setup the needed ecosystem

	'gem install fluentd'

```

After FluentD is successfully installed, the below plugins are required to be installed

```text

	'gem install fluent-plugin-datadog'
	'gem install fluent-plugin-jfrog-siem'
	'gem install fluent-plugin-jfrog-metrics'

```


Configure `fluent.conf.*` according to the instructions mentioned in [Fluentd Configuration for Datadog](#fluentd-configuration-for-datadog) section and then run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured.

````text
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
````

### Docker

In order to run fluentd as a docker image to send the log, siem and metrics data to datadog, the following commands needs to be executed on the host that runs the docker.

1. Check the docker installation is functional, execute command 'docker version' and 'docker ps'.

2. Once the version and process are listed successfully, build the intended docker image for the observability platform using the docker file,

	* Download Dockerfile from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/Dockerfile) to any directory which has write permissions.

3. Download the Dockerenvfile_<observability_platform>.txt file needed to run Jfrog/FluentD Docker Images for the intended observability platform,

	* Download Dockerenvfile_datadog.txt from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/Dockerenvfile_datadog.txt) to the directory where the docker file was downloaded.

```text

For Datadog as the observability platform, execute these commands to setup the docker container running the fluentd installation

1. Execute 'docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t <image_name> .'

Command example

'docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t jfrog/fluentd-datadog-rt .'

The above command will build the docker image.

2. Fill the necessary information in the Dockerenvfile_datadogk.txt file, if the value for any of the field requires to have a '/' use '\/' and if '\' is required use '\\'.

3. Execute 'docker run -it --name jfrog-fluentd-datadogk-rt -v <path_to_logs>:/var/opt/jfrog/artifactory --env-file Dockerenvfile_datadog.txt <image_name>' 

The <path_to_logs> should be an absolute path where the Jfrog Artifactory Logs folder resides, i.e for an Docker based Artifactory Installation,  ex: /var/opt/jfrog/artifactory/var/logs on the docker host.

Command example

'docker run -it --name jfrog-fluentd-datadog-rt -v /var/opt/jfrog/artifactory/var:/var/opt/jfrog/artifactory --env-file Dockerenvfile_datadog.txt jfrog/fluentd-datadog-rt'


```

### Kubernetes Deployment with Helm

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

Artifactory ⎈:

Replace the `datadog_api_key` at the end of the yaml file with apiKey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/) and then run the following helm command:

```text
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
