# Dockerfile for bitnami/fluentd sidecar image with all the necessary plugins for our log analytic providers
FROM bitnami/fluentd:latest
LABEL maintainer="Partner Engineering <partner_support@jfrog.com>"

## Build time Arguments, short circuit them to ENV Variables so they are available at run time also
ARG SOURCE=JFRT
ARG TARGET=DATADOG

## Environment Variables set by this docker file, there will be seperate env params set by a env file while running the containers
## For better maintainability always depend dockerfile code on the environment variables declared in this file to add more platforms
ENV SRC_PLATFORM=$SOURCE
ENV TGT_PLATFORM=$TARGET

USER root

## Install JFrog Plugins
RUN fluent-gem install fluent-plugin-jfrog-siem
RUN fluent-gem install fluent-plugin-jfrog-metrics

## Install custom Fluentd plugins
RUN if [ "$TGT_PLATFORM" = "DATADOG" ] ; then echo "Downloading the fluentd plugin for $TGT_PLATFORM "; fluent-gem install fluent-plugin-datadog; else echo "Not Downloading"; fi

## Download Config Files
RUN if [ "$SRC_PLATFORM" = "JFRT" ] ; then echo "Downloading the fluentd config file for $SRC_PLATFORM and $TGT_PLATFORM "; curl https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.rt -o /opt/bitnami/fluentd/conf/fluentd.conf; else echo "Not Downloading"; fi
RUN if [ "$SRC_PLATFORM" = "JFXRAY" ] ; then echo "Downloading the fluentd config file for $SRC_PLATFORM and $TGT_PLATFORM "; curl https://raw.githubusercontent.com/jfrog/log-analytics-datadog/master/fluent.conf.xray -o /opt/bitnami/fluentd/conf/fluentd.conf; else echo "Not Downloading"; fi


ENTRYPOINT if [ "$TGT_PLATFORM" = "DATADOG" ] ; then  sed -i
                                             -e "s/DATADOG_API_KEY/$DATADOG_API_KEY/g" \
                                             -e "s/JPD_URL/$JPD_URL/" \
                                             -e "s/JPD_USER_NAME/$JPD_USER_NAME/" \
                                             -e "s/JPD_API_KEY/$JPD_API_KEY/" /opt/bitnami/fluentd/conf/fluentd.conf && fluentd -v -c /opt/bitnami/fluentd/conf/fluentd.conf; fi

USER 1001

STOPSIGNAL SIGTERM