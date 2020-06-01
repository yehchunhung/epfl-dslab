FROM renku/renkulab:renku0.9.1-py3.7-0.5.2

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

USER root

# Install dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends openjdk-8-jre-headless && \
    apt-get clean

# Prepare configuration files
ARG HADOOP_DEFAULT_FS_ARG="hdfs://iccluster044.iccluster.epfl.ch:8020"
ARG YARN_RM_HOSTNAME_ARG="iccluster044.iccluster.epfl.ch"
ARG LIVY_SERVER_ARG="http://iccluster044.iccluster.epfl.ch:8998/"

ENV HADOOP_DEFAULT_FS=${HADOOP_DEFAULT_FS_ARG}
ENV HADOOP_HOME=/usr/hdp/current/hadoop-3.1.0/
ENV HADOOP_CONF_DIR=${HADOOP_HOME}/etc/hadoop/
ENV YARN_RM_HOSTNAME=${YARN_RM_HOSTNAME_ARG}
ENV YARN_RM_ADDRESS=${YARN_RM_HOSTNAME_ARG}:8050
ENV YARN_RM_SCHEDULER=${YARN_RM_HOSTNAME_ARG}:8030
ENV YARN_RM_TRACKER=${YARN_RM_HOSTNAME_ARG}:8025
ENV LIVY_SERVER_URL=${LIVY_SERVER_ARG}
ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/

# Install hadoop  3.1.0
RUN mkdir -p /usr/hdp/current && \
    cd /usr/hdp/current && \
    # Hadoop MapReduce
    wget -q https://archive.apache.org/dist/hadoop/core/hadoop-3.1.0/hadoop-3.1.0.tar.gz && \
    tar --no-same-owner -xf hadoop-3.1.0.tar.gz && \
    rm hadoop-3.1.0.tar.gz

# Configure Hadoop core-site.xml
RUN echo '<?xml version="1.0" encoding="UTF-8"?>\n\
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>\n\
<configuration>\n\
    <property>\n\
        <name>fs.defaultFS</name>\n\
        <value>'${HADOOP_DEFAULT_FS}'</value>\n\
        <final>true</final>\n\
    </property>\n\
</configuration>' > /usr/hdp/current/hadoop-3.1.0/etc/hadoop/core-site.xml

# Configure Yarn yarn-site.xml
RUN echo '<?xml version="1.0"?>\n\
<configuration>\n\
    <property>\n\
      <name>yarn.nodemanager.address</name>\n\
      <value>0.0.0.0:45454</value>\n\
    </property>\n\
    <property>\n\
      <name>yarn.nodemanager.bind-host</name>\n\
      <value>0.0.0.0</value>\n\
    </property>\n\
    <property>\n\
        <name>yarn.resourcemanager.hostname</name>\n\
        <value>'${YARN_RM_HOSTNAME}'</value>\n\
    </property>\n\
    <property>\n\
        <name>yarn.resourcemanager.address</name>\n\
        <value>'${YARN_RM_ADDRESS}'</value>\n\
    </property>\n\
    <property>\n\
      <name>yarn.resourcemanager.resource-tracker.address</name>\n\
      <value>'${YARN_RM_TRACKER}'</value>\n\
    </property>\n\
    <property>\n\
      <name>yarn.resourcemanager.scheduler.address</name>\n\
      <value>'${YARN_RM_SCHEDULER}'</value>\n\
    </property>\n\
</configuration>' > /usr/hdp/current/hadoop-3.1.0/etc/hadoop/yarn-site.xml

# Renku-hack modify entrypoint.sh
RUN if [ -e /entrypoint.sh ]; then \
    sed -i -Ee 's,^\$\@$,if \[\[ -x ~/.renkurc \]\];then . ~/.renkurc;fi\n\$\@,' /entrypoint.sh; \
    fi

USER ${NB_USER}

# Install sparkmagic
RUN /opt/conda/bin/pip install sparkmagic && \
    export JUPYTERLAB_DIR=/opt/conda/share/jupyter/lab && \
    export JUPYTERLAB_SETTINGS_DIR=/home/jovyan/.jupyter/lab/user-settings && \
    export JUPYTERLAB_WORKSPACES_DIR=/home/jovyan/.jupyter/lab/workspaces && \
    # /opt/conda/bin/jupyter nbextension enable --py --sys-prefix widgetsnbextension && \
    /opt/conda/bin/jupyter labextension install -y --log-level=INFO @jupyter-widgets/jupyterlab-manager && \
    cd "$(pip show sparkmagic|sed -En 's/Location: (.*)$/\1/p')" && \
    jupyter-kernelspec install sparkmagic/kernels/sparkkernel --user && \
    jupyter-kernelspec install sparkmagic/kernels/sparkrkernel --user && \
    jupyter-kernelspec install sparkmagic/kernels/pysparkkernel --user && \
    jupyter serverextension enable --py sparkmagic

# Set user environment
# + https://github.com/jupyter-incubator/sparkmagic/blob/master/sparkmagic/example_config.json
RUN echo 'export HADOOP_USER_NAME=${JUPYTERHUB_USER}' >> ~/.bashrc && \
    echo 'export PATH=${PATH}:${HADOOP_HOME}/bin' >> ~/.bashrc && \
    mkdir -p ~/.sparkmagic/ && \
    echo '{\n\
  "kernel_python_credentials" : {\n\
    "url": "'${LIVY_SERVER_URL}'"\n\
  },\n\n\
  "kernel_scala_credentials" : {\n\
    "url": "'${LIVY_SERVER_URL}'"\n\
  },\n\n\
  "custom_headers" : {\n\
    "X-Requested-By": "livy"\n\
  },\n\n\
  "session_configs" : {\n\
    "driverMemory": "1000M",\n\
    "executorMemory": "4G",\n\
    "executorCores": 4,\n\
    "numExecutors": 10\n\
  },\n\
  "server_extension_default_kernel_name": "pysparkkernel",\n\
  "use_auto_viz": true,\n\
  "coerce_dataframe": true,\n\
  "max_results_sql": 1000,\n\
  "pyspark_dataframe_encoding": "utf-8",\n\
  "heartbeat_refresh_seconds": 5,\n\
  "livy_server_heartbeat_timeout_seconds": 60,\n\
  "heartbeat_retry_seconds": 1\n\
}\n' > ~/.sparkmagic/config.json

# install the python dependencies
COPY requirements.txt environment.yml /tmp/
RUN conda env update -q -f /tmp/environment.yml && \
    /opt/conda/bin/pip install -r /tmp/requirements.txt && \
    conda clean -y --all && \
    conda env export -n "root"