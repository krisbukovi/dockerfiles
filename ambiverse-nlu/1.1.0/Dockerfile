FROM maven:3.5.0-jdk-8

MAINTAINER AmbiverseNLU <ambiversenlu-admin@mpi-inf.mpg.de>

ARG TAG=ambiversenlu-1.1.0

ENV AIDA_CONF=default

RUN java -version

WORKDIR /ambiverse-nlu

RUN git clone --branch ${TAG} https://github.com/ambiverse-nlu/ambiverse-nlu.git /ambiverse-nlu

RUN sed -i '/^log4j.appender.FILE.File=.*/c\log4j.appender.FILE.File=/ambiverse-nlu/logs/ambiverse-nlu.log' /ambiverse-nlu/src/main/resources/log4j.properties

RUN sed -i '/^log4j.appender.requestLog.File=.*/c\log4j.appender.requestLog.File=/ambiverse-nlu/logs/requests.log' /ambiverse-nlu/src/main/resources/log4j.properties

ENV MAVEN_OPTS -Djetty.port=8080 -Xmx64G -Dorg.eclipse.jetty.annotations.maxWait=180
ENTRYPOINT ["mvn", "jetty:run"]

EXPOSE 8080