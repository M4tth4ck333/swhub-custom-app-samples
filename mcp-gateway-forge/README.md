# deploy mcp-context-forge app to software hub dataplane

[ContextForge MCP Gateway](https://github.com/IBM/mcp-context-forge) is a feature-rich gateway, proxy and MCP Registry that federates MCP and REST services - unifying discovery, auth, rate-limiting, observability, virtual servers, multi-transport protocols, and an optional Admin UI into one clean endpoint for your AI clients

Checkout the [pre-requisites](../README.md#pre-requisites-to-deploy-sample-applications) required before deploying this application.

### deploy mcp-context-forge with sql lite database:  

- download proxy-config-yaml file:  
  ```
  download ./mcp-gateway-forge.conf.yaml and move to cpd-cli-workspace/olm-utils-workspace/work/mcp-gateway-forge.conf.yaml
  ```

- create app-envs-json-sqlite.json:  

  ```
  $ cat cpd-cli-workspace/olm-utils-workspace/work/app-envs-json-sqlite.json

    # cpd 5.3.0  
    {"HOST": "0.0.0.0", "JWT_SECRET_KEY": "zen-phy-loc-broker-secret-token", "BASIC_AUTH_USER": "admin@example.com", "BASIC_AUTH_PASSWORD": "changeme", "AUTH_REQUIRED": "true", "DATABASE_URL": "sqlite:////data/mcp.db", "SSL": "true", "CERT_FILE": "/etc/certs/tls.crt", "KEY_FILE": "/etc/certs/tls.key", "MCPGATEWAY_UI_ENABLED": "true", "MCPGATEWAY_ADMIN_API_ENABLED": "true"}

    # cpd 5.4.0  
    [{"name":"HOST","value":"0.0.0.0"},{"name": "JWT_SECRET_KEY", "valueFrom": {"secretKeyRef": {"name": "zen-phy-loc-broker-secret", "key": "token"}}},{"name":"BASIC_AUTH_USER","value":"admin@example.com"},{"name":"BASIC_AUTH_PASSWORD","value":"changeme"},{"name":"AUTH_REQUIRED","value":"true"},{"name":"DATABASE_URL","value":"sqlite:////data/mcp.db"},{"name":"SSL","value":"true"},{"name":"CERT_FILE","value":"/etc/certs/tls.crt"},{"name":"KEY_FILE","value":"/etc/certs/tls.key"},{"name": "MCPGATEWAY_UI_ENABLED","value":"true"},{"name":"MCPGATEWAY_ADMIN_API_ENABLED","value":"true"}]
  ```
  ***Note:***
    when adding mcp gateway server with oauth enabled, need to create a secret volumeMount i.e. /etc/ca-certs/ca.crt and add the following env variable:  
    `{"name": "SSL_CERT_FILE", "value": "/etc/ca-certs/ca.crt"}`  
    this is temporary workaround, this is needed beyond already adding ca certificate when adding mcp gateway server in mcpgateway admin console

- run cpd-cli create-dockerfile-application:  
  ```
  cpd-cli manage create-dockerfile-application --instance_ns=zen \
    --app_name=mcp-context-forge-sqlite  \
    --app_port=4444 \
    --app_port_tls=true \
    --repo_url=https://github.com/IBM/mcp-context-forge.git \
    --repo_branch=v1.0.0-RC2  \
    --dockerfile=Containerfile  \
    --app_envs_json=/tmp/work/app-envs-json-sqlite.json \
    --pvc_info={"size":"2Gi","mount_path":"/data"}  \
    --cpu=400m  \
    --memory=400Mi  \
    --cpu_limit=800m  \
    --memory_limit=2Gi
  ```  
  ***Note:***
    --repo_branch=v1.0.0-RC2 can be used together with --repo_url=https://github.com/IBM/mcp-context-forge.git in cpd 5.4.0, otherwise https://github.com/IBM/mcp-context-forge.git need to be forked/cloned somewhere else with branch created based on tag v1.0.0-RC2

- check that the pod is in running status:
  ```
  mcp-context-forge-sqlite-tcv6isrcslka-57bcbb95b-dshrl            1/1     Running
  ```  

- update application proxy config - copy `mcp-gateway-forge.conf.yaml` in this folder to `cpd-cli-workspace/olm-utils-workspace/work/`  
  ```
  ./cpd-cli manage update-custom-application-proxy-config \
  --instance_ns=zen \
  --app_name= mcp-context-forge-sqlite \
  --app_run_id=<from create-dockerfile-application> \
  --app_proxy_config_yaml=/tmp/work/mcp-gateway-forge.conf.yaml
  ```

### deploy mcp-context-forge with postgres database:  

  #### deploy postgresql  

  - download postgresql template tar
    ```
    download postgresql.template.tgz to cpd-cli-workspace/olm-utils-workspace/work/postgesql.template.tgz
    ```

  - run following cpd-cli command:  
    ```
    ./cpd-cli manage create-oc-template-application --instance_ns=zen  \
      --app_name=postgresql-mcp-context-forge  \
      --app_tar_file=/tmp/work/postgesql.template.tgz  \
      --cpu=400m  \
      --memory=200Mi  \
      --cpu_limit=500m  \
      --memory_limit=400Mi
    ```

  - check that the pod is in running state:  
    ```
    # oc get po -n wl | grep postgres
    postgresql-mcp-context-forge-1-g4ld7                              1/1     Running
    ```

  #### deploy mcp-context-forge

  - create app-envs-json.json:  
    ```
    $ cat cpd-cli-workspace/olm-utils-workspace/work/app-envs-json-postgresql.json  

    # cpd 5.3.0  
    {"HOST":"0.0.0.0","JWT_SECRET_KEY":"zen-phy-loc-broker-secret-token","BASIC_AUTH_USER":"admin@example.com","BASIC_AUTH_PASSWORD":"changeme","AUTH_REQUIRED":"true","DATABASE_URL":"postgresql+psycopg://postgres:secret@postgresql-mcp-context-forge:5432/mcp","SSL":"true","CERT_FILE":"/etc/certs/tls.crt","KEY_FILE":"/etc/certs/tls.key","MCPGATEWAY_UI_ENABLED":"true","MCPGATEWAY_ADMIN_API_ENABLED":"true"}

    # cpd 5.4.0  
      [{"name":"HOST","value":"0.0.0.0"},{"name": "JWT_SECRET_KEY", "valueFrom": {"secretKeyRef": {"name": "zen-phy-loc-broker-secret", "key": "token"}}},{"name":"BASIC_AUTH_USER","value":"admin@example.com"},{"name":"BASIC_AUTH_PASSWORD","value":"changeme"},{"name":"AUTH_REQUIRED","value":"true"},{"name":"DATABASE_URL","value":"postgresql+psycopg://postgres:secret@postgresql-mcp-context-forge:5432/mcp"},{"name":"SSL","value":"true"},{"name":"CERT_FILE","value":"/etc/certs/tls.crt"},{"name":"KEY_FILE","value":"/etc/certs/tls.key"},{"name": "MCPGATEWAY_UI_ENABLED","value":"true"},{"name":"MCPGATEWAY_ADMIN_API_ENABLED","value":"true"}]
    ```  
    ***Note:***
      when adding mcp gateway server with oauth enabled, need to create a secret volumeMount i.e. /etc/ca-certs/ca.crt and add the following env variable:  
        `{"name": "SSL_CERT_FILE", "value": "/etc/ca-certs/ca.crt"}`  
      this is temporary workaround, this is needed beyond already adding ca certificate when adding mcp gateway server in mcpgateway admin console

  - run cpd-cli create-dockerfile-application command:  
    ```
    cpd-cli manage create-dockerfile-application --instance_ns=zen \
      --app_name=mcp-context-forge-postgresql  \
      --app_port=4444 \
      --app_port_tls=true \
      --repo_url=https://github.com/IBM/mcp-context-forge.git \
      --repo_branch=v1.0.0-RC2  \
      --dockerfile=Containerfile  \
      --app_envs_json=/tmp/work/app-envs-json-postgresql.json \
      --cpu=400m  \
      --memory=400Mi  \
      --cpu_limit=800m  \
      --memory_limit=2Gi
    ```  
    ***Note:***
      --repo_branch=v1.0.0-RC2 can be used together with --repo_url=https://github.com/IBM/mcp-context-forge.git in cpd 5.4.0, otherwise https://github.com/IBM/mcp-context-forge.git need to be forked/cloned somewhere else with branch created based on tag v1.0.0-RC2

  - check that the pod is in running status:
    ```
    mcp-context-forge-postgresql-78bib026k314-57bcbb95b-dshrl            1/1     Running
    ```  
  - update application proxy config - copy `mcp-gateway-forge.conf.yaml` in this folder to `cpd-cli-workspace/olm-utils-workspace/work/`  
    ```
    ./cpd-cli manage update-custom-application-proxy-config \
      --instance_ns=zen \
      --app_name= mcp-context-forge-postgresql \
      --app_run_id=<from create-dockerfile-application> \
      --app_proxy_config_yaml=/tmp/work/mcp-gateway-forge.conf.yaml
    ```

 #### application available at `https://zen-route/physical_location/default-pl/<app_name-app_run_id>/admin`
