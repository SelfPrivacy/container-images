# Build stage for sonarqube-community-branch-plugin (Java plugin)
FROM gradle:8.9-jdk17-jammy AS plugin-builder

ARG COMMUNITY_BRANCH_PLUGIN_GITREF=master
ARG WORKDIR=/home/build/project

WORKDIR ${WORKDIR}
RUN git clone --recurse-submodules https://github.com/mc1arke/sonarqube-community-branch-plugin.git . && \
    git checkout ${COMMUNITY_BRANCH_PLUGIN_GITREF} && \
    git submodule update --init --recursive

RUN gradle build -x test --no-daemon

# Build stage for sonarqube-webapp
FROM node:22-alpine AS webapp-builder

ARG COMMUNITY_BRANCH_PLUGIN_GITREF=master
ARG WORKDIR=/home/build/project

WORKDIR ${WORKDIR}
RUN apk add --no-cache git
RUN git clone --recurse-submodules https://github.com/mc1arke/sonarqube-community-branch-plugin.git . && \
    git checkout ${COMMUNITY_BRANCH_PLUGIN_GITREF} && \
    git submodule update --init --recursive

WORKDIR ${WORKDIR}/sonarqube-webapp
RUN cp -r ${WORKDIR}/sonarqube-webapp-addons ${WORKDIR}/sonarqube-webapp/libs/sq-server-addons
RUN yarn install
RUN yarn nx run sq-server:build

# Main image
FROM docker.io/sonarqube:25.11.0.114957-community

USER root

ARG FLUTTER_VERSION=3.32.6
ARG FLUTTER_URL=https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_${FLUTTER_VERSION}-stable.tar.xz
ARG FLUTTER_ANALYZER_VERSION=0.5.0
ARG FLUTTER_ANALYZER_URL=https://github.com/insideapp-oss/sonar-flutter/releases/download/${FLUTTER_ANALYZER_VERSION}/sonar-flutter-plugin-${FLUTTER_ANALYZER_VERSION}.jar
ARG FLUTTER_HOME=${SONARQUBE_HOME}/flutter

RUN apt-get update && apt-get install -y xz-utils python3-pip git python3-bandit

ADD $FLUTTER_URL ${SONARQUBE_HOME}/
ADD --chown=sonarqube:sonarqube $FLUTTER_ANALYZER_URL ${SQ_EXTENSIONS_DIR}/plugins/

# Copy the built community branch plugin and rename to consistent name
COPY --from=plugin-builder /home/build/project/build/libs/sonarqube-community-branch-plugin-*.jar /tmp/
RUN mv /tmp/sonarqube-community-branch-plugin-*.jar ${SQ_EXTENSIONS_DIR}/plugins/sonarqube-community-branch-plugin.jar && \
    chown sonarqube:sonarqube ${SQ_EXTENSIONS_DIR}/plugins/sonarqube-community-branch-plugin.jar

# Copy the built webapp to replace the default web directory
RUN rm -rf ${SONARQUBE_HOME}/web/*
COPY --from=webapp-builder --chown=sonarqube:sonarqube /home/build/project/sonarqube-webapp/apps/sq-server/build/webapp/ ${SONARQUBE_HOME}/web/
RUN chmod -R 550 ${SONARQUBE_HOME}/web

RUN tar -vxf flutter_linux_${FLUTTER_VERSION}-stable.tar.xz && \
    rm flutter_linux_${FLUTTER_VERSION}-stable.tar.xz

RUN chown -R root:root flutter/

RUN ${FLUTTER_HOME}/bin/flutter doctor

RUN chown -R sonarqube flutter/

USER sonarqube

ENV FLUTTER_HOME=${FLUTTER_HOME} FLUTTER_ROOT=${FLUTTER_HOME} PATH="$PATH:${FLUTTER_HOME}/bin"
ENV SONAR_WEB_JAVAADDITIONALOPTS="-javaagent:./extensions/plugins/sonarqube-community-branch-plugin.jar=web"
ENV SONAR_CE_JAVAADDITIONALOPTS="-javaagent:./extensions/plugins/sonarqube-community-branch-plugin.jar=ce"
