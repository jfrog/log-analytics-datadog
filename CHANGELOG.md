# JFrog Log Analytics Changelog

All changes to the log analytics integration will be documented in this file.

## [1.0.18] - August 2026

* Added log collection for the `frontend`, `jfbus` and `jfmelt` services, which Artifactory 7.161.x deploys as standalone pods instead of containers inside the Artifactory StatefulSet - on 7.161.x the Artifactory pod's sidecar cannot see their log files, so none of them were collected. `frontend-request.log` is still collected by `fluent.conf.rt` on earlier versions; it only stops being produced in the Artifactory pod once frontend moves out (JOBS-2792)
* New logs-only fluentd configs `fluent.conf.rt.frontend`, `fluent.conf.rt.jfbus` and `fluent.conf.rt.jfmelt`, each collecting its own service and request logs plus `router-service.log`, `router-request.log` and `router-traefik.log`; the `concat` filter inherited unchanged from `fluent.conf.rt` also folds the bare ` - <message>` continuation line that jfbus 1.386.17 emits after every service-log line, so no jfbus-specific handling was needed
* `helm/artifactory-values.yaml` and `helm/artifactory-ha-values.yaml` now carry `customInitContainers` / `customSidecarContainers` blocks for the `frontend`, `jfbus` and `jfmelt` pods - no new `--set` flags are needed, the documented `helm upgrade` commands pick them up as-is. Init and sidecar containers inherit each service's `containerSecurityContext` when it is enabled
* All three sidecars ship with the same `dd_source jfrog_platform` and `service jfrog_artifactory` as the Artifactory sidecar, so existing dashboard filters keep matching; `hostname` identifies the originating pod, while `log_source` is the fluentd tag and so names the log file. Note that router log volume under `service:jfrog_artifactory` rises on upgrade, as `router-service.log`, `router-request.log` and `router-traefik.log` now arrive from four pods instead of one
* Platform metrics (`jfrog_metrics` / `jfrog_send_metrics`) and callhome remain in the Artifactory pod's `fluent.conf.rt` only, so they are still collected once per JPD rather than once per pod
* No fluentd sidecar image change - `releases-docker.jfrog.io/fluentd:4.22` already ships every plugin these configs require
* Requires Artifactory 7.161.x or later; on earlier versions the new values-file blocks are inert. Chart 107.161.x also blocks `splitServicesToContainers: false`
* Logs in the `frontend`, `jfbus` and `jfmelt` pods live on an `emptyDir`, so log files and fluentd position files are lost on pod restart
* Deliberately not collected: `jfbus-publish-events.log`, `jfbus-consume-events.log`, `jfbus-ack-events.log`, `jfbus-receive-polling.log` and `jfmelt-metrics.log`; `jfmelt-request-out.log` is collected but ships unparsed as a raw message, as `router-request.log` already does. `observability-service.log`, which the `observability` container in each of the three pods also writes, is not collected either

## [1.0.17] - June 2026

* Fluentd sidecar image bumped to 4.22: now consumes the released image `releases-docker.jfrog.io/fluentd:4.22` directly (fluentd 1.19.3 on the refreshed hardened Echo base), remediating the critical OS-package vulnerability CVE-2026-55200 in libssh2 (1.11.1-1+e1 -> 1.11.1-1+e2) carried by the 4.21 base image (JOBS-2583)
* `docker-build/Dockerfile` now builds `FROM releases-docker.jfrog.io/fluentd:4.22`

## [1.0.16] - June 2026

* Fluentd sidecar image bumped to 4.21: now consumes the released image `releases-docker.jfrog.io/fluentd:4.21` directly (fluentd 1.19.2 on a hardened, CVE-refreshed base), remediating Critical/High CVEs in 4.19 (JOBS-2475)
* `docker-build/Dockerfile` now builds `FROM releases-docker.jfrog.io/fluentd:4.21`, which already bundles curl and every required plugin (fluent-plugin-datadog, -jfrog-siem, -jfrog-metrics, -jfrog-send-metrics, -concat), so the per-plugin `fluent-gem install` and `apt-get` steps were removed

## [1.0.15] - April 2026

* Fix incorrect field name in access-security-audit log parsing: renamed `token_id` capture group to `trace_id` to match the actual log format documented at https://docs.jfrog.com/administration/docs/audit-trail-log (JOBS-2031)

## [1.0.14] - April 2026

* Added RTFS (JFrog Artifactory Federation Service) metrics collection support in Artifactory fluentd config (JOBS-1897)
* Requires fluent-plugin-jfrog-metrics >= 0.2.16

## [1.0.13] - March 18, 2025

* Update artifactory-ha helm values file
* Readme minor updates

## [1.0.12] - January 2, 2025

* FluentD sidecar image version bumped to 4.15, to upgrade base image to bitnami/fluentd 1.18.0

## [1.0.11] - November 19, 2024

* FluentD sidecar image version bumped to 4.14, to reflect logging improvements in `jfrog_metrics` and `jfrog_send_metrics` FluentD plugins 

## [1.0.10] - November 7, 2024

* FluentD sidecar image version bumped to 4.13, to reflect changes in `jfrog_siem` and `jfrog_send_metrics` FluentD plugins 

## [1.0.9] - October 25, 2024

* Add support for metrics outbound payload compression, with `gzip_compression` FluentD param in `jfrog_send_metrics` plugin
* Add support for a configurable http request timeout, with `request_timeout` FluentD param in `jfrog_metrics` and `jfrog_send_metrics` plugins
* FluentD sidecar version bumped to 4.9, to incorporate the above changes
* Add configuration support via environment variable for `verify_ssl` FluentD flag

## [1.0.8] - September 12, 2024

* FluentD sidecar image version bumped to 4.8, to add verify_ssl flag support for JFrog's FluentD metrics plugins

## [1.0.7] - August 19, 2024

* FluentD sidecar image version bumped to 4.7, to add http proxy support for JFrog's FluentD metrics plugin

## [1.0.6] - August 7, 2024

* Fix metrics configuration due to deprication of `artifactory.openMetrics` as part of Artifactory 7.87.x charts and renaming it to `artifactory.metrics`

## [1.0.5] - July 16, 2024

* Fluentd sidecar version bumped to 4.5, to upgrade base image to bitnami/fluentd 1.17.0
* Fixing fluent-plugin-jfrog-metrics issue (upgrading to 0.2.7) - resolving PTRENG-6234
* Metrics documentation changes - resolving PTRENG-6186

## [1.0.4] - June 6, 2024

* [BREAKING] Adding deprecation notice for partnership-pts-observability.jfrog.io docker registry
* FluentD sidecar version bumped to 4.3, to upgrade base image to bitnami/fluentd 1.16.5
* Minor bug fix to FluentD config - fixing dynamic DataDog host config for logs
* Update FluentD sidecar helm charts to match recent changes in JFrog's official charts

## [1.0.3] - April 23, 2024

* Fix order of request and response content length to match spec
* Add a new environment variable to support DataDog host configuration in Fluentd for log and metrics endpoints

## [1.0.2] - April 12, 2024

* Fluentd version bumped to 4.2, which has latest Fluentd plugins. Resolved PTRENG-5895

## [1.0.1] - April 11, 2024

* Fix Artifactory access's regex to match log input changes

## [1.0.0] - Jun 22, 2023

* Supporting only OS/VM, Docker and k8s installation types
* Adding .env files instead of setting/filling variables in fluentd config
* Adding jfrog and heap callhome in fluentd config
* Supporting only Artifactory and Xray Fluentd config

## [0.8.0] - Feb 09, 2022

* Added call home functionality to artifactory fluent configuration

## [0.7.0] - Oct 20, 2020

* Fixing issue with ip_address in access logs having space and . at the end

## [0.6.0] - Sept 25, 2020

* [BREAKING] Datadog fluentd configs updated to use JF_PRODUCT_DATA_INTERNAL env

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
