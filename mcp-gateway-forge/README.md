# deploy mcp-context-forge app to software hub dataplane

[ContextForge MCP Gateway](https://github.com/IBM/mcp-context-forge) is a feature-rich gateway, proxy and MCP Registry that federates MCP and REST services - unifying discovery, auth, rate-limiting, observability, virtual servers, multi-transport protocols, and an optional Admin UI into one clean endpoint for your AI clients

Checkout the [pre-requisites](../README.md#pre-requisites-to-deploy-sample-applications) required before deploying this application.

this instruction requires cpd 5.4.0 or cpd 5.3.1 patch 2 zen and olm-utils. 

### deploy mcp-context-forge with sql lite database:  

- download proxy-config-yaml file:  
  ```
  download ./mcp-gateway-forge.conf.yaml and move to cpd-cli-workspace/olm-utils-workspace/work/mcp-gateway-forge.conf.yaml
  ```

- create app-envs-json-sqlite.json:  
  ```
  $ cat cpd-cli-workspace/olm-utils-workspace/work/app-envs-json-sqlite.json 
    [{"name":"HOST","value":"0.0.0.0"},{"name": "JWT_SECRET_KEY", "valueFrom": {"secretKeyRef": {"name": "zen-phy-loc-broker-secret", "key": "token"}}},{"name":"BASIC_AUTH_USER","value":"admin@example.com"},{"name":"BASIC_AUTH_PASSWORD","value":"changeme"},{"name":"AUTH_REQUIRED","value":"true"},{"name":"REQUIRE_JTI","value":"false"},{"name":"DATABASE_URL","value":"sqlite:////data/mcp.db"},{"name":"SSL","value":"true"},{"name":"CERT_FILE","value":"/etc/certs/tls.crt"},{"name":"KEY_FILE","value":"/etc/certs/tls.key"},{"name": "MCPGATEWAY_UI_ENABLED","value":"true"},{"name":"MCPGATEWAY_ADMIN_API_ENABLED","value":"true"},{"name":"LOCATION_NAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['icpdsupport/physicalLocationName']"}}},{"name":"COMPONENT","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['component']"}}},{"name":"APP_RUN_ID","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['icpdata_run_id']"}}}]
  ```
  ***Note:***
    when adding mcp gateway server with oauth enabled that uses self-signed certificate, need to create a secret volumeMount i.e. /etc/ca-certs/ca.crt and add the following env variable:  
    &emsp;`{"name": "SSL_CERT_FILE", "value": "/etc/ca-certs/ca.crt"}`  
    and mount the referenced volume path by adding `--volumes_mounts_json=/tmp/work/volumes-mounts.json`, where:
    ```
    $ cat /tmp/work/volumes-mounts.json
      {"volumes":[{"volume_name":"ca-certs","volume_type":"Secret","volume_source":"ca-certs","mount_path":"/etc/ca-certs"}]}  
    ```  
    this is temporary workaround, this is needed beyond already adding ca certificate when adding mcp gateway server in mcpgateway admin console

- create command-json.json file  
  ```
  $ cat cpd-cli-workspace/olm-utils-workspace/work/command-json.json  
    ["sh", "-c", "export APP_ROOT_PATH=/physical_location/$(LOCATION_NAME)/$(COMPONENT)-$(APP_RUN_ID);./docker-entrypoint.sh"]
  ```  
- set a default storageClass for this deployment as pvc will be created:  
  i.e. `oc patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`  
- run cpd-cli create-dockerfile-application:  
  ```
  cpd-cli manage create-dockerfile-application --instance_ns=zen \
    --app_name=mcp-context-forge-sqlite  \
    --app_port=4444 \
    --app_port_tls=true \
    --repo_url=https://github.com/IBM/mcp-context-forge.git \
    --repo_branch=v1.0.0-RC2  \
    --dockerfile=Containerfile  \
    --command_json=/tmp/work/command-json.json \
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
    [{"name":"HOST","value":"0.0.0.0"},{"name": "JWT_SECRET_KEY", "valueFrom": {"secretKeyRef": {"name": "zen-phy-loc-broker-secret", "key": "token"}}},{"name":"BASIC_AUTH_USER","value":"admin@example.com"},{"name":"BASIC_AUTH_PASSWORD","value":"changeme"},{"name":"AUTH_REQUIRED","value":"true"},{"name":"REQUIRE_JTI","value":"false"},{"name":"DATABASE_URL","value":"postgresql+psycopg://postgres:secret@postgresql-mcp-context-forge:5432/mcp"},{"name":"SSL","value":"true"},{"name":"CERT_FILE","value":"/etc/certs/tls.crt"},{"name":"KEY_FILE","value":"/etc/certs/tls.key"},{"name": "MCPGATEWAY_UI_ENABLED","value":"true"},{"name":"MCPGATEWAY_ADMIN_API_ENABLED","value":"true"}, {"name":"LOCATION_NAME","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['icpdsupport/physicalLocationName']"}}},{"name":"COMPONENT","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['component']"}}},{"name":"APP_RUN_ID","valueFrom":{"fieldRef":{"apiVersion":"v1","fieldPath":"metadata.labels['icpdata_run_id']"}}}]
    ```  
    ***Note:***
      when adding mcp gateway server with oauth enabled that uses self-signed certificate, need to create a secret volumeMount i.e. /etc/ca-certs/ca.crt and add the following env variable:  
      &emsp;`{"name": "SSL_CERT_FILE", "value": "/etc/ca-certs/ca.crt"}`  
      and mount the referenced volume path by adding `--volumes_mounts_json=/tmp/work/volumes-mounts.json`, where:
      ```
      $ cat /tmp/work/volumes-mounts.json
        {"volumes":[{"volume_name":"ca-certs","volume_type":"Secret","volume_source":"ca-certs","mount_path":"/etc/ca-certs"}]}  
      ```  
      this is temporary workaround, this is needed beyond already adding ca certificate when adding mcp gateway server in mcpgateway admin console

  - create command-json.json file  
    ```
    $ cat cpd-cli-workspace/olm-utils-workspace/work/command-json.json  
      ["sh", "-c", "export APP_ROOT_PATH=/physical_location/$(LOCATION_NAME)/$(COMPONENT)-$(APP_RUN_ID);./docker-entrypoint.sh"]
    ```  
  - set a default storageClass for this deployment as pvc will be created:  
    i.e. `oc patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`
    
  - run cpd-cli create-dockerfile-application command:  
    ```
    cpd-cli manage create-dockerfile-application --instance_ns=zen \
      --app_name=mcp-context-forge-postgresql  \
      --app_port=4444 \
      --app_port_tls=true \
      --repo_url=https://github.com/IBM/mcp-context-forge.git \
      --repo_branch=v1.0.0-RC2  \
      --dockerfile=Containerfile  \
      --command_json=/tmp/work/command-json.json \
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

### mcp-context-forge documentation is available at https://ibm.github.io/mcp-context-forge, for quick start:
1. default username and password is admin@example.com/changeme, you'd need to change default password on first login
2. use "mcp servers" tab to add mcp servers as mcp gateway service, if mcp servers added as streamablehttp server with oauth, you'd need to click "authorize" link for added mcp server gateway to authorize and fetch tools
3. use "virtual servers" tab to create a virtual server that you'd like to have available tools included for the server, virtual server is available at `https://zen-route/physical_location/default-pl/<app_name-app_run_id>/servers/<virtual-server-id>/mcp` that mcp client can connect to
4. use "api tokens" tab to create an api token that is required for mcp client to connect to mcp virtual server, token is bearer token:
  `Authorization: Bearer $API_TOKEN` is needed for mcp client to connect to mcp virtual server
   more permission related token scope can also be set, by default mcp tools are "public" to the token. 

### integrate with zen:
1. create secret called mcp-gateway-secret:  
  `oc create secret generic mcp-gatway-secret --from-literal=token='<mcp gateway api token>' -n <zen namespace>`. 
2. update mcp gateway url:  
  `oc set env deployment/platform-ai-agent MCP_GATEWAY_URL='<mcp gateway virtual server url>' -n <zen namespace>`. 
