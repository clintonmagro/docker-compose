FROM liquibase/liquibase:latest

USER root

# PostgreSQL JDBC driver is bundled; add MySQL connector
RUN apt-get update -qq && \
    apt-get install -y --no-install-recommends wget ca-certificates && \
    wget -q -O /liquibase/lib/mysql-connector-j.jar \
        "https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.3.0/mysql-connector-j-8.3.0.jar" && \
    apt-get purge -y wget && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

USER liquibase
