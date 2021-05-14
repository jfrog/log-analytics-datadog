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

## Fluentd Install

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

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above:

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
Run the fluentd wrapper with one argument pointed to the configuration file to load:

````text
./fluentd test.conf
````

Next steps are to setup a  `fluentd.conf` file using the relevant integrations for Splunk, DataDog, Elastic, or Prometheus.

### Docker

Recommended install for Docker is to utilize the zip installer for Linux

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory the user has permissions to write such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above:

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
Run the fluentd wrapper with one argument pointed to the configuration file to load:

````text
./fluentd test.conf
````

Next steps are to setup a  `fluentd.conf` file using the relevant configuration files for Datadog.

### Kubernetes

Recommended installation for Kubernetes is to utilize the helm chart with the associated values.yaml in this repo.

| Product | Example Values File |
|---------|-------------|
| Artifactory | helm/artifactory-values.yaml |
| Artifactory HA | helm/artifactory-ha-values.yaml |
| Xray | helm/xray-values.yaml |


//TODO: HOW to modify the yaml?  


Update the values.yaml associated to the product you want to deploy with your Datadog settings.

Then deploy the helm chart as described below:

Note: To generate ``masterKey`` and ``joinKey``, use the command
``openssl rand -hex 32``

Artifactory ⎈:
```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/artifactory-values.yaml
```

Artifactory-HA ⎈:

For HA installation, please create a license secret on your cluster prior to installation.

```text
helm upgrade --install artifactory-ha  jfrog/artifactory-ha \
       --set artifactory.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set artifactory.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       --set artifactory.license.secret=<license_secret_name> \
       --set artifactory.license.dataKey=artifactory.cluster.license \
       -f helm/artifactory-ha-values.yaml
```

Xray ⎈:
```text
helm upgrade --install xray jfrog/xray --set xray.jfrogUrl=http://my-artifactory-nginx-url \
       --set xray.masterKey=FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF \
       --set xray.joinKey=EEEEEEEEEEEEEEEEEEEEEEEEEEEEEEEE \
       -f helm/xray-values.yaml
```

#### Kubernetes Deployment without Helm

To modify existing Kubernetes based deployments without using Helm users can use the zip installer for Linux:

| OS            | Package Manager | Link |
|---------------|-----------------|------|
| Linux (x86_64)| ZIP             | https://github.com/jfrog/log-analytics/raw/master/fluentd-installer/fluentd-1.11.0-linux-x86_64.tar.gz |

Download it to a directory, where the user has permissions to write, such as the `$JF_PRODUCT_DATA_INTERNAL` locations discussed above:

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
Run the fluentd wrapper with one argument pointed to the configuration file to load:

````text
./fluentd test.conf
````

Next steps are to setup a  `fluentd.conf` file using the relevant configuration files for Datadog.

### Fluentd Configuration for Datadog

#### Download fluentd.conf
Artifactory: 
````text
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.rt
````

Xray:
````text
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.xray
````

Mision Control:
````text
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.missioncontrol
````

Distribution:
````text
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.distribution
````

Pipelines:
````text
wget https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.pipelines
````

#### Configure fluentd.conf

Integration is done by specifying the apiKey

```API_KEY``` is the apiKey from Datadog

```include_tag_key``` defaults to false. It will add fluentd tag in the json record if set to true

```dd_source``` attribute is set to the name of the log integration in your logs in order to trigger the integration automatic setup in datadog.


These values override the last section of the `fluentd.conf` shown below:

```
<match jfrog.**>
  @type datadog
  @id datadog_agent_artifactory
  api_key API_KEY
  include_tag_key true
  dd_source fluentd
</match>
```

Instructions to run fluentd configuration files can be found at JFrog log analytic repository's [README.](https://github.com/jfrog/log-analytics/blob/master/README.md)

## Datadog Setup

Datadog setup can be done by going through the below onboarding steps or by using apiKey directly if one exists. If an apiKey exists, use the Datadog Fluentd plugin to forward logs directly from Fluentd to your datadog account. 

* Create an account in Datadog
* Run the datadog agent in your kubernetes cluster by deploying it with a helm chart
* To enable log collection, update datadog-values.yaml file given in the onboarding steps of datadog
* Once the agent starts reporting, you'll get an apiKey which we'll be using to send formatted logs through fluentd

Once datadog is setup, host and port should be configured in fluent.conf and td-agent can be started. We can access logs via Logs > Search. We can also select the specific source that we want to get logs from. Adding proper metadata is the key to unlocking the full potential of your logs in datadog. By default, the hostname and timestamp fields should be remapped.

* Add all attributes as facets from Facets > Add on the left side of the screen in Logs > search

### Dashboard
Dashboard is divided into three sections Application, Audit and Requests

* **Application** - This section tracks Log Volume(information about different log sources) and Artifactory Errors over time(bursts of application errors that may otherwise go undetected)
* **Audit** - This section tracks audit logs help you determine who is accessing your Artifactory instance and from where. These can help you track potentially malicious requests or processes (such as CI jobs) using expired credentials.
* **Requests** - This section tracks HTTP response codes, Top 10 IP addresses for uploads and downloads

## Demo Requirements

* Kubernetes Cluster
* Artifactory and/or Xray installed via [JFrog Helm Charts](https://github.com/jfrog/charts)
* Helm 3

## Generating Data for Testing
[Partner Integration Test Framework](https://github.com/jfrog/partner-integration-tests) can be used to generate data for metrics.

## References
* [Datadog](https://docs.datadoghq.com/getting_started/) - Cloud monitoring as a service
