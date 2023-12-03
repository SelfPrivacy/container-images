FROM docker.io/sonarqube:lts

USER root

ARG FLUTTER_VERSION=3.16.1
ARG FLUTTER_URL=https://storage.googleapis.com/flutter_infra_release/releases/stable/linux/flutter_linux_${FLUTTER_VERSION}-stable.tar.xz
ARG FLUTTER_ANALYZER_VERSION=0.5.0
ARG FLUTTER_ANALYZER_URL=https://github.com/insideapp-oss/sonar-flutter/releases/download/${FLUTTER_ANALYZER_VERSION}/sonar-flutter-plugin-${FLUTTER_ANALYZER_VERSION}.jar
ARG FLUTTER_HOME=${SONARQUBE_HOME}/flutter

RUN apt-get update && apt-get install -y xz-utils python3-pip git

RUN pip install bandit

ADD $FLUTTER_URL ${SONARQUBE_HOME}/
ADD --chown=sonarqube:sonarqube $FLUTTER_ANALYZER_URL ${SQ_EXTENSIONS_DIR}/plugins/

RUN tar -vxf flutter_linux_${FLUTTER_VERSION}-stable.tar.xz && \
    rm flutter_linux_${FLUTTER_VERSION}-stable.tar.xz

RUN ${FLUTTER_HOME}/bin/flutter doctor

RUN chown -R sonarqube:sonarqube flutter/

USER sonarqube

ENV FLUTTER_HOME=${FLUTTER_HOME} FLUTTER_ROOT=${FLUTTER_HOME} PATH="$PATH:${FLUTTER_HOME}/bin"
