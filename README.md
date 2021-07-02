# quay


 
     
    make sure all these ports are opened:
     
    8443/tcp 80/tcp 443/tcp 3306/tcp 6379/tcp 5432/tcp 6060/tcp 6161/tcp 6061/tcp 6062/tcp 6063/tcp 8080/tcp 8089/tcp 5433/tcp 8081/tcp
     
    Setup PostGRES for Clairv4
     
    mkdir /mnt/postgres-clair4
    setfacl -m u:26:-wx /mnt/postgres-clair4
     
    docker run -d --rm --name postgresql-clair4 -e POSTGRESQL_USER=clairuser -e POSTGRESQL_PASSWORD=clairpass -e POSTGRESQL_DATABASE=clair -e POSTGRESQL_ADMIN_PASSWORD=password -p 5433:5432 -v /mnt/postgres-clair4:/var/lib/pgsql/data:Z -d registry.redhat.io/rhel8/postgresql-10:1
     
    docker exec -it postgresql-clair4 /bin/bash -c 'echo "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\"" | psql -d clair -U postgres'
     
    docker run -d --rm --name redis -p 6379:6379 -e REDIS_PASSWORD=strongpassword registry.redhat.io/rhel8/redis-5:1
     
     
    clair4-config.yaml
     
    http_listen_addr: :8081
    introspection_addr: :8089
    log_level: debug
    indexer:
      connstring: host=quay.schmaustech.com port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable
      scanlock_retry: 10
      layer_scan_concurrency: 5
      migrations: true
    matcher:
      connstring: host=quay.schmaustech.com port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable
      max_conn_pool: 100
      run: ""
      migrations: true
      indexer_addr: clair-indexer
    notifier:
      connstring: host=quay.schmaustech.com port=5433 dbname=clair user=clairuser password=clairpass sslmode=disable
      delivery_interval: 1m
      poll_interval: 5m
      migrations: true
    auth:
      psk:
        key: "MTU5YzA4Y2ZkNzJoMQ=="
        iss: ["quay"]
    # tracing and metrics
    trace:
      name: "jaeger"
      probability: 1
      jaeger:
        agent_endpoint: "localhost:6831"
        service_name: "clair"
    metrics:
      name: "prometheus"
     
    Start Clairv4
     
    docker run -d --rm --name clairv4 -p 8081:8081 -p 8089:8089 -e CLAIR_CONF=/clair/config.yaml -e CLAIR_MODE=combo -v /mnt/clair4/config:/clair:Z registry.redhat.io/quay/clair-rhel8:v3.4.5
     
    Edit Config.yaml of quay to update the security section for scanner:
     
    FEATURE_SECURITY_NOTIFICATIONS: false
    FEATURE_SECURITY_SCANNER: true
     
     
    SECURITY_SCANNER_INDEXING_INTERVAL: 30
    SECURITY_SCANNER_V4_ENDPOINT: http://quay.schmaustech.com:8081
    SECURITY_SCANNER_V4_PSK: MTU5YzA4Y2ZkNzJoMQ==
    SERVER_HOSTNAME: quay.schmaustech.com
     
     
    Update Quay Configuration
     
    docker run --rm -it --name quay_config -p 8080:8080 -v /mnt/quay/config:/conf/stack:Z registry.redhat.io/quay/quay-rhel8:v3.4.5 config password
     
    replace confiuration with one saved from web ui
     
    (This did not work) docker run --restart=always -p 443:8443 -p 80:8080 --name quay --sysctl net.core.somaxconn=4096 --privileged=true -v /mnt/quay/config:/conf/stack:Z -v /mnt/quay/storage:/datastorage:Z -d registry.redhat.io/quay/quay-rhel8:v3.4.5
     
    docker run -d --rm -p 443:8443 -p 8080:8080 --name quay -v /mnt/quay/config:/conf/stack:Z -v /mnt/quay/storage:/datastorage:Z registry.redhat.io/quay/quay-rhel8:v3.4.5
    
    Upgrade to 3.5.2 
    
    docker run -d --rm --name clairv4 -p 8081:8081 -p 8089:8089 -e CLAIR_CONF=/clair/config.yaml -e CLAIR_MODE=combo -v /mnt/clair4/config:/clair:Z registry.redhat.io/quay/clair-rhel8:v3.5.2
    
    docker run -d --rm -p 443:8443 -p 8080:8080 --name quay -v /mnt/quay/config:/conf/stack:Z -v /mnt/quay/storage:/datastorage:Z registry.redhat.io/quay/quay-rhel8:v3.5.2
