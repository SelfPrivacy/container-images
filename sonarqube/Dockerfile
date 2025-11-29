# Clone stage for sonarqube-community-branch-plugin source
# Note: Using master branch to get latest updates for newer SonarQube versions.
# For production, consider pinning to a specific commit hash for reproducibility.
FROM alpine:3.20 AS source

ARG COMMUNITY_BRANCH_PLUGIN_GITREF=master

RUN apk add --no-cache git
WORKDIR /source
# Configure git to use HTTPS instead of SSH for GitHub URLs (submodules use SSH URLs)
RUN git config --global url."https://github.com/".insteadOf "git@github.com:" && \
    git clone --recurse-submodules https://github.com/mc1arke/sonarqube-community-branch-plugin.git . && \
    git checkout ${COMMUNITY_BRANCH_PLUGIN_GITREF} && \
    git submodule update --init --recursive

# Build stage for sonarqube-community-branch-plugin (Java plugin)
FROM gradle:8.9-jdk17-jammy AS plugin-builder

COPY --from=source /source /home/build/project
WORKDIR /home/build/project
RUN gradle build -x test --no-daemon

# Build stage for sonarqube-webapp
FROM node:22-alpine AS webapp-builder

# Copy only the webapp directory (not the entire source to avoid duplicate project definitions)
COPY --from=source /source/sonarqube-webapp /home/build/project
# Copy the addons contents into the libs directory
COPY --from=source /source/sonarqube-webapp-addons /home/build/project/libs/sq-server-addons
WORKDIR /home/build/project
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
ADD --chown=sonarqube $FLUTTER_ANALYZER_URL ${SQ_EXTENSIONS_DIR}/plugins/

# Copy the built community branch plugin and rename to consistent name
# The build produces a single JAR with version suffix; we rename it for predictable ENV variable references
COPY --from=plugin-builder /home/build/project/build/libs/sonarqube-community-branch-plugin-*.jar /tmp/
RUN JAR_COUNT=$(ls -1 /tmp/sonarqube-community-branch-plugin-*.jar 2>/dev/null | wc -l) && \
    if [ "$JAR_COUNT" -ne 1 ]; then echo "Error: Expected 1 JAR file, found $JAR_COUNT" && exit 1; fi && \
    mv /tmp/sonarqube-community-branch-plugin-*.jar ${SQ_EXTENSIONS_DIR}/plugins/sonarqube-community-branch-plugin.jar && \
    chown sonarqube ${SQ_EXTENSIONS_DIR}/plugins/sonarqube-community-branch-plugin.jar

# Replace the default web directory with the plugin's modified webapp
# This is required for branch/PR UI functionality as per plugin installation instructions
RUN rm -rf ${SONARQUBE_HOME}/web/*
COPY --from=webapp-builder --chown=sonarqube /home/build/project/apps/sq-server/build/webapp/ ${SONARQUBE_HOME}/web/
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
