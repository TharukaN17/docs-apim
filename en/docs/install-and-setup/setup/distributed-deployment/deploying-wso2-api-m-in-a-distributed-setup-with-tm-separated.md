# Configuring a Distributed API-M Deployment with Traffic Manager Separated from the Control Plane

The WSO2 API-M server can be deployed as an [all-in-one deployment]({{base_path}}/install-and-setup/setup/single-node/all-in-one-deployment-overview) or as a distributed deployment. In the distributed setup, the [API-M server profiles]({{base_path}}/install-and-setup/setup/distributed-deployment/product-profiles) are deployed as separate API-M nodes. 

Given below are the API-M nodes you can have in a distributed deployment.

!!! Tip
    To enable high availability, you need a minimum of two nodes running each profile.

<table>
    <tr>
        <th>
            API-M Node (Profile)
        </th>
        <th>
            Description
        </th>
    </tr>
    <tr>
        <td>
            Gateway Worker Node
        </td>
        <td>
            API-M nodes running the Gateway profile.
        </td>
    </tr>
    <tr>
        <td>
            Control Plane Node
        </td>
        <td>
            API-M nodes running the Control Plane profile. The Control Plane includes the Traffic Manager, Key Manager, Publisher, and Developer Portal components.
        </td>
    </tr>
    <tr>
        <td>
            Traffic Manager Node (Optional)
        </td>
        <td>
            If required you can configure a separate API-M node to run the Traffic Manager component. That is, the Control Plane nodes run the Key Manager, Publisher, and Developer Portal, while the Traffic Manager runs on a separate node.
        </td>
    </tr>
</table>

<a href="{{base_path}}/assets/img/setup-and-install/deployment-tm.png"><img src="{{base_path}}/assets/img/setup-and-install/deployment-tm.png" width="100%"></a>

## Step 1 - Install WSO2 API-M

To install and set up the API-M servers:

1.  Download the [WSO2 API Manager](https://wso2.com/api-manager/).
2.  Create copies of the API-M distribution for the individual profiles.

## Step 2 - Install and configure the databases

You can create the required databases for the API-M deployment in a separate server and point to the databases from the respective nodes. 

For information, see [Installing and Configuring the Databases]({{base_path}}/install-and-setup/setup/setting-up-databases/overview/).

## Step 3 - Configure your deployment with production hardening

Ensure that you have taken into account the respective security hardening factors (e.g., changing and encrypting the default passwords, configuring JVM security, etc.) before deploying WSO2 API-M. 

For more information, see [Production Deployment Guidelines]({{base_path}}/install-and-setup/deploying-wso2-api-manager/production-deployment-guidelines/#common-guidelines-and-checklist).

## Step 4 - Create and import SSL certificates

Create an SSL certificate for each of the WSO2 API-M nodes and import them to the keystore and the truststore. This ensures that hostname mismatch issues in the certificates will not occur. 

!!! Note
    The same primary keystore should be used for all API Manager instances to decrypt the registry resources. For more information, see [Configuring the Primary Keystore]({{base_path}}/install-and-setup/setup/security/configuring-keystores/configuring-keystores-in-wso2-api-manager/#configuring-the-primary-keystore).

For more information, see [Creating SSL Certificates]({{base_path}}/install-and-setup/setup/security/configuring-keystores/keystore-basics/creating-new-keystores/).

## Step 5 - Configure API-M Analytics

API Manager Analytics is delivered via the API Manager Analytics cloud solution. You need to configure the API Manager Gateway to publish analytics data to the cloud.

See the instructions on [configuring the API Gateway]({{base_path}}/api-analytics/gateways/configure-synapse-gateway) with the cloud-based analytics solution.

## Step 6 - Configure and start the profiles

Let's configure the API-M nodes in the deployment.

### Configure the Gateway nodes

Configure the Gateway to communicate with the Control Plane and the Traffic Manager nodes.

Follow the instructions given below to configure the Gateway node so that it can communicate with the Control Plane node:

1. Open the `<API-M_HOME>/repository/conf/deployment.toml` file of the Gateway node.

2. Add the following configurations to the deployment.toml file.

   * **Connecting the Gateway to the Control Plane node**:
     
     ```toml tab="Control Plane with High Availability"
     # Key Manager configuration
     [apim.key_manager]
     service_url = "https://[control-plane-LB-host]/services/"
     username = "$ref{super_admin.username}"
     password = "$ref{super_admin.password}"

     # Event Listener configurations
     [[event_listener]]
     id = "token_revocation"
     type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
     name = "org.wso2.is.notification.ApimOauthEventInterceptor"
     order = 1

     [event_listener.properties]
     notification_endpoint = "https://[control-plane-LB-host]/internal/data/v1/notify"
     username = "${admin.username}"
     password = "${admin.password}"
     'header.X-WSO2-KEY-MANAGER' = "default"

     # Event Hub configurations
     [apim.event_hub]
     enable = true
     username = "$ref{super_admin.username}"
     password = "$ref{super_admin.password}"
     service_url = "https://[control-plane-LB-host]/services/"
     event_listening_endpoints = ["tcp://control-plane-1-host:5672", "tcp://control-plane-2-host:5672"]

     [[apim.event_hub.publish.url_group]]
     urls = ["tcp://control-plane-1-host:9611"]
     auth_urls = ["ssl://control-plane-1-host:9711"]

     [[apim.event_hub.publish.url_group]]
     urls = ["tcp://control-plane-2-host:9611"]
     auth_urls = ["ssl://control-plane-2-host:9711"]
     
     ```

     ```toml tab="Single Control Plane"
     # Key Manager configuration
     [apim.key_manager]
     service_url = "https://[control-plane-host]:${mgt.transport.https.port}/services/"
     username = "$ref{super_admin.username}"
     password = "$ref{super_admin.password}"

     # Event Listener configurations
     [[event_listener]]
     id = "token_revocation"
     type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
     name = "org.wso2.is.notification.ApimOauthEventInterceptor"
     order = 1

     [event_listener.properties]
     notification_endpoint = "https://[control-plane-host]:${mgt.transport.https.port}/internal/data/v1/notify"
     username = "${admin.username}"
     password = "${admin.password}"
     'header.X-WSO2-KEY-MANAGER' = "default"

     # Event Hub configurations
     [apim.event_hub]
     enable = true
     username = "$ref{super_admin.username}"
     password = "$ref{super_admin.password}"
     service_url = "https://[control-plane-host]:${mgt.transport.https.port}/services/"
     event_listening_endpoints = ["tcp://control-plane-host:5672"]

     [[apim.event_hub.publish.url_group]]
     urls = ["tcp://control-plane-host:9611"]
     auth_urls = ["ssl://control-plane-host:9711"]

     ```

    * **Connecting the Gateway to the Traffic Manager node**:
      
      ```toml tab="Traffic Manager with HA"
      [apim.throttling]
      service_url = "https://[traffic-manager-LB-host]/services/"
      throttle_decision_endpoints = ["tcp://traffic-manager-1-host:5672", "tcp://Traffic-Manager-2-host:5672"]

      [[apim.throttling.url_group]]
      traffic_manager_urls = ["tcp://traffic-manager-1-host:9611"]
      traffic_manager_auth_urls = ["ssl://traffic-manager-1-host:9711"]

      [[apim.throttling.url_group]]
      traffic_manager_urls = ["tcp://traffic-manager-2-host:9611"]
      traffic_manager_auth_urls = ["ssl://traffic-manager-2-host:9711"]

      ```

      ```toml tab="Single Traffic Manager"
      [apim.throttling]
      service_url = "https://[traffic-manager-host]:${mgt.transport.https.port}/services/"
      throttle_decision_endpoints = ["tcp://traffic-manager-host:5672"]

      [[apim.throttling.url_group]]
      traffic_manager_urls = ["tcp://traffic-manager-host:9611"]
      traffic_manager_auth_urls = ["ssl://traffic-manager-host:9711"]
      ```

3. Add the following configurations to the deployment.toml file to configure the Gateway environment. Change the `gateway_labels` property based on your Gateway environment.

    ```toml
    [apim.sync_runtime_artifacts.gateway]
    gateway_labels =["Default"]
    ```

4. Enable JSON Web Token (JWT) if required. For instructions, see [Generating JSON Web Token]({{base_path}}/deploy-and-publish/deploy-on-gateway/api-gateway/passing-enduser-attributes-to-the-backend-via-api-gateway).

5. Add the public certificate of the private key (that is used for signing the tokens) to the truststore under the "gateway_certificate_alias" alias. For instructions, see [Create and import SSL certificates]({{base_path}}/install-and-setup/setup/security/configuring-keystores/keystore-basics/creating-new-keystores).

    !!! Note
        This is not applicable if you use the default certificates, which are the certificates that are shipped with the product itself.

6. Follow the steps given below to configure High Availability (HA) for the API-M Gateway:

    1. Create a copy of the API-M Gateway node that you just configured. This is the second node of the API-M Gateway cluster.

    2. Configure a load balancer fronting the two Gateway nodes in your deployment. For instructions, see [Configuring the Proxy Server and the Load Balancer]({{base_path}}/install-and-setup/setup/setting-up-proxy-server-and-the-load-balancer/configuring-the-proxy-server-and-the-load-balancer).

        !!! Note
            To keep custom runtime artifacts deployed in the Gateway, add the following configuration in the `<API-M_HOME>/repository/conf/deployment.toml` file of the Gateway nodes.

            ```toml
            [apim.sync_runtime_artifacts.gateway.skip_list]
            apis = ["api1.xml","api2.xml"]
            endpoints = ["endpoint1.xml"]
            sequences = ["post_with_nobody.xml"]
            local-entries = ["file.xml"]

            ```

    3. Open the deployment.toml files of each Gateway node and add the cluster hostname. For example, if the hostname is `gw.am.wso2.com` the configuration will be:

        ```toml
        [server]
        hostname = "gw.wso2.com"

        ```


    4. Specify the following incoming connection configurations in the `deployment.toml` files of both nodes.

        ```toml
        [transport.http]
        properties.port = 9763
        properties.proxyPort = 80

        [transport.https]
        properties.port = 9443
        properties.proxyPort = 443

        ```

    5. Open the server's `/etc/hosts` file and map the hostnames to IPs.

        Format:
        
        ```
        <GATEWAY-IP> gw.wso2.com
        ```
 
        Example:
        
        ```
        xxx.xxx.xxx.xx4 gw.wso2.com
        ```

#### Sample configuration for the Gateway

```toml
[server]
hostname = "gw.wso2.com"
node_ip = "127.0.0.1"
server_role = "gateway-worker"
offset=0

[user_store]
type = "database_unique_id"

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[database.shared_db]
type = "h2"
url = "jdbc:h2:./repository/database/WSO2SHARED_DB;DB_CLOSE_ON_EXIT=FALSE"
username = "wso2carbon"
password = "wso2carbon"

[keystore.tls]
file_name =  "wso2carbon.jks"
type =  "JKS"
password =  "wso2carbon"
alias =  "wso2carbon"
key_password =  "wso2carbon"

[truststore]
file_name = "client-truststore.jks"
type = "JKS"
password = "wso2carbon"

# Key Manager configuration
[apim.key_manager]
service_url = "https://cp.wso2.com:9443/services/"
username = "$ref{super_admin.username}"
password = "$ref{super_admin.password}"

# Traffic Manager configurations
[apim.throttling]
service_url = "https://tm.wso2.com:9443/services/"
throttle_decision_endpoints = ["tcp://tm.wso2.com:5672"]

[[apim.throttling.url_group]]
traffic_manager_urls=["tcp://tm.wso2.com:9611"]
traffic_manager_auth_urls=["ssl://tm.wso2.com:9711"]

# Event Listener configurations
[[event_listener]]
id = "token_revocation"
type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
name = "org.wso2.is.notification.ApimOauthEventInterceptor"
order = 1

[event_listener.properties]
notification_endpoint = "https://cp.wso2.com:9443/internal/data/v1/notify"
username = "${admin.username}"
password = "${admin.password}"
'header.X-WSO2-KEY-MANAGER' = "default"

# Event Hub configurations
[apim.event_hub]
enable = true
username = "$ref{super_admin.username}"
password = "$ref{super_admin.password}"
service_url = "https://cp.wso2.com:9443/services/"
event_listening_endpoints = ["tcp://cp.wso2.com:5672"]

[[apim.event_hub.publish.url_group]]
urls = ["tcp://cp.wso2.com:9611"]
auth_urls = ["ssl://cp.wso2.com:9711"]

[apim.cors]
allow_origins = "*"
allow_methods = ["GET","PUT","POST","DELETE","PATCH","OPTIONS"]
allow_headers = ["authorization","Access-Control-Allow-Origin","Content-Type","SOAPAction"]
allow_credentials = false

[apim.sync_runtime_artifacts.gateway]
gateway_labels =["Default"]

```

### Configure the Control Plane nodes

Follow the steps given below to configure the Control Plane nodes to communicate with the Gateway.

1. Open the `<API-M_HOME>/repository/conf/deployment.toml` file of the Control Plane node.

2. Add the following configurations to the deployment.toml file.

    **Connecting the Control Plane to the Gateway node**:

    !!! Info
        This configuration is used for publishing APIs to the Gateway and for connecting the Developer Portal component to the Gateway.

    ```toml tab="Gateway with High Availability"

    [[apim.gateway.environment]]
    name = "Default"
    type = "hybrid"
    display_in_api_console = true
    description = "This is a hybrid gateway that handles both production and sandbox token traffic."
    show_as_token_endpoint_url = true
    service_url = "https://[api-gateway-LB-host]/services/"
    ws_endpoint = "ws://[api-gateway-LB-host-or-ip]:9099"
    wss_endpoint = "wss://[api-gateway-LB-host-or-ip]:8099"
    http_endpoint = "http://[api-gateway-LB-host]"
    https_endpoint = "https://[api-gateway-LB-host]"

    ```

    ```toml tab="Single gateway"
    [[apim.gateway.environment]]
    name = "Default"
    type = "hybrid"
    display_in_api_console = true
    description = "This is a hybrid gateway that handles both production and sandbox token traffic."
    show_as_token_endpoint_url = true
    service_url = "https://[api-gateway-host]:9443/services/"
    ws_endpoint = "ws://[api-gateway-host]:9099"
    wss_endpoint = "wss://[api-gateway-host]:8099"
    http_endpoint = "http://[api-gateway-host]:${http.nio.port}"
    https_endpoint = "https://[api-gateway-host]:${https.nio.port}"

    ```

    **Connecting the Control Plane to the Traffic Manager node**:

    !!! Info
        This configuration enables the publishing of throttling policies, custom templates, block conditions, and API events to the Traffic Manager node.

    ```toml tab="Traffic Manager with HA"

    [apim.throttling]
    username = "$ref{super_admin.username}"
    password = "$ref{super_admin.password}"
    enable_data_publishing = true
    service_url = "https://[traffic-manager-LB-host]/services/"
    event_duplicate_url = ["tcp://control-plane-2-host:5672"]

    [[apim.throttling.url_group]]
    traffic_manager_urls = ["tcp://traffic-manager-1-host:9611"]
    traffic_manager_auth_urls = ["ssl://traffic-manager-1-host:9711"]

    [[apim.throttling.url_group]]
    traffic_manager_urls = ["tcp://traffic-manager-2-host:9611"]
    traffic_manager_auth_urls = ["ssl://traffic-manager-2-host:9711"]

    ```

    ```toml tab="Single Traffic Manager"

    [apim.throttling]
    username = "$ref{super_admin.username}"
    password = "$ref{super_admin.password}"
    enable_data_publishing = true
    service_url = "https://[traffic-manager-host]:${mgt.transport.https.port}/services/"

    [[apim.throttling.url_group]]
    traffic_manager_urls = ["tcp://traffic-manager-host:9611"]
    traffic_manager_auth_urls = ["ssl://traffic-manager-host:9711"]

    ```

    !!! Note
        Configure the `event_duplicate_url` if the Control Plane is configured for High Availability (HA).

    **Add Event Listener and Event Hub Configurations**:

    ```toml tab="Control Plane with High Availability"

    # Event Listener configurations
    [[event_listener]]
    id = "token_revocation"
    type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
    name = "org.wso2.is.notification.ApimOauthEventInterceptor"
    order = 1

    [event_listener.properties]
    notification_endpoint = "https://[control-plane-LB-host]/internal/data/v1/notify"
    username = "${admin.username}"
    password = "${admin.password}"
    'header.X-WSO2-KEY-MANAGER' = "default"

    # Event Hub configurations
    [apim.event_hub]
    enable = true
    username= "$ref{super_admin.username}"
    password= "$ref{super_admin.password}"
    service_url = "https://localhost:${mgt.transport.https.port}/services/"
    event_listening_endpoints = ["tcp://localhost:5672"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-1-host:9611"]
    auth_urls = ["ssl://control-plane-1-host:9711"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-2-host:9611"]
    auth_urls = ["ssl://control-plane-2-host:9711"]

    ```

    ```toml tab="Single Control Plane"

    # Event Listener configurations
    [[event_listener]]
    id = "token_revocation"
    type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
    name = "org.wso2.is.notification.ApimOauthEventInterceptor"
    order = 1

    [event_listener.properties]
    notification_endpoint = "https://[control-plane-host]:${mgt.transport.https.port}/internal/data/v1/notify"
    username = "${admin.username}"
    password = "${admin.password}"
    'header.X-WSO2-KEY-MANAGER' = "default"

    # Event Hub configurations
    [apim.event_hub]
    enable = true
    username= "$ref{super_admin.username}"
    password= "$ref{super_admin.password}"
    service_url = "https://localhost:${mgt.transport.https.port}/services/"
    event_listening_endpoints = ["tcp://localhost:5672"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-host:9611"]
    auth_urls = ["ssl://control-plane-host:9711"]

    ```

3. If required, encrypt the Auth Keys (access tokens, client secrets, and authorization codes), see [Encrypting OAuth Keys]({{base_path}}/design/api-security/oauth2/encrypting-oauth2-tokens).

4. Optionally, add the following configuration to enable distributed cache invalidation within the Control Plane nodes.

    ```toml
    [apim.cache_invalidation]
    enabled = true
    domain = "control-plane-domain"

    ```

5. Follow the steps given below to configure High Availability (HA) for the Control Plane:

    1. Create a copy of the API-M Control Plane node that you just configured. This is the second node of the API-M Control Plane cluster.

    2. Configure a load balancer fronting the two Control Plane nodes in your deployment. For instructions, see [Configuring the Proxy Server and the Load Balancer]({{base_path}}/install-and-setup/setup/setting-up-proxy-server-and-the-load-balancer/configuring-the-proxy-server-and-the-load-balancer).

#### Sample configuration for the Control Plane

```toml
[server]
hostname = "cp.wso2.com"
node_ip = "127.0.0.1"
server_role = "control-plane"
offset=0

[user_store]
type = "database_unique_id"

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[database.apim_db]
type = "mysql"
hostname = "db.wso2.com"
name = "apim_db"
port = "3306"
username = "root"
password = "root"

[database.shared_db]
type = "mysql"
hostname = "db.wso2.com"
name = "shared_db"
port = "3306"
username = "root"
password = "root"

[keystore.tls]
file_name =  "wso2carbon.jks"
type =  "JKS"
password =  "wso2carbon"
alias =  "wso2carbon"
key_password =  "wso2carbon"

# Gateway configuration
[[apim.gateway.environment]]
name = "Default"
type = "hybrid"
display_in_api_console = true
description = "This is a hybrid gateway that handles both production and sandbox token traffic."
show_as_token_endpoint_url = true
service_url = "https://gw.wso2.com:9443/services/"
username= "${admin.username}"
password= "${admin.password}"
ws_endpoint = "ws://gw.wso2.com:9099"
wss_endpoint = "wss://gw.wso2.com:8099"
http_endpoint = "http://gw.wso2.com:8280"
https_endpoint = "https://gw.wso2.com:8243"

# Traffic Manager configurations
[apim.throttling]
username= "$ref{super_admin.username}"
password= "$ref{super_admin.password}"
enable_data_publishing = true
service_url = "https://tm.wso2.com:9443/services/"
throttle_decision_endpoints = ["tcp://tm.wso2.com:5672"]

[[apim.throttling.url_group]]
traffic_manager_urls=["tcp://tm.wso2.com:9611"]
traffic_manager_auth_urls=["ssl://tm.wso2.com:9711"]

# Event Listener configurations
[[event_listener]]
id = "token_revocation"
type = "org.wso2.carbon.identity.core.handler.AbstractIdentityHandler"
name = "org.wso2.is.notification.ApimOauthEventInterceptor"
order = 1

[event_listener.properties]
notification_endpoint = "https://cp.wso2.com:9443/internal/data/v1/notify"
username = "${admin.username}"
password = "${admin.password}"
'header.X-WSO2-KEY-MANAGER' = "default"

# Event Hub configurations
[apim.event_hub]
enable = true
username = "$ref{super_admin.username}"
password = "$ref{super_admin.password}"
service_url = "https://cp.wso2.com:9443/services/"
event_listening_endpoints = ["tcp://cp.wso2.com:5672"]

[[apim.event_hub.publish.url_group]]
urls = ["tcp://cp.wso2.com:9611"]
auth_urls = ["ssl://cp.wso2.com:9711"]

[apim.cors]
allow_origins = "*"
allow_methods = ["GET","PUT","POST","DELETE","PATCH","OPTIONS"]
allow_headers = ["authorization","Access-Control-Allow-Origin","Content-Type","SOAPAction"]
allow_credentials = false

```

### Configure the Traffic Manager Nodes

In a typical distributed deployment, all API-M components (excluding the API-M Gateway) run in the Control Plane. However, you have the option of separating the Traffic Manager from the Control Plane.

Configure the Traffic Manager to communicate with the Control Plane.

1. Open the `<API-M_HOME>/repository/conf/deployment.toml` file of the Traffic Manager node.

2. Add the following configurations to the deployment.toml file.

    **Connecting the Traffic Manager to the Control Plane node**:

    ```toml tab="Control Plane with High Availability"
 
    # Event Hub configurations
    [apim.event_hub]
    enable = true
    username = "$ref{super_admin.username}"
    password = "$ref{super_admin.password}"
    service_url = "https://[control-plane-LB-host]/services/"
    event_listening_endpoints = ["tcp://control-plane-1-host:5672", "tcp://control-plane-2-host:5672"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-1-host:9611"]
    auth_urls = ["ssl://control-plane-1-host:9711"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-2-host:9611"]
    auth_urls = ["ssl://control-plane-2-host:9711"] 

    ```
 
    ```toml tab="Single Control Plane"
 
    # Event Hub configurations
    [apim.event_hub]
    enable = true
    username = "$ref{super_admin.username}"
    password = "$ref{super_admin.password}"
    service_url = "https://[control-plane-host]/services/"
    event_listening_endpoints = ["tcp://control-plane-host:5672"]

    [[apim.event_hub.publish.url_group]]
    urls = ["tcp://control-plane-host:9611"]
    auth_urls = ["ssl://control-plane-host:9711"]

    ```

    If the Traffic manager node is configured with High Availability (HA), configure throttling as follows.

    ```toml

    [apim.throttling]
    event_duplicate_url = ["tcp://traffic-manager-2-host:5672"]
    service_url = "https://[traffic-manager-LB-host]/services/"
    throttle_decision_endpoints = ["tcp://localhost:5672"]

    [[apim.throttling.url_group]]
    traffic_manager_urls = ["tcp://traffic-manager-1-host:9611"]
    traffic_manager_auth_urls = ["ssl://traffic-manager-1-host:9711"]

    [[apim.throttling.url_group]]
    traffic_manager_urls = ["tcp://traffic-manager-2-host:9611"]
    traffic_manager_auth_urls = ["ssl://traffic-manager-2-host:9711"]

    ```

    !!! Note
        The `event_duplicate_url` should be added in order to publish events to the other node.

3. Follow the steps given below to configure High Availability (HA) for the Traffic Manager.

    1. Create a copy of the Traffic Manager node that you just configured. This is the second node of the Traffic Manager cluster.

    2. Configure a load balancer fronting the two Traffic Manager nodes in your deployment.

        !!! Note
            In each startup of a Traffic Manager node, the rate-limiting policies are redeployed by retrieving the latest policy details from the database. This maintains the consistency between the Traffic Manager nodes. If you need to avoid redeploying certain rate-limiting policies, add the following configuration to the `<API-M_HOME>/repository/conf/deployment.toml` file in the Traffic Manager node.

            ```toml
            [apim.throttling]
            skip_redeploying_policies = ["throttle_policy_1","throttle_policy_2"]

            ```

#### Sample configuration for the Traffic Manager

```toml
[server]
hostname = "tm.wso2.com"
node_ip = "127.0.0.1"
server_role = "traffic-manager"
offset=0

[user_store]
type = "database_unique_id"

[super_admin]
username = "admin"
password = "admin"
create_admin_account = true

[database.apim_db]
type = "h2"
url = "jdbc:h2:./repository/database/WSO2AM_DB;AUTO_SERVER=TRUE;DB_CLOSE_ON_EXIT=FALSE"
username = "wso2carbon"
password = "wso2carbon"

[database.shared_db]
type = "h2"
url = "jdbc:h2:./repository/database/WSO2SHARED_DB;DB_CLOSE_ON_EXIT=FALSE"
username = "wso2carbon"
password = "wso2carbon"

[keystore.tls]
file_name =  "wso2carbon.jks"
type =  "JKS"
password =  "wso2carbon"
alias =  "wso2carbon"
key_password =  "wso2carbon"

[truststore]
file_name = "client-truststore.jks"
type = "JKS"
password = "wso2carbon"

# Event Hub configurations
[apim.event_hub]
enable = true
username = "$ref{super_admin.username}"
password = "$ref{super_admin.password}"
service_url = "https://cp.wso2.com:9443/services/"
event_listening_endpoints = ["tcp://cp.wso2.com:5672"]

[[apim.event_hub.publish.url_group]]
urls = ["tcp://cp.wso2.com:9611"]
auth_urls = ["ssl://cp.wso2.com:9711"]

```

## Step 7 - Start the API-M nodes

Once you have successfully configured all the API-M nodes in the deployment, you can start the servers.

-   Starting the Gateway nodes

    Open a terminal, navigate to the `<API-M-GATEWAY-HOME>/bin` folder, and execute the following command:
    
    ``` java tab="Linux/Mac OS"
    cd <API-M_HOME>/bin/
    sh api-manager.sh -Dprofile=gateway-worker
    ```
    
    ``` java tab="Windows"
    cd <API-M_HOME>\bin\
    api-manager.bat --run -Dprofile=gateway-worker
    ```

-   Start the Control Plane nodes

    Open a terminal, navigate to the `<API-M-CONTROL-PLANE-HOME>/bin` folder, and execute the following command:

    ``` java tab="Linux/Mac OS"
    cd <API-M_HOME>/bin/
    sh api-manager.sh -Dprofile=control-plane
    ```

    ``` java tab="Windows"
    cd <API-M_HOME>\bin\
    api-manager.bat --run -Dprofile=control-plane
    ```

-   Start the Traffic Manager nodes

    Open a terminal, navigate to the `<API-M-TRAFFIC-MANAGER-HOME>/bin` folder, and execute the following command:

    ``` java tab="Linux/Mac OS"
    cd <API-M_HOME>/bin/
    sh api-manager.sh -Dprofile=traffic-manager
    ```

    ``` java tab="Windows"
    cd <API-M_HOME>\bin\
    api-manager.bat --run -Dprofile=traffic-manager
    ```

For more information on starting API-M profiles, see [API-M Profiles]({{base_path}}/install-and-setup/setup/distributed-deployment/product-profiles).
