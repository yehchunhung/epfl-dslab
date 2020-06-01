FROM renku/singleuser:0.4.3-renku0.8.2

# Uncomment and adapt if code is to be included in the image
# COPY src /code/src

# Uncomment and adapt if your R or python packages require extra linux (ubuntu) software
# e.g. the following installs apt-utils and vim; each pkg on its own line, all lines
# except for the last end with backslash '\' to continue the RUN line
# 
# USER root
# RUN apt-get update && \
#    apt-get install -y --no-install-recommends \
#    apt-utils \
#    vim
# USER ${NB_USER}

# ARG values are only available during the docker image build and can be changed
# from the docker build command line with `--build-arg ARG_NAME=arg_value`
#
# * HIVE_JDBC_ARG is the url to the hiveserver2 on the Hadoop platform. It uses the
#   Zookeeper form of the url to find the address of the hive sever
#   * see: https://cwiki.apache.org/confluence/display/Hive/HiveServer2%2BClients#HiveServer2Clients-ConnectionURLs
ARG HIVE_JDBC_ARG="jdbc:hive2://iccluster059.iccluster.epfl.ch:2181,iccluster054.iccluster.epfl.ch:2181,iccluster044.iccluster.epfl.ch:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2"
ARG HADOOP_DEFAULT_FS_ARG="hdfs://iccluster044.iccluster.epfl.ch:8020"

# ENV values are available during the docker image build and as environment variables
# to applications inside the container when the container is running.
# They cannot be modified directly from the docker build command line.
# However, they can by set to ARG values.
ENV HIVE_JDBC=${HIVE_JDBC_ARG}
ENV HADOOP_DEFAULT_FS=${HADOOP_DEFAULT_FS_ARG}

USER root

# Install the pyhive, beeline and hdfs client dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends openjdk-8-jre-headless && \
    apt-get install -y --no-install-recommends libsasl2-dev libsasl2-2 libsasl2-modules-gssapi-mit && \
    apt-get clean 

# Install hadoop and hive packages (as root).
# This is optional, it is needed only if you want to access hive and hdfs via command lines.
# The packages contain both clients and servers. Only the clients are required, and we could
# further reduce the size of the docker image if we cherry pick them.
# The installation process consists of:
# - mkdir, cd: prepare the destination folder
# - wget: download the package installation files
# - tar xf: extract the packages into the destination folder
# - rm: clean up the installation files
# - sed: adapt the client configurations beeline-site.xml (hive), and core-site.xml (hdfs)
# - modify beeline to set the username in ~/.beeline/beelin-hs2-connection.xml

USER root

RUN  mkdir -p /usr/hdp/current && \
     cd /usr/hdp/current && \
     wget -q https://archive.apache.org/dist/hive/hive-3.1.0/apache-hive-3.1.0-bin.tar.gz && \
     wget -q https://archive.apache.org/dist/hadoop/core/hadoop-3.1.0/hadoop-3.1.0.tar.gz && \
     tar --no-same-owner -xf apache-hive-3.1.0-bin.tar.gz && \
     tar --no-same-owner -xf hadoop-3.1.0.tar.gz && \
     rm apache-hive-3.1.0-bin.tar.gz && \
     rm hadoop-3.1.0.tar.gz && \
     echo '<configuration  xmlns:xi=“http://www.w3.org/2001/XInclude”><property><name>beeline.hs2.jdbc.url.container</name><value>HIVE_JDBC</value></property><property><name>beeline.hs2.jdbc.url.default</name><value>container</value></property></configuration>' >> /tmp/beeline-site.xml && \
     echo '<?xml version=“1.0” encoding=“UTF-8"?>' >> /tmp/core-site.xml && \
     echo '<?xml-stylesheet type=“text/xsl” href=“configuration.xsl”?>' >> /tmp/core-site.xml && \
     echo '<configuration><property><name>fs.defaultFS</name><value>HADOOP_DEFAULT_FS</value><final>true</final></property></configuration>' >>  /tmp/core-site.xml && \
     sed -i -e "s|HIVE_JDBC|${HIVE_JDBC}|g" /tmp/beeline-site.xml && \
     sed -i -e "s|HADOOP_DEFAULT_FS|${HADOOP_DEFAULT_FS}|g"  /tmp/core-site.xml && \
     mv /tmp/beeline-site.xml /usr/hdp/current/apache-hive-3.1.0-bin/conf/beeline-site.xml && \
     mv /tmp/core-site.xml /usr/hdp/current/hadoop-3.1.0/etc/hadoop/core-site.xml

# Configure the local user environment.
# This is needed only if you have installed hadoop and hive packages above.
# variables (PATH, etc) in the user's .bashrc
USER ${NB_USER}
ENV  JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
ENV  HADOOP_HOME=/usr/hdp/current/hadoop-3.1.0
ENV  HIVE_HOME=/usr/hdp/current/apache-hive-3.1.0-bin
RUN  echo 'export HADOOP_USER_NAME=${JUPYTERHUB_USER}' >> ~/.bashrc && \
     echo 'export PATH=${PATH}:${HIVE_HOME}/bin' >> ~/.bashrc && \
     echo 'export PATH=${PATH}:${HADOOP_HOME}/bin' >> ~/.bashrc

# Uncomment the RUN below to install the Hive kernel for Jupyter notebooks.
# Note: use of this kernel is a bit unstable, and not recommended.
#RUN /opt/conda/bin/pip install --upgrade hiveqlKernel && \
#    jupyter hiveql install --user && \
#    conda clean -y --all && \
#    conda env export -n "root"

# Install the python dependencies
COPY requirements.txt environment.yml /tmp/
RUN conda env update -q -f /tmp/environment.yml && \
    /opt/conda/bin/pip install -r /tmp/requirements.txt && \
    /opt/conda/bin/pip install bash_kernel && \
    python -m bash_kernel.install && \
    conda clean -y --all && \
    conda env export -n "root"