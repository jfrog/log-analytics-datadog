# JFrog Log Analytics Changelog

All changes to the log analytics integration will be documented in this file.

## [1.0.2] - April 12, 2024

* Fluentd version bumped to 4.2, which has latest Fluentd plugins. Resolved PTRENG-5895. 

## [1.0.1] - April 11, 2024

* Fix Artifactory access's regex to match log input changes

## [1.0.0] - Jun 22, 2023

* Supporting only OS/VM, Docker and k8s installation types
* Adding .env files instead of setting/filling variables in fluentd config
* Adding jfrog and heap callhome in fluentd config
* Supporting only Artifactory and Xray Fluentd config

## [0.8.0] - Feb 09, 2022

* Added call home functionality to artifactory fluent configuration.

## [0.7.0] - Oct 20, 2020

* Fixing issue with ip_address in access logs having space and . at the end

## [0.6.0] - Sept 25, 2020

* [BREAKING] Datadog fluentd configs updated to use JF_PRODUCT_DATA_INTERNAL env.

## [0.5.0] - Sept 8, 2020

* Adding JFrog Pipelines fluent configuration files to capture logs

## [0.4.0] - Sept 4, 2020

* Adding JFrog Mission Control fluent configuration files to capture logs

## [0.3.0] - Aug 26, 2020

* Adding JFrog Distribution fluent configuration files to capture logs

## [0.2.0] - Aug 24, 2020

* Splunk updates to launch new version of Splunkbase app v1.1.0

## [0.1.1] - June 1, 2020

* Removing the need for user to specify splunk host , user, and token twice
* Fixing issue with regex on the audit security log
* Fixed issue with the repo and image when not docker api url

## [0.1.0] - May 12, 2020

* Initial release of Jfrog Logs Analytic integration
