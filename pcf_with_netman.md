# DEPRECATED: Please follow [Pivotal documentation](http://docs.pivotal.io/pivotalcf/1-10/devguide/deploy-apps/cf-networking.html) to enable container networking 
# Deploying netman to Pivotal Cloud Foundry

## Pre-requisites
This document assumes that you are familiar with Ops Manager and Elastic Runtime. You know how to SSH to the OpsMan VM and run BOSH commands against the BOSH director.

These instructions work best on a greenfield deployment using the versions that have been validated. When applied to an existing environment, you will need to `recreate` your cells for container networking policies to be applied to existing apps, and may need to upgrade `cf-release` and `diego-release` to match those specified in netman [release-notes](https://github.com/cloudfoundry-incubator/netman-release/releases). 

## Tested Versions
- Ops Manager : v1.8.9.0
- Elastic Runtime : v1.8.11
- netman-release : v0.5.0

## Step 1 - Deploy Ops Manager and ERT
Follow Pivotal [documentation](https://docs.pivotal.io/pivotalcf/1-8/installing/) to install Ops Manager, the BOSH director and Elastic Runtime. 

## Step 2 - Access BOSH director
Follow [these instructions](https://docs.pivotal.io/pivotalcf/1-7/customizing/trouble-advanced.html) for accessing the BOSH director from the OpsMan VM.

## Step 3 - Upload netman release
From the BOSH Director command line, follow the insturctions on [bosh.io](http://bosh.io/releases/github.com/cloudfoundry-incubator/netman-release?all=1) to upload netman release.

## Step 4 - Edit deployment manifest
Run `bosh deployments` to get a list of deployments. The CF deployment is prefixed with `cf-`. For example, in our setup it is `cf-0caa8053ec39e45fcd72`

The deployment manifest can be found under `/var/tempest/workspaces/default/deployments/` and will have the same name as the manifest. In our example the file is `cf-0caa8053ec39e45fcd72.yml`. Use your favorite CLI editor to change the following items in the manifest: 

### Releases
```diff
+- name: netman
+  version: latest
```

### Instance Groups
In `mysql.properties.cf_mysql.mysql.seeded_databases:` add a new database:
```diff
+        - name: policy_server
+          username: policy_server
+          password: DATABASE_PASSWORD
```

In `uaa.properties.uaa.clients.cf`, modify the list of `scopes` to include `network.admin`:
```diff
-          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,routing.router_groups.write
+          scope: cloud_controller.read,cloud_controller.write,openid,password.write,cloud_controller.admin,scim.read,scim.write,doppler.firehose,uaa.user,routing.router_groups.read,routing.router_groups.write,network.admin
```

In `uaa.properties.uaa.clients`, add a new client called `network-policy`:
```diff
+        network-policy:
+          authorities: uaa.resource
+          secret: NETWORK_POLICY_SERVER_UAA_CLIENT_SECRET
```


In `uaa.properties.uaa.scim.users`, modify the `admin` user to belong to a new group called `network.admin`:
```diff
           - console.admin
           - console.support
           - doppler.firehose
+          - network.admin
           - notification_preferences.read
           - notification_preferences.write
           - notifications.manage
```

Add a new instance group called `policy-server` which includes the `consul` job with the standard `consul` encryption key and certificates.

```diff
+- instances: 1
+  name: policy-server
+  vm_type: medium.mem
+  lifecycle: service
+  stemcell: bosh-vsphere-esxi-ubuntu-trusty-go_agent
+  azs:
+  - default
+  properties:
+    consul:
+      encrypt_keys:
+      - CONSUL_ENCRYPT_KEY
+      ca_cert: CONSUL_CA_CERT
+      agent_cert: CONSUL_AGENT_CERT
+      agent_key: CONSUL_AGENT_KEY
+      server_cert: CONSUL_SERVER_CERT
+      server_key: CONSUL_SERVER_KEY
+      agent:
+        domain: cf.internal
+        services:
+          policy-server:
+            check:
+              interval: 5s
+              script: "/bin/true"
+            name: policy-server
+        servers:
+          lan:
+          - CONSUL_LAN_IP
+    nats:
+      user: NATS_USERNAME
+      password: NATS_PASSWORD
+      port: 4222
+      machines:
+      - NATS_IP
+    route_registrar:
+      routes:
+      - name: policy-server
+        port: 4002
+        registration_interval: 20s
+        uris:
+        - api.SYSTEM_DOMAIN/networking
+    policy-server:
+      ca_cert: POLICY_SERVER_CA_CERT
+      server_cert: POLICY_SERVER_SERVER_CERT
+      server_key: POLICY_SERVER_SERVER_KEY
+      database:
+        connection_string: policy_server:DATABASE_PASSWORD@tcp(MYSQL_PROXY_IP:3306)/policy_server
+        type: mysql
+      skip_ssl_validation: true
+      uaa_client_secret: NETWORK_POLICY_SERVER_UAA_CLIENT_SECRET
+      uaa_url: https://uaa.SYSTEM_DOMAIN
+  templates:
+  - name: policy-server
+    release: netman
+  - name: route_registrar
+    release: cf
+  - name: consul_agent
+    release: cf
+  env:
+    bosh:
+      password: ENV_BOSH_PASSWORD
+  update:
+    serial: true
+    max_in_flight: 1
+  networks:
+  - name: default
+    default:
+    - dns
+    - gateway
+  persistent_disk_type: '1024'
```

In the `diego_cell` instance group...

1. Add these jobs:

  ```diff
  +  - name: garden-cni
  +    release: netman
  +    consumes: {}
  +    provides: {}
  +  - name: cni-flannel
  +    release: netman
  +    consumes: {}
  +    provides: {}
  +  - name: vxlan-policy-agent
  +    release: netman
  +    consumes: {}
  +    provides: {}
  +  - name: netmon
  +    release: netman
  +    consumes: {}
  +    provides: {}
  ```

2. Edit `properties.garden` to include:

  ```diff
  +      network_plugin: "/var/vcap/packages/runc-cni/bin/garden-external-networker"
  +      network_plugin_extra_args:
  +      - "--configFile=/var/vcap/jobs/garden-cni/config/adapter.json"
  ```

3. Add to `properties`
  ```diff
  +    cni-flannel:
  +      etcd_ca_cert:
  +      etcd_client_cert:
  +      etcd_client_key:
  +      etcd_endpoints:
  +      - ETCD_IP
  +      flannel:
  +        etcd:
  +          require_ssl: false
  +    vxlan-policy-agent:
  +      policy_server_url: https://policy-server.service.cf.internal:4003
  +      ca_cert: VXLAN_POLICY_AGENT_CA_CERT
  +      client_cert: VXLAN_POLICY_AGENT_CLIENT_CERT
  +      client_key: VXLAN_POLICY_AGENT_CLIENT_KEY
  +    garden-cni:
  +      cni_config_dir: "/var/vcap/jobs/cni-flannel/config/cni"
  +      cni_plugin_dir: "/var/vcap/packages/flannel/bin"
  ```

  NOTE: The client certificate and key that the `vxlan-policy-agent` presents to the `policy-server` must be generated from the same `ca_cert` that is used to generate the server certificate and key for the `policy-server`.
  There is a [script in netman-release](https://github.com/cloudfoundry-incubator/netman-release/blob/develop/scripts/generate-certs) to generate certs/keys for this.

## Step 5 - Re-deploy Elastic Runtime
Set the BOSH deployment to the edited manifest and deploy. For example:
```
bosh deployment cf-0caa8053ec39e45fcd72.yml
bosh deploy
```
For a brownfield deployment provide the `--recreate` option while deploying to recreate the Diego cells. 

## Step 6 - Try container networking
If the deployment succeeds, validate container networking is working by trying the [Cats & Dogs](https://github.com/cloudfoundry-incubator/netman-release/blob/develop/src/example-apps/cats-and-dogs) example.
