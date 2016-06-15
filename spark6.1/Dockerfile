FROM ubuntu:16.04

#use bash
RUN rm /bin/sh && ln -s /bin/bash /bin/sh

RUN cd /home

RUN apt-get update
RUN apt-get install -y software-properties-common python-software-properties wget
RUN apt-get install -y sudo
RUN apt-get install -y vim

# Install Java.
RUN \
  echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | debconf-set-selections && \
  add-apt-repository -y ppa:webupd8team/java && \
  apt-get update && \
  apt-get install -y oracle-java8-installer && \
  rm -rf /var/lib/apt/lists/* && \
  rm -rf /var/cache/oracle-jdk8-installer

# Define working directory.
#WORKDIR /data

# Define commonly used JAVA_HOME variable
#ENV JAVA_HOME /usr/lib/jvm/java-8-oracle

# Ubuntu16.04/Cassandra missing libraries
RUN wget "http://launchpadlibrarian.net/109052632/python-support_1.0.15_all.deb" \
	&& dpkg -i python-support_1.0.15_all.deb

#Scala
RUN useradd sparky
RUN usermod -a -G sudo sparky
RUN mkdir /home/sparky
RUN touch /home/sparky/.bashrc

RUN export SCALA_HOME=/usr/local/share/scala
RUN export PATH=$SCALA_HOME/bin:$PATH

RUN echo SCALA_HOME=/usr/local/share/scala >> /home/sparky/.bashrc
RUN echo PATH=\$SCALA_HOME/bin:\$PATH >> /home/sparky/.bashrc

RUN wget http://downloads.lightbend.com/scala/2.11.8/scala-2.11.8.tgz
RUN tar xvf scala-2.11.8.tgz
RUN mv scala-2.11.8 /usr/local/share/scala

# Spark
RUN apt-get update
RUN apt-get -y install maven
RUN wget http://d3kbcqa49mib13.cloudfront.net/spark-1.6.1.tgz
RUN tar xvf spark-1.6.1.tgz
RUN cd spark-1.6.1 \
	&& ./dev/change-scala-version.sh 2.11 \
	&&mvn -Pyarn -Phadoop-2.4 -Dscala-2.11 -DskipTests clean package
RUN mv spark-1.6.1 /usr/local/share/spark
RUN cp /usr/local/share/spark/conf/spark-env.sh.template /usr/local/share/spark/conf/spark-env.sh
RUN echo export SPARK_MASTER_PORT=7080 >> /usr/local/share/spark/conf/spark-env.sh

RUN export SPARK_HOME=/usr/local/share/spark
RUN export PATH=$SPARK_HOME/bin:$PATH

RUN echo SPARK_HOME=/usr/local/share/spark >> /home/sparky/.bashrc
RUN echo PATH=\$SPARK_HOME/bin:\$PATH >> /home/sparky/.bashrc
RUN chown -R sparky:sparky /home/sparky

RUN wget http://dl.bintray.com/spark-packages/maven/datastax/spark-cassandra-connector/1.6.0-s_2.11/spark-cassandra-connector-1.6.0-s_2.11.jar

# explicitly set user/group IDs
RUN groupadd -r cassandra --gid=999 && useradd -r -g cassandra --uid=999 cassandra

# grab gosu for easy step-down from root
ENV GOSU_VERSION 1.7
RUN set -x \
	&& apt-get update && apt-get install -y --no-install-recommends ca-certificates wget && rm -rf /var/lib/apt/lists/* \
	&& wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture)" \
	&& wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$(dpkg --print-architecture).asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4 \
	&& gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu \
	&& rm -r "$GNUPGHOME" /usr/local/bin/gosu.asc \
	&& chmod +x /usr/local/bin/gosu \
	&& gosu nobody true \
	&& apt-get purge -y --auto-remove ca-certificates wget

RUN apt-key adv --keyserver ha.pool.sks-keyservers.net --recv-keys 514A2AD631A57A16DD0047EC749D6EEC0353B12C

RUN echo 'deb http://www.apache.org/dist/cassandra/debian 35x main' >> /etc/apt/sources.list.d/cassandra.list

ENV CASSANDRA_VERSION 3.5

RUN apt-get update \
	&& apt-get install -y cassandra="$CASSANDRA_VERSION" \
	&& rm -rf /var/lib/apt/lists/*

# https://issues.apache.org/jira/browse/CASSANDRA-11661
RUN sed -ri 's/^(JVM_PATCH_VERSION)=.*/\1=25/' /etc/cassandra/cassandra-env.sh

ENV CASSANDRA_CONFIG /etc/cassandra

COPY docker-entrypoint.sh /docker-entrypoint.sh

RUN mkdir -p /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chown -R cassandra:cassandra /var/lib/cassandra "$CASSANDRA_CONFIG" \
	&& chmod 777 /var/lib/cassandra "$CASSANDRA_CONFIG"

VOLUME /var/lib/cassandra
ENTRYPOINT ["/docker-entrypoint.sh"]
# 7000: intra-node communication
# 7001: TLS intra-node communication
# 7199: JMX
# 9042: CQL
# 9160: thrift service
# 7077: spark master
# 7080: spark ui
EXPOSE 7000 7001 7077 7080 7199 9042 9160
CMD ["cassandra", "-f"]
