**Mô hình LAB**

|Server|IP|
|------|--|
|master1|10.1.38.128|
|master2|10.1.38.111|
|master3|10.1.38.149|
|worker1|10.1.38.144|
|worker2|10.1.38.146|
|mysql|10.1.38.147|
|NFS|10.1.38.129|

## 1. Thực hiện trên NFS
### 1.1 Tạo thư mục chia sẻ và phân quyền trên NFS
```sh
mkdir -p /data/wso2/worker
mkdir -p /data/wso2/dashboard
mkdir -p /data/wso2/apim
chown -R wso2carbon:wso2 /data/wso2
```
### 1.2 Chỉnh sửa Config
**Clone repository wso2:**
```sh
git clone https://github.com/wso2/docker-apim.git
```
**Copy các file cấu hình vào thư mục chia sẻ tương ứng:**
```sh
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim/config /data/wso2/apim
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim/artifact /data/wso2/apim 

co- r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-worker/config /data/wso2/worker

cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-dashboard/config /data/wso2/dashboard
cp -r /root/docker-apim/docker-compose/apim-with-analytics/apim-analytics-dashboard/artifact /data/wso2/dashboard
```
**Sửa file cấu hình của `apim` như sau: 
```sh
vim /data/wso2/apim/config/repository/conf/deployment.toml
[server]
hostname = "10.1.38.128"
node_ip = "10.1.38.128"
#offset=0
mode = "single" #single or ha
base_path = "${carbon.protocol}://${carbon.host}:${carbon.management.port}"
#discard_empty_caches = false
server_role = "default"

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[user_store]
type = "database"

[database.apim_db]
type = "mysql"
url = "jdbc:mysql://10.1.38.147:3306/WSO2AM_DB?autoReconnect=true&amp;useSSL=false"
username = "wso2carbon"
password = "wso2carbon"
driver = "com.mysql.cj.jdbc.Driver"

[database.shared_db]
type = "mysql"
url = "jdbc:mysql://10.1.38.147:3306/WSO2AM_SHARED_DB?autoReconnect=true&amp;useSSL=false"
username = "wso2carbon"
password = "wso2carbon"
driver = "com.mysql.cj.jdbc.Driver"

[keystore.tls]
file_name =  "wso2carbon.jks"
type =  "JKS"
password =  "wso2carbon"
alias =  "wso2carbon"
key_password =  "wso2carbon"

[[apim.gateway.environment]]
name = "Production and Sandbox"
type = "hybrid"
display_in_api_console = true
description = "This is a hybrid gateway that handles both production and sandbox token traffic."
show_as_token_endpoint_url = true
service_url = "https://10.1.38.128:${mgt.transport.https.port}/services/"
username= "${admin.username}"
password= "${admin.password}"
ws_endpoint = "ws://10.1.38.128:9099"
wss_endpoint = "wss://10.1.38.128:8099"
http_endpoint = "http://10.1.38.128:${http.nio.port}"
https_endpoint = "https://10.1.38.128:${https.nio.port}"

#[apim.cache.gateway_token]
#enable = true
#expiry_time = "900s"

#[apim.cache.resource]
#enable = true
#expiry_time = "900s"

#[apim.cache.km_token]
#enable = false
#expiry_time = "15m"

#[apim.cache.recent_apis]
#enable = false

#[apim.cache.scopes]
#enable = true

#[apim.cache.publisher_roles]
#enable = true

#[apim.cache.jwt_claim]
#enable = true
#expiry_time = "15m"

#[apim.cache.tags]
#expiry_time = "2m"

[apim.analytics]
enable = true
store_api_url = "https://am-analytics-worker:7444"
#username = "$ref{super_admin.username}"
#password = "$ref{super_admin.password}"
#event_publisher_type = "default"
#event_publisher_impl = "org.wso2.carbon.apimgt.usage.publisher.APIMgtUsageDataBridgeDataPublisher"
#publish_response_size = true

[[apim.analytics.url_group]]
analytics_url =["tcp://am-analytics-worker:7612"]
analytics_auth_url =["ssl://am-analytics-worker:7712"]
#type = "loadbalance"

#[[apim.analytics.url_group]]
#analytics_url =["tcp://analytics1:7612","tcp://analytics2:7612"]
#analytics_auth_url =["ssl://analytics1:7712","ssl://analytics2:7712"]
#type = "failover"

#[apim.key_manager]
#service_url = "https://localhost:${mgt.transport.https.port}/services/"
#username = "$ref{super_admin.username}"
#password = "$ref{super_admin.password}"
#pool.init_idle_capacity = 50
#pool.max_idle = 100
#key_validation_handler_type = "default"
#key_validation_handler_type = "custom"
#key_validation_handler_impl = "org.wso2.carbon.apimgt.keymgt.handlers.DefaultKeyValidationHandler"

[apim.jwt]
enable = true
encoding = "base64" # base64,base64url
generator_impl = "org.wso2.carbon.apimgt.keymgt.token.JWTGenerator"
claim_dialect = "http://wso2.org/claims"
header = "X-JWT-Assertion"
signing_algorithm = "SHA256withRSA"
enable_user_claims = true
claims_extractor_impl = "org.wso2.carbon.apimgt.impl.token.DefaultClaimsRetriever"

[apim.oauth_config]
enable_outbound_auth_header = true
auth_header = "Authorization"
revoke_endpoint = "https://10.1.38.128:${https.nio.port}/revoke"
enable_token_encryption = false
enable_token_hashing = false

#[apim.devportal]
#url = "https://localhost:${mgt.transport.https.port}/devportal"
#enable_application_sharing = false
#if application_sharing_type, application_sharing_impl both defined priority goes to application_sharing_impl
#application_sharing_type = "default" #changed type, saml, default #todo: check the new config for rest api
#application_sharing_impl = "org.wso2.carbon.apimgt.impl.SAMLGroupIDExtractorImpl"
#display_multiple_versions = false
#display_deprecated_apis = false
#enable_comments = true
#enable_ratings = true
#enable_forum = true

[apim.cors]
allow_origins = "*"
allow_methods = ["GET","PUT","POST","DELETE","PATCH","OPTIONS"]
allow_headers = ["authorization","Access-Control-Allow-Origin","Content-Type","SOAPAction"]
allow_credentials = false

#[apim.throttling]
#enable_data_publishing = true
#enable_policy_deploy = true
#enable_blacklist_condition = true
#enable_persistence = true
#throttle_decision_endpoints = ["tcp://localhost:5672","tcp://localhost:5672"]

#[apim.throttling.blacklist_condition]
#start_delay = "5m"
#period = "1h"

#[apim.throttling.jms]
#start_delay = "5m"

#[apim.throttling.event_sync]
#hostName = "0.0.0.0"
#port = 11224

#[apim.throttling.event_management]
#hostName = "0.0.0.0"
#port = 10005

#[[apim.throttling.url_group]]
#traffic_manager_urls = ["tcp://localhost:9611","tcp://localhost:9611"]
#traffic_manager_auth_urls = ["ssl://localhost:9711","ssl://localhost:9711"]
#type = "loadbalance"

#[[apim.throttling.url_group]]
#traffic_manager_urls = ["tcp://localhost:9611","tcp://localhost:9611"]
#traffic_manager_auth_urls = ["ssl://localhost:9711","ssl://localhost:9711"]
#type = "failover"

#[apim.workflow]
#enable = false
#service_url = "https://localhost:9445/bpmn"
#username = "$ref{super_admin.username}"
#password = "$ref{super_admin.password}"
#callback_endpoint = "https://localhost:${mgt.transport.https.port}/api/am/admin/v0.15/workflows/update-workflow-status"
#token_endpoint = "https://localhost:${https.nio.port}/token"
#client_registration_endpoint = "https://localhost:${mgt.transport.https.port}/client-registration/v0.15/register"
#client_registration_username = "$ref{super_admin.username}"
#client_registration_password = "$ref{super_admin.password}"

#data bridge config
#[transport.receiver]
#type = "binary"
#worker_threads = 10
#session_timeout = "30m"
#keystore.file_name = "$ref{keystore.tls.file_name}"
#keystore.password = "$ref{keystore.tls.password}"
#tcp_port = 9611
#ssl_port = 9711
#ssl_receiver_thread_pool_size = 100
#tcp_receiver_thread_pool_size = 100
#ssl_enabled_protocols = ["TLSv1","TLSv1.1","TLSv1.2"]
#ciphers = ["SSL_RSA_WITH_RC4_128_MD5","SSL_RSA_WITH_RC4_128_SHA"]

#[apim.notification]
#from_address = "APIM.com"
#username = "APIM"
#password = "APIM+123"
#hostname = "localhost"
#port = 3025
#enable_start_tls = false
#enable_authentication = true

#[apim.token.revocation]
#notifier_impl = "org.wso2.carbon.apimgt.keymgt.events.TokenRevocationNotifierImpl"
#enable_realtime_notifier = true
#realtime_notifier.ttl = 5000
#enable_persistent_notifier = true
#persistent_notifier.hostname = "https://localhost:2379/v2/keys/jti/"
#persistent_notifier.ttl = 5000
#persistent_notifier.username = "root"
#persistent_notifier.password = "root"

[[event_handler]]
name="userPostSelfRegistration"
subscriptions=["POST_ADD_USER"]

[service_provider]
sp_name_regex = "^[\\sa-zA-Z0-9._-]*$"
```
Trong đó cần lưu ý các trường sau:
- `10.1.38.128`: Thay đổi toàn bộ IP này bằng IP của máy mình
- `10.1.38.147`: Thay đổi toàn bộ IP này bằng IP của máy MYSQL

**Sửa file cấu hình của `apim` như sau: 
```sh
################################################################################
#   Copyright (c) 2017, WSO2 Inc. (http://www.wso2.org) All Rights Reserved
#
#   Licensed under the Apache License, Version 2.0 (the \"License\");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an \"AS IS\" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
################################################################################

# Carbon Configuration Parameters
wso2.carbon:
  type: wso2-apim-analytics
  # value to uniquely identify a server
  id: wso2-am-analytics
  # server name
  name: WSO2 API Manager Analytics Server
  # ports used by this server
  ports:
    # port offset
    offset: 1

wso2.transport.http:
  transportProperties:
    -
      name: "server.bootstrap.socket.timeout"
      value: 60
    -
      name: "client.bootstrap.socket.timeout"
      value: 60
    -
      name: "latency.metrics.enabled"
      value: true

  listenerConfigurations:
    -
      id: "default"
      host: "0.0.0.0"
      port: 9091
    -
      id: "msf4j-https"
      host: "0.0.0.0"
      port: 9444
      scheme: https
      keyStoreFile: "${carbon.home}/resources/security/wso2carbon.jks"
      keyStorePassword: wso2carbon
      certPass: wso2carbon

  senderConfigurations:
    -
      id: "http-sender"

siddhi.stores.query.api:
  transportProperties:
    -
      name: "server.bootstrap.socket.timeout"
      value: 60
    -
      name: "client.bootstrap.socket.timeout"
      value: 60
    -
      name: "latency.metrics.enabled"
      value: true

  listenerConfigurations:
    -
      id: "default"
      host: "0.0.0.0"
      port: 7071
    -
      id: "msf4j-https"
      host: "0.0.0.0"
      port: 7444
      scheme: https
      keyStoreFile: "${carbon.home}/resources/security/wso2carbon.jks"
      keyStorePassword: wso2carbon
      certPass: wso2carbon

  # Configuration used for the databridge communication
databridge.config:
  # No of worker threads to consume events
  # THIS IS A MANDATORY FIELD
  workerThreads: 10
    # Maximum amount of messages that can be queued internally in MB
  # THIS IS A MANDATORY FIELD
  maxEventBufferCapacity: 10000000
    # Queue size; the maximum number of events that can be stored in the queue
  # THIS IS A MANDATORY FIELD
  eventBufferSize: 2000
    # Keystore file path
  # THIS IS A MANDATORY FIELD
  keyStoreLocation : ${sys:carbon.home}/resources/security/wso2carbon.jks
    # Keystore password
  # THIS IS A MANDATORY FIELD
  keyStorePassword : wso2carbon
    # Session Timeout value in mins
  # THIS IS A MANDATORY FIELD
  clientTimeoutMin: 30
    # Data receiver configurations
  # THIS IS A MANDATORY FIELD
  dataReceivers:
    -
      # Data receiver configuration
      dataReceiver:
        # Data receiver type
        # THIS IS A MANDATORY FIELD
        type: Thrift
        # Data receiver properties
        properties:
          tcpPort: '7611'
          sslPort: '7711'

    -
      # Data receiver configuration
      dataReceiver:
        # Data receiver type
        # THIS IS A MANDATORY FIELD
        type: Binary
        # Data receiver properties
        properties:
          tcpPort: '9611'
          sslPort: '9711'
          tcpReceiverThreadPoolSize: '100'
          sslReceiverThreadPoolSize: '100'
          hostName: 0.0.0.0

  # Configuration of the Data Agents - to publish events through databridge
data.agent.config:
  # Data agent configurations
  # THIS IS A MANDATORY FIELD
  agents:
    -
      # Data agent configuration
      agentConfiguration:
        # Data agent name
        # THIS IS A MANDATORY FIELD
        name: Thrift
          # Data endpoint class
        # THIS IS A MANDATORY FIELD
        dataEndpointClass: org.wso2.carbon.databridge.agent.endpoint.thrift.ThriftDataEndpoint
        # Data publisher strategy
        publishingStrategy: async
        # Trust store path
        trustStorePath: '${sys:carbon.home}/resources/security/client-truststore.jks'
        # Trust store password
        trustStorePassword: 'wso2carbon'
        # Queue Size
        queueSize: 32768
        # Batch Size
        batchSize: 200
        # Core pool size
        corePoolSize: 1
        # Socket timeout in milliseconds
        socketTimeoutMS: 30000
        # Maximum pool size
        maxPoolSize: 1
        # Keep alive time in pool
        keepAliveTimeInPool: 20
        # Reconnection interval
        reconnectionInterval: 30
        # Max transport pool size
        maxTransportPoolSize: 250
        # Max idle connections
        maxIdleConnections: 250
        # Eviction time interval
        evictionTimePeriod: 5500
        # Min idle time in pool
        minIdleTimeInPool: 5000
        # Secure max transport pool size
        secureMaxTransportPoolSize: 250
        # Secure max idle connections
        secureMaxIdleConnections: 250
        # secure eviction time period
        secureEvictionTimePeriod: 5500
        # Secure min idle time in pool
        secureMinIdleTimeInPool: 5000
        # SSL enabled protocols
        sslEnabledProtocols: TLSv1.1,TLSv1.2
        # Ciphers
        ciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_DHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
    -
      # Data agent configuration
      agentConfiguration:
        # Data agent name
        # THIS IS A MANDATORY FIELD
        name: Binary
          # Data endpoint class
        # THIS IS A MANDATORY FIELD
        dataEndpointClass: org.wso2.carbon.databridge.agent.endpoint.binary.BinaryDataEndpoint
        # Data publisher strategy
        publishingStrategy: async
        # Trust store path
        trustStorePath: '${sys:carbon.home}/resources/security/client-truststore.jks'
        # Trust store password
        trustStorePassword: 'wso2carbon'
        # Queue Size
        queueSize: 32768
        # Batch Size
        batchSize: 200
        # Core pool size
        corePoolSize: 1
        # Socket timeout in milliseconds
        socketTimeoutMS: 30000
        # Maximum pool size
        maxPoolSize: 1
        # Keep alive time in pool
        keepAliveTimeInPool: 20
        # Reconnection interval
        reconnectionInterval: 30
        # Max transport pool size
        maxTransportPoolSize: 250
        # Max idle connections
        maxIdleConnections: 250
        # Eviction time interval
        evictionTimePeriod: 5500
        # Min idle time in pool
        minIdleTimeInPool: 5000
        # Secure max transport pool size
        secureMaxTransportPoolSize: 250
        # Secure max idle connections
        secureMaxIdleConnections: 250
        # secure eviction time period
        secureEvictionTimePeriod: 5500
        # Secure min idle time in pool
        secureMinIdleTimeInPool: 5000
        # SSL enabled protocols
        sslEnabledProtocols: TLSv1.1,TLSv1.2
        # Ciphers
        ciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_DHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256

# This is the main configuration for metrics
wso2.metrics:
  # Enable Metrics
  enabled: false
  reporting:
    console:
      - # The name for the Console Reporter
        name: Console

        # Enable Console Reporter
        enabled: false

        # Polling Period in seconds.
        # This is the period for polling metrics from the metric registry and printing in the console
        pollingPeriod: 5

wso2.metrics.jdbc:
  # Data Source Configurations for JDBC Reporters
  dataSource:
    # Default Data Source Configuration
    - &JDBC01
      # JNDI name of the data source to be used by the JDBC Reporter.
      # This data source should be defined in a *-datasources.xml file in conf/datasources directory.
      dataSourceName: java:comp/env/jdbc/WSO2MetricsDB
      # Schedule regular deletion of metrics data older than a set number of days.
      # It is recommended that you enable this job to ensure your metrics tables do not get extremely large.
      # Deleting data older than seven days should be sufficient.
      scheduledCleanup:
        # Enable scheduled cleanup to delete Metrics data in the database.
        enabled: true

        # The scheduled job will cleanup all data older than the specified days
        daysToKeep: 3

        # This is the period for each cleanup operation in seconds.
        scheduledCleanupPeriod: 86400

  # The JDBC Reporter is in the Metrics JDBC Core feature
  reporting:
    # The JDBC Reporter configurations will be ignored if the Metrics JDBC Core feature is not available in runtime
    jdbc:
      - # The name for the JDBC Reporter
        name: JDBC

        # Enable JDBC Reporter
        enabled: true

        # Source of Metrics, which will be used to identify each metric in database -->
        # Commented to use the hostname by default
        # source: Carbon

        # Alias referring to the Data Source configuration
        dataSource: *JDBC01

        # Polling Period in seconds.
        # This is the period for polling metrics from the metric registry and updating the database with the values
        pollingPeriod: 60

  # Deployment configuration parameters
wso2.artifact.deployment:
  # Scheduler update interval
  updateInterval: 5

  # Periodic Persistence Configuration
state.persistence:
  enabled: false
  intervalInMin: 1
  revisionsToKeep: 2
  persistenceStore: org.wso2.carbon.streaming.integrator.core.persistence.FileSystemPersistenceStore
  config:
    location: siddhi-app-persistence

  # Secure Vault Configuration
wso2.securevault:
  secretRepository:
    type: org.wso2.carbon.secvault.repository.DefaultSecretRepository
    parameters:
      privateKeyAlias: wso2carbon
      keystoreLocation: ${sys:carbon.home}/resources/security/securevault.jks
      secretPropertiesFile: ${sys:carbon.home}/conf/${sys:wso2.runtime}/secrets.properties
  masterKeyReader:
    type: org.wso2.carbon.secvault.reader.DefaultMasterKeyReader
    parameters:
      masterKeyReaderFile: ${sys:carbon.home}/conf/${sys:wso2.runtime}/master-keys.yaml

  # Datasource Configurations
wso2.datasources:
  dataSources:
    # carbon metrics data source
    - name: WSO2_METRICS_DB
      description: The datasource used for dashboard feature
      jndiConfig:
        name: jdbc/WSO2MetricsDB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:h2:${sys:carbon.home}/wso2/dashboard/database/metrics;AUTO_SERVER=TRUE'
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 30
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    - name: WSO2_PERMISSIONS_DB
      description: The datasource used for permission feature
      jndiConfig:
        name: jdbc/PERMISSION_DB
        useJndiReference: true
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_PERMISSIONS_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 10
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    - name: GEO_LOCATION_DATA
      description: "The data source used for geo location database"
      jndiConfig:
        name: jdbc/GEO_LOCATION_DATA
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:h2:${sys:carbon.home}/wso2/worker/database/GEO_LOCATION_DATA;AUTO_SERVER=TRUE'
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 50
          idleTimeout: 60000
          validationTimeout: 30000
          isAutoCommit: false

    - name: APIM_ANALYTICS_DB
      description: "The datasource used for APIM statistics aggregated data."
      jndiConfig:
        name: jdbc/APIM_ANALYTICS_DB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_STATS_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 50
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    #Main datasource used in API Manager
    - name: AM_DB
      description: Main datasource used by API Manager
      jndiConfig:
        name: jdbc/AM_DB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: "jdbc:h2:${sys:carbon.home}/../wso2am-3.0.0/repository/database/WSO2AM_DB;AUTO_SERVER=TRUE"
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 10
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    - name: WSO2AM_MGW_ANALYTICS_DB
      description: "The datasource used for APIM MGW analytics data."
      jndiConfig:
        name: jdbc/WSO2AM_MGW_ANALYTICS_DB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:h2:${sys:carbon.home}/wso2/worker/database/WSO2AM_MGW_ANALYTICS_DB;AUTO_SERVER=TRUE'
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 50
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false
    -
      name: WSO2_CLUSTER_DB
      description: "The datasource used by cluster coordinators in HA deployment"
      definition:
        type: RDBMS
        configuration:
          connectionTestQuery: "SELECT 1"
          driverClassName: org.h2.Driver
          idleTimeout: 60000
          isAutoCommit: false
          jdbcUrl: "jdbc:h2:${sys:carbon.home}/wso2/${sys:wso2.runtime}/database/WSO2_CLUSTER_DB;DB_CLOSE_ON_EXIT=FALSE;LOCK_TIMEOUT=60000;AUTO_SERVER=TRUE"
          maxPoolSize: 10
          password: wso2carbon
          username: wso2carbon
          validationTimeout: 30000

siddhi:
  refs:
    - ref:
        name: 'grpcSource'
        type: 'grpc'
        properties:
          receiver.url : grpc://localhost:9806/org.wso2.grpc.EventService/consume
  extensions:
    -
      extension:
        name: 'findCountryFromIP'
        namespace: 'geo'
        properties:
          geoLocationResolverClass: org.wso2.extension.siddhi.execution.geo.internal.impl.DefaultDBBasedGeoLocationResolver
          isCacheEnabled: true
          cacheSize: 10000
          isPersistInDatabase: true
          datasource: GEO_LOCATION_DATA
    -
      extension:
        name: 'findCityFromIP'
        namespace: 'geo'
        properties:
          geoLocationResolverClass: org.wso2.extension.siddhi.execution.geo.internal.impl.DefaultDBBasedGeoLocationResolver
          isCacheEnabled: true
          cacheSize: 10000
          isPersistInDatabase: true
          datasource: GEO_LOCATION_DATA
    #Enabling GRPC Service with an Extension
    -
      extension:
        name: 'grpc'
        namespace: 'source'
        properties:
          keyStoreFile : ${sys:carbon.home}/resources/security/wso2carbon.jks
          keyStorePassword : wso2carbon
          keyStoreAlgorithm : SunX509
          trustStoreFile : ${sys:carbon.home}/resources/security/client-truststore.jks
          trustStorePassword : wso2carbon
          trustStoreAlgorithm : SunX509

  # Cluster Configuration
cluster.config:
  enabled: false
  groupId:  sp
  coordinationStrategyClass: org.wso2.carbon.cluster.coordinator.rdbms.RDBMSCoordinationStrategy
  strategyConfig:
    datasource: WSO2_CLUSTER_DB
    heartbeatInterval: 1000
    heartbeatMaxRetry: 2
    eventPollingInterval: 1000

# Authentication configuration
auth.configs:
  type: 'local'        # Type of the IdP client used
  userManager:
    adminRole: admin   # Admin role which is granted all permissions
    userStore:         # User store
      users:
        -
          user:
            username: admin
            password: YWRtaW4=
            roles: 1
      roles:
        -
          role:
            id: 1
            displayName: admin

  # Configuration to enable apim alerts
  #analytics.solutions:
  #  APIM-alerts.enabled: true


  # Sample of deployment.config for Two node HA
  #deployment.config:
  #  type: ha
  #  eventSyncServer:
  #    host: localhost
  #    port: 9893
  #    advertisedHost: localhost
  #    advertisedPort: 9893
  #    bossThreads: 10
  #    workerThreads: 10
  #  eventSyncClientPool:
  #    maxActive: 10
  #    maxTotal: 10
  #    maxIdle: 10
  #    maxWait: 60000
  #    minEvictableIdleTimeMillis: 120000

  # Sample of deployment.config for Distributed deployment
#deployment.config:
#  type: distributed
#  httpsInterface:
#    host: 192.168.1.3
#    port: 9443
#    username: admin
#    password: admin
#  leaderRetryInterval: 10000
#  resourceManagers:
#    - host: 192.168.1.1
#      port: 9543
#      username: admin
#      password: admin
#    - host: 192.168.1.2
#      port: 9543
#      username: admin
#      password: admin
```
Trong đó lưu ý 1 số trường sau:
- `10.1.38.147`: Thay toàn bộ IP này bằng IP của máy MYSQL

**Sửa file cấu hình của `apim` như sau: 
```sh
################################################################################
#   Copyright (c) 2017, WSO2 Inc. (http://www.wso2.org) All Rights Reserved
#
#   Licensed under the Apache License, Version 2.0 (the \"License\");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#   http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an \"AS IS\" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.
################################################################################

# Carbon Configuration Parameters
wso2.carbon:
  type: wso2-apim-analytics
  # value to uniquely identify a server
  id: wso2-am-analytics
  # server name
  name: WSO2 API Manager Analytics Server
  # enable/disable hostname verifier
  hostnameVerificationEnabled: true
  # ports used by this server
  ports:
    # port offset
    offset: 3

  # Configuration used for the databridge communication
databridge.config:
  # No of worker threads to consume events
  # THIS IS A MANDATORY FIELD
  workerThreads: 10
    # Maximum amount of messages that can be queued internally in MB
  # THIS IS A MANDATORY FIELD
  maxEventBufferCapacity: 10000000
    # Queue size; the maximum number of events that can be stored in the queue
  # THIS IS A MANDATORY FIELD
  eventBufferSize: 2000
    # Keystore file path
  # THIS IS A MANDATORY FIELD
  keyStoreLocation : ${sys:carbon.home}/resources/security/wso2carbon.jks
    # Keystore password
  # THIS IS A MANDATORY FIELD
  keyStorePassword : wso2carbon
    # Session Timeout value in mins
  # THIS IS A MANDATORY FIELD
  clientTimeoutMin: 30
    # Data receiver configurations
  # THIS IS A MANDATORY FIELD
  dataReceivers:
    -
      # Data receiver configuration
      dataReceiver:
        # Data receiver type
        # THIS IS A MANDATORY FIELD
        type: Thrift
        # Data receiver properties
        properties:
          tcpPort: '7611'
          sslPort: '7711'

    -
      # Data receiver configuration
      dataReceiver:
        # Data receiver type
        # THIS IS A MANDATORY FIELD
        type: Binary
        # Data receiver properties
        properties:
          tcpPort: '9611'
          sslPort: '9711'
          tcpReceiverThreadPoolSize: '100'
          sslReceiverThreadPoolSize: '100'
          hostName: 0.0.0.0

  # Configuration of the Data Agents - to publish events through databridge
data.agent.config:
  # Data agent configurations
  # THIS IS A MANDATORY FIELD
  agents:
    -
      # Data agent configuration
      agentConfiguration:
        # Data agent name
        # THIS IS A MANDATORY FIELD
        name: Thrift
          # Data endpoint class
        # THIS IS A MANDATORY FIELD
        dataEndpointClass: org.wso2.carbon.databridge.agent.endpoint.thrift.ThriftDataEndpoint
        # Data publisher strategy
        publishingStrategy: async
        # Trust store path
        trustStorePath: '${sys:carbon.home}/resources/security/client-truststore.jks'
        # Trust store password
        trustStorePassword: 'wso2carbon'
        # Queue Size
        queueSize: 32768
        # Batch Size
        batchSize: 200
        # Core pool size
        corePoolSize: 1
        # Socket timeout in milliseconds
        socketTimeoutMS: 30000
        # Maximum pool size
        maxPoolSize: 1
        # Keep alive time in pool
        keepAliveTimeInPool: 20
        # Reconnection interval
        reconnectionInterval: 30
        # Max transport pool size
        maxTransportPoolSize: 250
        # Max idle connections
        maxIdleConnections: 250
        # Eviction time interval
        evictionTimePeriod: 5500
        # Min idle time in pool
        minIdleTimeInPool: 5000
        # Secure max transport pool size
        secureMaxTransportPoolSize: 250
        # Secure max idle connections
        secureMaxIdleConnections: 250
        # secure eviction time period
        secureEvictionTimePeriod: 5500
        # Secure min idle time in pool
        secureMinIdleTimeInPool: 5000
        # SSL enabled protocols
        sslEnabledProtocols: TLSv1.1,TLSv1.2
        # Ciphers
        ciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_DHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
    -
      # Data agent configuration
      agentConfiguration:
        # Data agent name
        # THIS IS A MANDATORY FIELD
        name: Binary
          # Data endpoint class
        # THIS IS A MANDATORY FIELD
        dataEndpointClass: org.wso2.carbon.databridge.agent.endpoint.binary.BinaryDataEndpoint
        # Data publisher strategy
        publishingStrategy: async
        # Trust store path
        trustStorePath: '${sys:carbon.home}/resources/security/client-truststore.jks'
        # Trust store password
        trustStorePassword: 'wso2carbon'
        # Queue Size
        queueSize: 32768
        # Batch Size
        batchSize: 200
        # Core pool size
        corePoolSize: 1
        # Socket timeout in milliseconds
        socketTimeoutMS: 30000
        # Maximum pool size
        maxPoolSize: 1
        # Keep alive time in pool
        keepAliveTimeInPool: 20
        # Reconnection interval
        reconnectionInterval: 30
        # Max transport pool size
        maxTransportPoolSize: 250
        # Max idle connections
        maxIdleConnections: 250
        # Eviction time interval
        evictionTimePeriod: 5500
        # Min idle time in pool
        minIdleTimeInPool: 5000
        # Secure max transport pool size
        secureMaxTransportPoolSize: 250
        # Secure max idle connections
        secureMaxIdleConnections: 250
        # secure eviction time period
        secureEvictionTimePeriod: 5500
        # Secure min idle time in pool
        secureMinIdleTimeInPool: 5000
        # SSL enabled protocols
        sslEnabledProtocols: TLSv1.1,TLSv1.2
        # Ciphers
        ciphers: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256,TLS_DHE_RSA_WITH_AES_128_CBC_SHA256,TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,TLS_DHE_RSA_WITH_AES_128_CBC_SHA,TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_DHE_RSA_WITH_AES_128_GCM_SHA256

  # Deployment configuration parameters
wso2.artifact.deployment:
  # Scheduler update interval
  updateInterval: 5

  # HA Configuration
state.persistence:
  enabled: false
  intervalInMin: 1
  revisionsToKeep: 2
  persistenceStore: org.wso2.carbon.streaming.integrator.core.persistence.FileSystemPersistenceStore
  config:
    location: siddhi-app-persistence

  # Secure Vault Configuration
wso2.securevault:
  secretRepository:
    type: org.wso2.carbon.secvault.repository.DefaultSecretRepository
    parameters:
      privateKeyAlias: wso2carbon
      keystoreLocation: ${sys:carbon.home}/resources/security/securevault.jks
      secretPropertiesFile: ${sys:carbon.home}/conf/${sys:wso2.runtime}/secrets.properties
  masterKeyReader:
    type: org.wso2.carbon.secvault.reader.DefaultMasterKeyReader
    parameters:
      masterKeyReaderFile: ${sys:carbon.home}/conf/${sys:wso2.runtime}/master-keys.yaml


# Data Sources Configuration
wso2.datasources:
  dataSources:
    # Dashboard data source
    - name: WSO2_DASHBOARD_DB
      description: The datasource used for dashboard feature
      jndiConfig:
        name: jdbc/DASHBOARD_DB
        useJndiReference: true
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:h2:${sys:carbon.home}/wso2/${sys:wso2.runtime}/database/DASHBOARD_DB;IFEXISTS=TRUE;DB_CLOSE_ON_EXIT=FALSE;LOCK_TIMEOUT=60000;MVCC=TRUE'
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 20
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false
    - name: BUSINESS_RULES_DB
      description: The datasource used for dashboard feature
      jndiConfig:
        name: jdbc/BUSINESS_RULES_DB
        useJndiReference: true
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_BUSINESS_RULES_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 20
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    # carbon metrics data source
    - name: WSO2_METRICS_DB
      description: The datasource used for dashboard feature
      jndiConfig:
        name: jdbc/WSO2MetricsDB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:h2:${sys:carbon.home}/wso2/dashboard/database/metrics;AUTO_SERVER=TRUE'
          username: wso2carbon
          password: wso2carbon
          driverClassName: org.h2.Driver
          maxPoolSize: 20
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    - name: WSO2_PERMISSIONS_DB
      description: The datasource used for dashboard feature
      jndiConfig:
        name: jdbc/PERMISSION_DB
        useJndiReference: true
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_PERMISSIONS_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 10
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    #Data source for APIM Analytics
    - name: APIM_ANALYTICS_DB
      description: Datasource used for APIM Analytics
      jndiConfig:
        name: jdbc/APIM_ANALYTICS_DB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_STATS_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 50
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

    #Main datasource used in API Manager
    - name: AM_DB
      description: Main datasource used by API Manager
      jndiConfig:
        name: jdbc/AM_DB
      definition:
        type: RDBMS
        configuration:
          jdbcUrl: 'jdbc:mysql://10.1.38.147:3306/WSO2AM_DB?useSSL=false'
          username: wso2carbon
          password: wso2carbon
          driverClassName: com.mysql.cj.jdbc.Driver
          maxPoolSize: 10
          idleTimeout: 60000
          connectionTestQuery: SELECT 1
          validationTimeout: 30000
          isAutoCommit: false

wso2.business.rules.manager:
  datasource: BUSINESS_RULES_DB
  # rule template wise configuration for deploying business rules
  deployment_configs:
    -
      # <IP>:<HTTPS Port> of the Worker node
      localhost:9444:
        # UUIDs of rule templates that are needed to be deployed on the node
        - stock-data-analysis
        - stock-exchange-input
        - stock-exchange-output
        - identifying-continuous-production-decrease
        - popular-tweets-analysis
        - http-analytics-processing
        - message-tracing-source-template
        - message-tracing-app-template
  # credentials for worker nodes
  username: admin
  password: admin

wso2.transport.http:
  transportProperties:
    - name: "server.bootstrap.socket.timeout"
      value: 60
    - name: "client.bootstrap.socket.timeout"
      value: 60
    - name: "latency.metrics.enabled"
      value: true

  listenerConfigurations:
    - id: "default-https"
      host: "0.0.0.0"
      port: 9643
      scheme: https
      keyStoreFile: "${carbon.home}/resources/security/wso2carbon.jks"
      keyStorePassword: wso2carbon
      certPass: wso2carbon

## Dashboard data provider authorization
data.provider.configs:
  authorizingClass: org.wso2.carbon.dashboards.core.DashboardDataProviderAuthorizer

## Additional APIs that needs to be added to the server.
## Should be provided as a key value pairs { API context path: Microservice implementation class }
## The configured APIs will be available as https://{host}:{port}/analytics-dashboard/{API_context_path}
additional.apis:
  /apis/analytics/v1.0/apim: org.wso2.analytics.apim.rest.api.proxy.ApimApi

## Authentication configuration
auth.configs:
  type: apim
  ssoEnabled: true
  properties:
    adminScope: apim_analytics:admin_carbon.super
    allScopes: apim_analytics:admin apim_analytics:product_manager apim_analytics:api_developer apim_analytics:app_developer apim_analytics:devops_engineer apim_analytics:analytics_viewer apim_analytics:everyone openid apim:api_view apim:subscribe
    adminServiceBaseUrl: https://api-manager:9443
    adminUsername: admin
    adminPassword: admin
    kmDcrUrl: https://api-manager:9443/client-registration/v0.15/register
    kmTokenUrlForRedirection: https://localhost:9443/oauth2
    kmTokenUrl: https://api-manager:9443/oauth2
    kmUsername: admin
    kmPassword: admin
    portalAppContext: analytics-dashboard
    businessRulesAppContext : business-rules
    cacheTimeout: 900
    baseUrl: https://localhost:9643
    grantType: authorization_code
    publisherUrl: https://api-manager:9443
    #storeUrl: https://localhost:9443

wso2.dashboard:
  roles:
    creators:
      - apim_analytics:admin_carbon.super
```
Trong đó lưu ý 1 số trường sau:
- `10.1.38.147`: Thay toàn bộ IP này bằng IP của máy MYSQL của mình.

## 2. Thực hiện trên Node Master
Tạo 3 file cho 3 serice: `am-analytics-worker.yaml`, `api-manager`, `am-analytics-dashboard`.**
### 2.1 Tạo file `api-manager.yaml` với nội dung gồm các phần như sau:
**Tạo 2 `PersistentVolume` và 2 `PersistentVolumeClaim` tương ứng như sau:**
```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-config
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/config"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-artifact
```
Cần lưu ý 1 số trường như sau: 
- `spec.nfs`: IP của NFS server và đường dẫn. Thay đổi với Topologi của mình cho phù hợp, ở dây là `10.1.38.129` và path: `/data/wso2/apim/artifact`
- Các trường khác có thể giữ nguyên hoặc thay đổi nhưng cần chú ý label cho khớp.

**Tạo Deployment với nội dung như sau:**
```sh
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-manager-deployment
  namespace: wso2
  labels:
    app: api-manager
    role: api-manager
spec:
  selector:
    matchLabels:
      app: api-manager
      role: api-manager
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: api-manager
        role: api-manager
    spec:
      containers:
      - name: api-manager
        image: wso2/wso2am:3.0.0
        ports:
        - containerPort: 9763
          name: port1
        - containerPort: 9443
          name: port2
        - containerPort: 8280
          name: port3
        - containerPort: 8243
          name: port4
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: api-manager-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: api-manager-artifact
      volumes:
      - name: api-manager-config
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-config
      - name: api-manager-artifact
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-artifact
```
**Tạo file Service với nội dung sau:**
```sh
apiVersion: v1
kind: Service
metadata:
  name: api-manager
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9763
    name: port1
    nodePort: 31124
  - port: 9443
    name: port2
    nodePort: 31125
  - port: 8280
    name: port3
    nodePort: 31126
  - port: 8243
    name: port4
    nodePort: 31127
  selector:
    app: api-manager
    role: api-manager
```
Tất cả các File được gom lại thành 1 file `api-manager.yaml` như sau:
```sh
cat << EOF > api-manager.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-config
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/config"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: api-manager-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: api-manager-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/apim/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: api-manager-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: api-manager-nfs-pv-artifact
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-manager-deployment
  namespace: wso2
  labels:
    app: api-manager
    role: api-manager
spec:
  selector:
    matchLabels:
      app: api-manager
      role: api-manager
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: api-manager
        role: api-manager
    spec:
      containers:
      - name: api-manager
        image: wso2/wso2am:3.0.0
        ports:
        - containerPort: 9763
          name: port1
        - containerPort: 9443
          name: port2
        - containerPort: 8280
          name: port3
        - containerPort: 8243
          name: port4
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: api-manager-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: api-manager-artifact
      volumes:
      - name: api-manager-config
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-config
      - name: api-manager-artifact
        persistentVolumeClaim:
          claimName: api-manager-nfs-pvc-artifact
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-api-manager
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-manager-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-manager
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9763
    name: port1
    nodePort: 31124
  - port: 9443
    name: port2
    nodePort: 31125
  - port: 8280
    name: port3
    nodePort: 31126
  - port: 8243
    name: port4
    nodePort: 31127
  selector:
    app: api-manager
    role: api-manager
EOF
```
### 2.2 Tạo file `am-analytics-worker.yaml` với nội dung sau:**
```sh
cat << EOF > am-analytics-worker.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-worker-nfs-pv-config
  namespace: wso2
  labels:
    storage: am-analytics-worker-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/worker"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-worker-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-worker-nfs-pv-config
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: am-analytics-worker-deployment
  namespace: wso2
  labels:
    app: am-analytics-worker
    role: am-analytics-worker
spec:
  selector:
    matchLabels:
      app: am-analytics-worker
      role: am-analytics-worker
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: am-analytics-worker
        role: am-analytics-worker
    spec:
      containers:
      - name: am-analytics-worker
        image: wso2/wso2am-analytics-worker:3.0.0
        ports:
        - containerPort: 9091
          name: port1
        - containerPort: 9444
          name: port2
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: am-analytics-worker-config
      volumes:
      - name: am-analytics-worker-config
        persistentVolumeClaim:
          claimName: am-analytics-worker-nfs-pvc-config
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-am-analytics-worker
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: am-analytics-worker-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: am-analytics-worker
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9091
    name: port1
    nodePort: 31122
  - port: 9444
    name: port2
    nodePort: 31123
  selector:
    app: am-analytics-worker
    role: am-analytics-worker
EOF
```
### 2.3 Tạo file `am-analytics-dashboard.yaml` với nội dung sau:
```sh
cat << EOF > am-analytics-dashboard.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-dashboard-nfs-pv-config
  namespace: wso2
  labels:
    storage: am-analytics-dashboard-nfs-pv-config
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/dashboard/config"
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: am-analytics-dashboard-nfs-pv-artifact
  namespace: wso2
  labels:
    storage: am-analytics-dashboard-nfs-pv-artifact
spec:
  storageClassName: nfs-volume
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 10.1.38.129
    path: "/data/wso2/dashboard/artifact"
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-dashboard-nfs-pvc-config
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-dashboard-nfs-pv-config
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: am-analytics-dashboard-nfs-pvc-artifact
  namespace: wso2
spec:
  storageClassName: nfs-volume
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  selector:
    matchLabels:
      storage: am-analytics-dashboard-nfs-pv-artifact
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: am-analytics-dashboard-deployment
  namespace: wso2
  labels:
    app: am-analytics-dashboard
    role: am-analytics-dashboard
spec:
  selector:
    matchLabels:
      app: am-analytics-dashboard
      role: am-analytics-dashboard
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0
  minReadySeconds: 120
  template:
    metadata:
      labels:
        app: am-analytics-dashboard
        role: am-analytics-dashboard
    spec:
      containers:
      - name: am-analytics-dashboard
        image: wso2/wso2am-analytics-dashboard:3.0.0
        ports:
        - containerPort: 9643
          name: port1
        resources:
          requests:
            cpu: "1000m"
            memory: "4096Mi"
          limits: 
            cpu: "1000m"
            memory: "4096Mi"
        volumeMounts:
        - mountPath: /home/wso2carbon/wso2-config-volume
          name: am-analytics-dashboard-config
        - mountPath: /home/wso2carbon/wso2-artifact-volume
          name: am-analytics-dashboard-artifact
      volumes:
      - name: am-analytics-dashboard-config
        persistentVolumeClaim:
          claimName: am-analytics-dashboard-nfs-pvc-config
      - name: am-analytics-dashboard-artifact
        persistentVolumeClaim:
          claimName: am-analytics-dashboard-nfs-pvc-artifact
---
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-am-analytics-dashboard
  namespace: wso2
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: am-analytics-dashboard-deployment
  minReplicas: 1
  maxReplicas: 1
  metrics:
  - type: Resource
    resource:
      name: memory
      targetAverageUtilization: 80
  - type: Resource
    resource:
      name: cpu
      targetAverageUtilization: 80
---
apiVersion: v1
kind: Service
metadata:
  name: am-analytics-dashboard
  namespace: wso2
spec:
  type: NodePort
  ports:
  - port: 9643
    name: port1
    nodePort: 31128
  selector:
    app: am-analytics-dashboard
    role: am-analytics-dashboard
EOF
```
## 3. Thực hiện trên MYSQL 

## 4. Thực hiện deploy các `service`, `deployment`, `hpa`, `pv`, `pvc` vừa tạo như sau:
Tạo Namespace `wso2`
```sh
kubectl create namespace wso2
```
Thực hiện `deploy`, `service`, `deployment`, `hpa`, `pv`, `pvc`
```sh
kubectl apply -f am-analytics-dashboard.yaml
kubectl apply -f api-manager.yaml
kubectl apply -f am-analytics-worker.yaml
```
