FROM public.ecr.aws/sumologic/kubernetes-fluentd:latest-alpine@sha256:dc65b35632c35168ae181a8886dd0269ca80e4bdda852b63081ba110538f3dc3 AS sumo

# Adapted from https://github.com/SumoLogic/sumologic-kubernetes-fluentd/blob/main/alpine.Dockerfile#L102C1-L142C16
FROM ruby:3.3.5-alpine3.19 as base

ARG BUILD_DEPS=" \
      make gcc g++ libc-dev \
      wget bzip2 zlib-dev git linux-headers \
      automake autoconf libtool build-base \
      ruby-dev libc6-compat geoip-dev \
      snappy-dev gnupg \
      "

RUN addgroup -S -g 101 fluent && adduser -S -G fluent -u 100 fluent \
    # for log storage (maybe shared with host)
    && mkdir -p /fluentd/log \
    # configuration/plugins path (default: copied from .)
    && mkdir -p /fluentd/etc /fluentd/plugins \
    && chown -R fluent /fluentd && chgrp -R fluent /fluentd \
    && mkdir -p /buffers && chown -R fluent /buffers

COPY --from=sumo --chown=fluent:fluent /usr/local/bundle /usr/local/bundle

RUN apk add --no-cache \
        ca-certificates \
        ruby ruby-irb ruby-etc \
        tini libmaxminddb geoip \
        snappy \
 && apk add --no-cache $BUILD_DEPS \
 && echo 'gem: --no-document' >> /etc/gemrc \
 # need to rebuild these gems to have the extensions installed
 && fluent-gem install \
         bigdecimal:1.4.4 \
         oj:3.15.0 \
         google-protobuf:3.21.12 \
         nio4r:2.6.0 \
         snappy:0.3.0 \
 # The only required fluentd dependency is the label router
 # kubeclient install is upgrading the one from the base image
 && fluent-gem install specific_install -v 0.3.8 \
 && fluent-gem specific_install -l https://github.com/ManageIQ/kubeclient.git --ref 054bff2c5e31a555004be2b3c4d32fb9dc5e6a0f \
 && fluent-gem specific_install -l https://github.com/kube-logging/fluent-plugin-label-router.git --ref 2ff43789f895735b7852e0a1a809280e22d9e8ef \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

COPY fluent.conf /fluentd/etc/
COPY entrypoint.sh /bin/
COPY healthy.sh /bin/

ENV FLUENTD_CONF="fluent.conf"
ENV LD_PRELOAD=""
EXPOSE 24224 5140

USER fluent

ENTRYPOINT ["tini",  "--", "/bin/entrypoint.sh"]
CMD ["fluentd"]

### Image with all the filters
FROM base AS filters

ADD ./filters/Gemfile /Gemfile.filters
ADD ./filters/Gemfile.lock /Gemfile.filters.lock

USER root

ENV BUNDLE_WITHOUT="output,test,development"

RUN apk add --no-cache $BUILD_DEPS \
 && touch /etc/gemrc \
 && fluent-gem install --file Gemfile.filters \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

USER fluent

### Image with all the filters and outputs
FROM filters as full

ADD ./outputs/Gemfile /Gemfile.outputs
ADD ./outputs/Gemfile.lock /Gemfile.outputs.lock

USER root

RUN apk add --no-cache $BUILD_DEPS \
 && touch /etc/gemrc \
 && fluent-gem specific_install -l https://github.com/kube-logging/fluent-plugin-gcs.git --ref 71a0932e75509545a2d28e337642ee1f973cca90 \
 && fluent-gem specific_install -l https://github.com/acquia/fluent-plugin-syslog_rfc5424.git --ref 5199be67d1a385f529fa1d6b6023e95ba7fac27d \
 && fluent-gem install --file /Gemfile.outputs \
 && find /usr/local/bundle/gems/ -newer /etc/gemrc -exec chown fluent:fluent {} \; \
 && apk del $BUILD_DEPS \
 && rm -rf /usr/local/bundle/cache/* && find /usr/local/bundle -name '*.o' -delete

USER fluent
