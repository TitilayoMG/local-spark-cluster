ARG SPARK_IMAGE=apache/spark:4.3.0-scala2.13-java21-python3-ubuntu

FROM ${SPARK_IMAGE} AS spark

USER root
RUN set -eu; \
    JARS_DIR=/opt/spark/jars; \
    HADOOP_VERSION=$(ls "${JARS_DIR}" | sed -n 's/^hadoop-client-api-\(.*\)\.jar$/\1/p'); \
    test -n "${HADOOP_VERSION}" || { echo "hadoop-client-api not found in ${JARS_DIR}" >&2; exit 1; }; \
    M2=https://repo1.maven.org/maven2; \
    POM=$(curl -fsSL "${M2}/org/apache/hadoop/hadoop-project/${HADOOP_VERSION}/hadoop-project-${HADOOP_VERSION}.pom"); \
    SDK_V2=$(printf '%s\n' "${POM}" | sed -n 's:.*<aws-java-sdk-v2.version>\(.*\)</aws-java-sdk-v2.version>.*:\1:p' | head -n1); \
    SDK_V1=$(printf '%s\n' "${POM}" | sed -n 's:.*<aws-java-sdk.version>\(.*\)</aws-java-sdk.version>.*:\1:p' | head -n1); \
    ACCEL=$(printf '%s\n' "${POM}" | sed -n 's:.*<amazon-s3-analyticsaccelerator-s3.version>\(.*\)</amazon-s3-analyticsaccelerator-s3.version>.*:\1:p' | head -n1); \
    curl -fsSL "${M2}/org/apache/hadoop/hadoop-aws/${HADOOP_VERSION}/hadoop-aws-${HADOOP_VERSION}.jar" \
      -o "${JARS_DIR}/hadoop-aws-${HADOOP_VERSION}.jar"; \
    if [ -n "${SDK_V2}" ]; then \
      curl -fsSL "${M2}/software/amazon/awssdk/bundle/${SDK_V2}/bundle-${SDK_V2}.jar" \
        -o "${JARS_DIR}/bundle-${SDK_V2}.jar"; \
      echo "Hadoop ${HADOOP_VERSION}: hadoop-aws + AWS SDK v2 ${SDK_V2}"; \
    else \
      curl -fsSL "${M2}/com/amazonaws/aws-java-sdk-bundle/${SDK_V1}/aws-java-sdk-bundle-${SDK_V1}.jar" \
        -o "${JARS_DIR}/aws-java-sdk-bundle-${SDK_V1}.jar"; \
      echo "Hadoop ${HADOOP_VERSION}: hadoop-aws + AWS SDK v1 ${SDK_V1}"; \
    fi; \
    if [ -n "${ACCEL}" ]; then \
      curl -fsSL "${M2}/software/amazon/s3/analyticsaccelerator/analyticsaccelerator-s3/${ACCEL}/analyticsaccelerator-s3-${ACCEL}.jar" \
        -o "${JARS_DIR}/analyticsaccelerator-s3-${ACCEL}.jar"; \
      echo "Hadoop ${HADOOP_VERSION}: analyticsaccelerator-s3 ${ACCEL}"; \
    fi; \
    chown spark:spark "${JARS_DIR}"/hadoop-aws-*.jar "${JARS_DIR}"/*bundle-*.jar "${JARS_DIR}"/analyticsaccelerator-s3-*.jar
USER spark


FROM spark AS jupyter

USER root
RUN set -eu; \
    pip3 install --no-cache-dir jupyterlab pandas pyarrow; \
    SITE_PACKAGES=$(python3 -c 'import site; print(site.getsitepackages()[0])'); \
    mkdir -p "${SITE_PACKAGES}"; \
    { echo /opt/spark/python; ls /opt/spark/python/lib/py4j-*-src.zip; } > "${SITE_PACKAGES}/spark-home.pth"; \
    python3 -c 'import pyspark; assert pyspark.__file__.startswith("/opt/spark/python"), pyspark.__file__; print("pyspark", pyspark.__version__)'; \
    mkdir -p /opt/spark/conf /home/CDE; \
    printf '%s\n' \
      'spark.master                spark://spark-master:7077' \
      'spark.driver.host           jupyter' \
      'spark.driver.bindAddress    0.0.0.0' \
      > /opt/spark/conf/spark-defaults.conf; \
    chown -R spark:spark /home/CDE; \
    usermod -d /home/CDE spark
USER spark

ENV HOME=/home/CDE
WORKDIR /home/CDE
EXPOSE 8888 4040

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["jupyter", "lab", "--ServerApp.ip=0.0.0.0", "--ServerApp.port=8888", "--no-browser", "--IdentityProvider.token=", "--ServerApp.root_dir=/home/CDE"]