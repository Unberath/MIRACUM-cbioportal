# This image includes both the web app and the data import related scripts in
# one image. Because of historic reasons some of these scripts run java
# artifacts under the hood. It therefore currently makes more sense to have a
# single image supporting both instead of moving the scripts out to a separate
# container.
#
# Use from root directory of repo like:
#
# docker build -f docker/web-and-data/Dockerfile -t cbioportal-container:tag-name .
#
# NOTE: the .git folder is included in the build stage, but excluded 
# from the final image. No confidential information is exposed.
# (see: stackoverflow.com/questions/56278325)
FROM maven:3-openjdk-11 as build

RUN git clone --single-branch --branch therapy_recommendation https://github.com/nr23730/cbioportal.git /cbioportal
WORKDIR /cbioportal
RUN mvn -DskipTests clean install
RUN unzip /cbioportal/portal/target/cbioportal*.war -d /unzipped

FROM alpine:3.12.1

# create user for not running image with root privileges
RUN addgroup -g 1001 -S cbioportal && adduser -u 1001 -S cbioportal -G cbioportal

# download system dependencies first to take advantage of docker caching
RUN apk add --update \
        bash \
        build-base \
        gcc \
        mariadb-client \
        mariadb-dev \
        openjdk11-jre-headless \
        perl \
        py3-pip \
        python3 \
        python3-dev \
    && rm -rf /var/cache/apk/* \
    && pip3 install wheel

ENV PORTAL_WEB_HOME=/cbioportal-webapp
ENV JAVA_HOME=/usr/lib/jvm/java-11-openjdk

RUN mkdir -p /cbioportal && chown -R cbioportal:cbioportal /cbioportal
RUN mkdir -p /target/tomcat.8080 && chown -R cbioportal:cbioportal /target/tomcat.8080

COPY --from=build /cbioportal/portal/target/dependency/webapp-runner.jar /webapp-runner.jar
# copy over core jar and scripts
COPY --from=build /cbioportal/core /cbioportal/core
COPY --from=build /cbioportal/scripts /cbioportal/scripts
COPY --from=build /cbioportal/db-scripts /cbioportal/db-scripts
COPY --from=build /cbioportal/requirements.txt /cbioportal/requirements.txt

# install build and runtime dependencies
# ignore update failures
RUN  pip3 install -r /cbioportal/requirements.txt

# add importer scripts to PATH for easy running in containers
RUN find /cbioportal/core/src/main/scripts/ -type f -executable \! -name '*.pl'  -print0 | xargs -i -- ln -s {} /usr/local/bin

# put config files in this folder if you want to override config
ENV PORTAL_HOME=/cbioportal

RUN mkdir -p $PORTAL_WEB_HOME
COPY --from=build /unzipped $PORTAL_WEB_HOME

# add entrypoint
COPY --from=build /cbioportal/docker/web-and-data/docker-entrypoint.sh /usr/local/bin/

# script for calling dumpPortalInfo.py from docker-compose
RUN echo "cd /cbioportal/core/src/main/scripts && ./dumpPortalInfo.pl \$1" >> /cbioportal/dumpPortalInfo.sh

# switch to non-root user
USER cbioportal
ENTRYPOINT ["docker-entrypoint.sh"]
