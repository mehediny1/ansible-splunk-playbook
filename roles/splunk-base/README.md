# Base Node Overview and Purpose

The base Splunk role runs against all members of the specified inventory to install Splunk in a consistent manner. 
The **splunk-base** role configures server optimizations, directory structures, installs splunk, deploys certificates and keys, and sets default configuration parameters.

Server optimizations:  
Disabling transparent-huge-pages, setting swappiness, and ulimits for each node.

Directory structure:  
Configuring the correct directory structure, and preparing disks for storage volumes, and linking /opt/splunk to the appropriate location.

Install Splunk:  
Installing Splunk using a .deb package which takes care of adding the splunk user ad group among other things.

Deploy certificate and keys:  
Installing certificates allows for three options. One option is using default certs that are generated during the splunk install.
This will get an environment up and running but is not the most secure way to run a splunk cluster. Using the default certificates can leave an environment vulnerable to MITM attacks.
Another options is using the **splunk-ca** role, which creates an internal certificate authority(CA) where certs and keys are generated.
Those artifacts are then distributed via the **splunk-base** role to the appropriate machines for each specific role (indexers, search heads etc.)
A third option is to distribute certificates purchased from a third party in the same manner as running a local CA.

Set configuration parameters:  
Once the app is installed and certs are deployed, making the final configuration changes to different files is important for a flexible install.
Now the role creates **inputs.conf**, **web.conf**, **authentication.conf**, and **alert_actions.conf** in the system local directory on the appropriate server for each configuration.
Not all config file belong on all servers. For instance, the alert_actions.conf does not need ot be deployed to an indexer.
The app is started and the default license is accepted. Splunk's boot-start option is set so Splunk will start automatically after a server reboot.

Edits to the server.conf file are made to add in proper values for **serverCert**, **sslRootCAPath**, and **sslPassword**.  
NOTE: redefining the default sslPassword from "password" will cause grief if the CA is not setup correctly.

The last thing this role does is set the proper attribute for the license master and restarts splunk.

## All about certificates

The general process for manually handling certs and certificate signing requests(CSR), looks like this:
1. Generate private key on client node
2. Generate CSR on client node using clients private key
3. Send CSR to certificate authority(CA)
4. Generate client cert from CSR using CA key pair
5. Return CA public key and client cert

Create a directory for splunk certificates. In this example, **$SPLUNK_HOME/etc/auth/myOrg** is used:

When preparing certs for splunk, there should be three files:
  - splunk-server-cert.pem
  - splunk-server-priv-key.pem
  - ca-cert.pem

Combine the **client server certificate**, **client server private key**, and the **CA cert**, into a single PEM file, maintaining that specific order.
ex:
```
cat splunk-server-cert.pem splunk-server-priv-key.pem ca-pubkey.pem > splunk-server-cert-bundle.pem
```

Again, the contents of the **splunk-server-cert-bundle.pem** should contain, in the following order:
  - The client node certificate (splunk-server-cert.pem)
  - The client node private key (splunk-server-priv-key.pem)
  - The certificate authority cert file (ca-cert.pem)

Enabling the **splunk-ca** role will generate the appropriate key and cert fies necessary to secure the splunk backend services for each node.

Configure **inputs.conf** on the indexer(s) to use the new server certificate. In **$SPLUNK_HOME/etc/system/local/inputs.conf** (or in the appropriate directory of any app you are using to distribute your forwarding configuration).

Some examples of how this configuration maps to the splunk docs.
  > **clientCert** = The full path to the client SSL certificate in PEM format. If (and only if) specified, this connection will use SSL.  
**$SPLUNK_HOME/etc/auth/myOrg/splunk-server-cert-bundle.pem**

  > **serverCert** =  The full path to the PEM format server certificate file. Default certificates 
($SPLUNK_HOME/etc/auth/server.pem) are generated by Splunk at start. To secure Splunk, 
you should replace the default cert with your own PEM file.  
**$SPLUNK_HOME/etc/auth/myOrg/splunk-server-cert-bundle.pem**

  > **sslRootCAPath** = absolute path to the operating system's root CA (Certificate Authority) PEM 
format file containing one or more root CA. Do not configure this attribute on Windows.  
**$SPLUNK_HOME/etc/auth/myOrg/ca-cert.pem**

NOTE: If you have followed along with the Splunk documentation they leave out one critical bit of information when using your own CA.
You have to set the value for **serverCert** to the server.conf file otherwise devices will not be able to connect.

## Splunk Base Node Role Objectives
server optimizations, directory structures, installs splunk, deploys certificates and keys, and sets default configuration parameters
1. Configure the Splunk Base Nodes after splunk-base has run
    1. Configure disk partitions and mount filesystems
    2. Install splunk from a package management package(.deb)
    3. Create/Copy certificates and key for a secure Splunk environment
    4. Set configuration options to allow proper operation of a Splunk Cluster
    5. Setup option for LDAP

### The playbook will configure:
  - base
    - splunk-base : Disable transparent-huge-pages
    - splunk-base : Set swappiness to 10 in /etc/sysctl.conf
    - splunk-base : Set ulimits for splunk user
    - splunk-base : Create directory structure and setuo disk partitions using parted
    - splunk-base : Create and mount filesystem on device
    - splunk-base : Create symlink from /opt/splunk to /storage/splunk
    - splunk-base : Install Splunk from .deb package
    - splunk-base : Fix up perms after splunk user created
    - splunk-base : Sets default acl for splunk on /var/log
    - splunk-base : Distribute priv-key.pem, cert.pem, and ca-pubkey.pem to host
    - splunk-base : Remove a cert bundle if it exists to allow creation of new
    - splunk-base : Generate a cert bundle from artifacts or wildcard cert
    - splunk-base : Generate an OpenSSL private key for SplunkWeb
    - splunk-base : Generate an OpenSSL CSR for SplunkWeb
    - splunk-base : Generate a Self Signed OpenSSL certificate for SplunkWeb
    - splunk-base : Copy Splunk inputs.conf, web.conf, authentication.conf, and alert_actions.conf for system local
    - splunk-base : Create default admin user seed file
    - splunk-base : Accept Splunk license and set Splunk to start at boot using splunk user
    - splunk-base : Check passwd file contains current admin hash value
    - splunk-base : Reset Splunk admin password if changed
    - splunk-base : Set sslPassword using splunk_cluster_sslPassword
    - splunk-base : Add serverCert and sslRootCAPath to system/local server.conf
    - splunk-base : Add nodes to license master
    - TODO: Copy 3rd Party certs for Splunk Web
    - TODO: Configure LDAP for Splunk (optional)

### Files included in this role:

    splunk-base/
    ├── README.md
    ├── defaults
    │   └── main.yml
    ├── files
    │   ├── disable-transparent-hugepages.init
    │   └── splunk-7.1.0-2e75b3406c5b-linux-2.6-amd64.deb
    ├── handlers
    │   └── main.yml
    ├── tasks
    │   ├── certs.yml
    │   ├── config.yml
    │   ├── deploy.yml
    │   ├── kernel.yml
    │   ├── main.yml
    │   ├── mounts.yml
    │   └── service.yml
    └── templates
        ├── splunk.init.j2
        ├── system_local.alert_actions.j2
        ├── system_local.authentication.j2
        └── system_local.web.j2

## How does this work?

Once **splunk-base** has been applied, nodes can be configured for individual capabilities based on their inventory group.  
The inventory is based on the **hosts.example** file located within the playbook.  
However, inventory can be managed in other ways.

In the **defaults/main.yml**, configure the settings to utilize different options and configurations.  
Options can be overridden in higher level configs or a secrets file.

Define the base directory where splunk is installed.
```yaml
splunk_base: '/opt/splunk'
```

Use wget to grab the latest package from splunk.com and place the file under -> ***splunk-base/files*** folder.
```yaml
splunk_package_file: 'splunk-7.1.0-2e75b3406c5b-linux-2.6-amd64.deb'
splunk_version: '7.1.0'
```

Default disk info (will apply unless overridden by a group_var config like **splunk-staging-idx**).
```yaml
# Default disk info (will apply unless overridden by a group_var config)
disk_mounts:
  - device: "/dev/xvdf"
    type: "default"
    fs_type: "xfs"
    mount_path: "/storage"
```

Enable splunk web by default and set port to use.
```yaml
# Enable splunk web by default and set port to use
use_splunk_web: 1
splunk_httpport: 8443
```

Set the "stage" for deployment. Use this setting to differentiate between environments like **staging**, **development** or **production**.
Create a group var file **group_vars/splunk-staging** and set the override there to apply settings to entire inventory groups. 
```yaml
# Set in the appropriate group_vars file
deploy_stage: 'default_stage'
```

These settings are overridden by default as part of the overall splunk deployment.  
The variable overrides are located in the group_vars/secrets.yml file.

If using the splunk-ca role the ssl path setting should only need the "myOrg" change to the appropriate value.
```yaml
# Set under the appropriate environment variable in the secrets file
default_stage:
  splunk_mailserver: "mail.server.tld"
  splunk_pdf_header_left: "none"
  splunk_pdf_header_right: "none"
  splunk_auth_password: ""
  splunk_auth_username: ""
  splunk_use_tls: '0'
  splunk_use_ssl: '0'
  splunk_maxresults: '100'
  ssl:
    splunk_cluster_sslPassword: 'password'
    splunk_clientCert: "$SPLUNK_HOME/etc/auth/myOrg/XXXHOSTCERTXXX"
    splunk_sslRootCAPath: "/opt/splunk/etc/auth/myOrg/ca-cert.pem"
    splunk_serverCert: "$SPLUNK_HOME/etc/auth/myOrg/XXXHOSTCERTXXX"
    splunkweb_privKeyPath: "etc/auth/myOrg/XXXHOSTCERTXXX"
    splunkweb_serverCert: "etc/auth/myOrg/XXXHOSTCERTXXX"
    splunkforwarder_clientCert: "$SPLUNK_HOME/etc/auth/myOrg/XXXHOSTCERTXXX"
    splunkforwarder_sslRootCAPath: "/opt/splunkforwarder/etc/auth/myOrg/ca-cert.pem"
```