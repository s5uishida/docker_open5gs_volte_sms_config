FROM ubuntu:focal

ENV DEBIAN_FRONTEND=noninteractive

RUN apt update && \
    apt install -y wget gnupg software-properties-common ca-certificates lsb-release && \
    wget -qO - https://www.mongodb.org/static/pgp/server-4.4.asc | gpg --dearmor -o /etc/apt/trusted.gpg.d/mongodb-4.4.gpg && \
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu $(lsb_release -cs)/mongodb-org/4.4 multiverse" | tee /etc/apt/sources.list.d/mongodb-org-4.4.list && \
    apt update && \
    apt install -y --no-install-recommends mongodb-org-mongos=4.4.18 mongodb-org-tools=4.4.18 mongodb-org-shell=4.4.18 mongodb-org-database-tools-extra=4.4.18 mongodb-org=4.4.18 mongodb-org-server=4.4.18 && \
    apt autoremove -y && apt autoclean

CMD ["mongod"]
