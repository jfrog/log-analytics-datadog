# JFrog Artifactory and Xray Log Analytics with Fluentd and Datadog

The following document describes how to configure Datadog to gather logs, metrics and violations from Artifactory and Xray through the use of FluentD.

## Table of Contents

`Note! You must follow the order of the steps throughout Datadog Configuration`
1. [Datadog Setup](#datadog-setup)
2. [Fluentd Installation](#fluentd-installation)
   * [OS / Virtual Machine](#os--virtual-machine)
   * [Docker](#docker)
   * [Kubernetes Deployment with Helm](#kubernetes-deployment-with-helm)
3. [Dashboards](#dashboards)
5. [References](#references)

## Datadog Setup

Datadog setup can be done by going through the below onboarding steps or by using apiKey directly if one exists. If an apiKey exists, use the Datadog Fluentd plugin to forward logs directly from Fluentd to your datadog account.

* Create an account in Datadog
* Run the datadog agent in your kubernetes cluster by deploying it with a helm chart
* To enable log collection, update datadog-values.yaml file given in the onboarding steps of datadog
* Once the agent starts reporting, you'll get an apiKey which we'll be using to send formatted logs through fluentd

Once datadog is setup, we can access logs via Logs > Search. We can also select the specific source that we want to get logs from. Adding proper metadata is the key to unlocking the full potential of your logs in datadog. By default, the hostname and timestamp fields should be remapped.

* Add all attributes as facets from Facets > Add on the left side of the screen in Logs > search

## Fluentd Installation

### OS / Virtual Machine
Ensure you have access to the Internet from VM. Recommended install is through fluentd's native OS based package installs:

| OS             | Package Manager        | Link                                                 |
|----------------|------------------------|------------------------------------------------------|
| CentOS/RHEL    | Linux - RPM (YUM)      | https://docs.fluentd.org/installation/install-by-rpm |
| Debian/Ubuntu  | Linux - APT            | https://docs.fluentd.org/installation/install-by-deb |
| MacOS/Darwin   | MacOS - DMG            | https://docs.fluentd.org/installation/install-by-dmg |
| Windows        | Windows - MSI          | https://docs.fluentd.org/installation/install-by-msi |
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

````text
gem install fluent-plugin-concat
gem install fluent-plugin-datadog
gem install fluent-plugin-jfrog-siem
gem install fluent-plugin-jfrog-metrics
gem install fluent-plugin-jfrog-send-metrics
````

#### Configure Fluentd
We rely heavily on environment variables so that the correct log files are streamed to your observability dashboards. Ensure that you fill in the .env file with correct values. Download the .env file from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/.env_jfrog)

* **JF_PRODUCT_DATA_INTERNAL**: The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location. For each JFrog service you will find its active log files in the `$JFROG_HOME/<product>/var/log` directory
* **DATADOG_API_KEY**: APIkey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
* **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
* **COMMON_JPD**: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

Apply the .env files and then run the fluentd wrapper with one argument pointed to the `fluent.conf.*` file configured.

````text
source .env_jfrog
./fluentd $JF_PRODUCT_DATA_INTERNAL/fluent.conf.<product_name>
````

### Docker
In order to run fluentd as a docker image to send the logs, violations and metrics data to datadog, the following commands needs to be executed on the host that runs the docker.

1. Check the docker installation is functional, execute command 'docker version' and 'docker ps'.

2. Once the version and process are listed successfully, build the intended docker image for Datadog using the docker file,

    * Download Dockerfile from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/Dockerfile) to any directory which has write permissions.

3. Download the Dockerenvfile.txt file needed to run Jfrog/FluentD Docker Images for Datadog,

    * Download Dockerenvfile.txt from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/docker-build/Dockerenvfile.txt) to the directory where the docker file was downloaded.

```text

For Datadog as the observability platform, execute these commands to setup the docker container running the fluentd installation

1. Execute 'docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t <image_name> .'

    Command example

    'docker build --build-arg SOURCE="JFRT" --build-arg TARGET="DATADOG" -t jfrog/fluentd-datadog-rt .'

    The above command will build the docker image.

2. Fill the necessary information in the Dockerenvfile.txt file

    JF_PRODUCT_DATA_INTERNAL: The environment variable JF_PRODUCT_DATA_INTERNAL must be defined to the correct location. For each JFrog service you will find its active log files in the `$JFROG_HOME/<product>/var/log` directory
    DATADOG_API_KEY: APIkey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
    JPD_URL: Artifactory JPD URL of the format `http://<ip_address>`
    JPD_ADMIN_USERNAME: Artifactory username for authentication
    JFROG_ADMIN_TOKEN: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
    COMMON_JPD: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

3. Execute 'docker run -it --name jfrog-fluentd-datadog-rt -v <path_to_logs>:/var/opt/jfrog/artifactory --env-file Dockerenvfile.txt <image_name>' 

    The <path_to_logs> should be an absolute path where the Jfrog Artifactory Logs folder resides, i.e for an Docker based Artifactory Installation,  ex: /var/opt/jfrog/artifactory/var/logs on the docker host.

    Command example

    'docker run -it --name jfrog-fluentd-datadog-rt -v $JFROG_HOME/artifactory/var/:/var/opt/jfrog/artifactory --env-file Dockerenvfile.txt jfrog/fluentd-datadog-rt'


```

### Kubernetes Deployment with Helm
Recommended installation for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product        | Example Values File             |
|----------------|---------------------------------|
| Artifactory    | helm/artifactory-values.yaml    |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray           | helm/xray-values.yaml           |

Add JFrog Helm repository:

```text
helm repo add jfrog https://charts.jfrog.io
helm repo update
```
Replace placeholders with your ``masterKey`` and ``joinKey``. To generate each of them, use the command
``openssl rand -hex 32``

#### Artifactory ⎈:
For Artifactory installation, download the .env file from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/.env_jfrog). Fill in the .env_jfrog file with correct values.

* **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying docker images. Not a required field for k8s installation
* **DATADOG_API_KEY**: APIkey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
* **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
* **COMMON_JPD**: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

Apply the .env files and then run the helm command below

````text
source .env_jfrog
````
```text
helm upgrade --install artifactory  jfrog/artifactory \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set datadog.api_key=$DATADOG_API_KEY  \
       --set jfrog.observability.jpd_url=$JPD_URL \
       --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
       --set jfrog.observability.access_token=$JFROG_ADMIN_TOKEN \
       --set jfrog.observability.common_jpd=$COMMON_JPD \
       -f helm/artifactory-values.yaml
```

#### Artifactory-HA ⎈:
For HA installation, please create a license secret on your cluster prior to installation.

```text
kubectl create secret generic artifactory-license --from-file=<path_to_license_file>artifactory.cluster.license 
```
Download the .env file from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/.env_jfrog). Fill in the .env_jfrog file with correct values.

* **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying docker images. Not a required field for k8s installation
* **DATADOG_API_KEY**: APIkey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
* **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
* **COMMON_JPD**: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

Apply the .env files and then run the helm command below

````text
source .env_jfrog
````
```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set datadog.api_key=$DATADOG_API_KEY  \
       --set jfrog.observability.jpd_url=$JPD_URL \
       --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
       --set jfrog.observability.access_token=$JFROG_ADMIN_TOKEN \
       --set jfrog.observability.common_jpd=$COMMON_JPD \
       -f helm/artifactory-ha-values.yaml
```

#### Xray ⎈:
For Artifactory installation, download the .env file from [here](https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/.env_jfrog). Fill in the .env_jfrog file with correct values.

* **JF_PRODUCT_DATA_INTERNAL**: Helm based installs will already have this defined based upon the underlying docker images. Not a required field for k8s installation
* **DATADOG_API_KEY**: APIkey from [Datadog](https://docs.datadoghq.com/account_management/api-app-keys/)
* **JPD_URL**: Artifactory JPD URL of the format `http://<ip_address>`
* **JPD_ADMIN_USERNAME**: Artifactory username for authentication
* **JFROG_ADMIN_TOKEN**: Artifactory [Access Token](https://jfrog.com/help/r/how-to-generate-an-access-token-video/artifactory-creating-access-tokens-in-artifactory) for authentication
* **COMMON_JPD**: This flag should be set as true only for non-kubernetes installations or installations where JPD base URL is same to access both Artifactory and Xray (ex: https://sample_base_url/artifactory or https://sample_base_url/xray)

Apply the .env files and then run the helm command below

````text
source .env_jfrog
````

Use the same `joinKey` as you used in Artifactory installation to allow Xray node to successfully connect to Artifactory.

```text
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set datadog.api_key=$DATADOG_API_KEY  \
       --set jfrog.observability.jpd_url=$JPD_URL \
       --set jfrog.observability.username=$JPD_ADMIN_USERNAME \
       --set jfrog.observability.access_token=$JFROG_ADMIN_TOKEN \
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

#### JFrog Xray Metrics Dashboard
This dashboard tracks System Metrics, and data metrics about Scanned Artifacts and Scanned Components

#### JFrog Xray Violations Dashboard
This dashboard provides an aggregated summary of all the license violations and security vulnerabilities found by Xray. Information is segment by watch policies and rules. Trending information is provided on the type and severity of violations over time, as well as, insights on most frequently occurring CVEs, top impacted artifacts and components.

## Demo Requirements

* Kubernetes Cluster
* Artifactory and/or Xray installed via [JFrog Helm Charts](https://github.com/jfrog/charts)
* Helm 3

## Generating Data for Testing

[Partner Integration Test Framework](https://github.com/jfrog/partner-integration-tests) can be used to generate data for metrics.

## References

* [Datadog](https://docs.datadoghq.com/getting_started/) - Cloud monitoring as a service
