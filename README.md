# HDP cluster registration in DPS with Knox HA and custom JWT cookie Name

### Description : 
Knox SSO uses private key from gateway.jks to sign JWT tokens and issue to clients. These JWT token will be used by client under cookie name set with config property knoxsso.cookie.name in SSO topology. 

As a part of setup, service for which SSO is enabled will be configured with PEM cert of Knox server, Which is plublic key pair of private key used for signing tokens. With this PEM cert, services will be able to validate the jwt token forwarded by clients after successful knox sso authentication.

In case of Knox HA configuration, where each of the knox instances has their own gateawy.jks will not work if service is configured to us Knox LB URL. In this case JWT can be issued by any of the knox instance choosed by LB routing algorithm, and the service that is configured for SSO can only be configured with one PEM cert which might not be of the knox instance that signed JWT token. 

#### We can use below  3 options to make SSO auth work with Knox HA/LB: 

1) Use the same Private key to create/request public cert for both knox instance. This way we are create key pair with one private key but multiple public keys. 
   Once we have private and pulic key, recreate gateway.jks with correct public key for each knox host. With this approach JWT will be signed by same private key that can be validated by any of the knox PEM certs. So the services configured for knox sso can be configured with any of the knox PEM cert.

2) Create a gateway.jks with common private/public key with SAN entries that includes FQDN of both the knox instances. 
   #### Example keytool command : 
     ```
    #keytool -genkey -dname "CN=hive.squadron.support.hortonworks.com, OU=Support, O=HWX, L=Durham, ST=North Carolina, C=US" -keysize 2048 -alias gateway-identity -validity 3650 -keyalg RSA -keystore gateaway.jks -storepass hadoop  -ext SAN=dns:c416-node3.squadron.support.hortonworks.com,dns:c416-node4.squadron.support.hortonworks.com,dns:c416-node2.squadron.support.hortonworks.com
    ```
    
3) Use a different jks file to sign JWT tokens, with this option  Knox SSO no longer depends on gateway.jks for signing jwt tokens. And services configured for SSO should be configured with PEM cert from the new jks file configured for signing. 


In this doc we will configured knox to use different JKS file for jwt token singing. Note that in all three cases jks file created must be copied/replaced under the path `/usr/hdp/current/knox-server/data/security/keystores`

### Configuring Knox SSO to use dedicated keystore for JWT signing : 

Below steps are performed assuming that `knoxsso.xml` is already configured to use custom cookie name other than default `hadoop-jwt` (using `knoxsso.cookie.name` defined in sso topology)

Creating the JKS file may involve different steps based on environment requirement. If the environment needs  the certs to be CA signed, then please follow the process to request CA signed certs (as this jks is only used for singing, CN dont have to be FQDN hostnames)

In our case we would be creating a selfsigned cert uisng keytool : 

**Step 1** : Create jks keystore using keytool command (note that this is selfsigned cert and should not compromise security standards as this cert is only used for signing certs and not for https connections)
```
#keytool -genkey -dname "CN=KnoxSSO, OU=Support, O=HWX, L=Durham, ST=North Carolina, C=US" -keysize 2048 -alias knoxsso -validity 3650 -keyalg RSA -keystore /var/tmp/knoxSSO.jks -storepass <Password> 
```
(Password used here is same as Knox-Master-Secret and First name and Lastname is set as KnoxSSO, or it can be anything)

**Step 2** : Create passphrase alias, here the passphrase alias prompts for the password, which is same password set for knoxSSO.jks in step1

```
#/usr/hdp/current/knox-server/bin/knoxcli.sh create-alias signing.key.passphrase --cluster knoxsso 
```

**Step 3** : Extract the public key from the knox SSO.jks which is in the DER format.

```
#keytool -export -file /tmp/sign.crt -alias knoxsso -keystore knoxSSO.jks 
```

**Step 4**: Convert the DER to PEM format which can be used configure ambari server jwt-cert.pem

```
#openssl x509 -in /tmp/sign.crt -inform DER -outform PEM > /tmp/sign.pem
```


**Step 5**: Copy the knoxSSO.jks to keystore path of knox
```
#cp /var/tmp/knoxSSO.jks /usr/hdp/current/knox-server/data/security/keystores/
```

**Step 6**:  Copy the knoxSSO.jks to the second knox instance:
```
#scp knoxSSO.jks <knox2>:/usr/hdp/current/knox-server/data/security/keystores/
```

**Step 7** : On Knox 2 host create the passphrase alias(step was done on knox 1 already):
```
#/usr/hdp/current/knox-server/bin/knoxcli.sh create-alias signing.key.passphrase --cluster knoxsso
```


#### Configure gateway-site with below two properties:

`Ambari > Knox> Configs > Custom gateway-site > Add`

*gateway.signing.keystore.name=knoxSSO.jks
gateway.signing.key.alias=knoxsso*

On Ambari server, if already configured for knox SSO, replace the content of `jwt-cert.pem` with the new pem file content /tmp/sign.pem (created on knox 1 Instance)
```
#cp /etc/ambari-server/conf/jwt-cert.pem /etc/ambari-server/conf/jwt-cert.pem.Old
#> /etc/ambari-server/conf/jwt-cert.pem 
#vi /etc/ambari-server/conf/jwt-cert.pem   
```

(optionally if LB FQDN is not used while configuring SSO for the first time, then change the property url by executing 'ambari-server setup-sso' on Ambari server, which will prompt for disalbing SSO)

Restart Knox and Ambari server services and verify the SSO access, which must be redirected to knox LB URL instead of individual Knox hosts. 
```
#ambari-server restart
```

#### Configuring Knox Trusted proxy for Ambari Server : 

Knox Trusted Proxy(KTP) for ambari is supported from Ambari 2.7.3 version and would need changes on Knox ambari/ui service definitions and also  setting up spnego authentication for Ambari. 

Pre-requisite for KTP is to configure SPNEGO/kerberos authentication for Ambari. So kerberos must be enabled on cluster and Ambari server has the spnego.service.keytab available. 


**Step 1**: Enable kerberos/spnego authentication for Ambari: 

``` #ambari-server setup-kerberos
Using python  /usr/bin/python
Setting up Kerberos authentication
Enable Kerberos authentication [true|false] (false): true
SPNEGO principal (HTTP/_HOST):
SPNEGO keytab file (/etc/security/keytabs/spnego.service.keytab):
Auth-to-local rules (DEFAULT):
Properties to be updated / written into ambari properties:
{'authentication.kerberos.auth_to_local.rules': 'DEFAULT',
 'authentication.kerberos.enabled': 'true',
 'authentication.kerberos.spnego.keytab.file': '/etc/security/keytabs/spnego.service.keytab',
 'authentication.kerberos.spnego.principal': 'HTTP/_HOST'}
Save settings [y/n] (y)? y
Kerberos authentication settings successfully saved. Please restart the server in order for the new settings to take effect.
Ambari Server 'setup-kerberos' completed successfully
```


**Step 2**: With the first completed, we can now enable trusted proxy on Ambari using below command: 

```
#ambari-server setup-trusted-proxy
Using python  /usr/bin/python
Setting up Trusted Proxy support...
Enter Ambari Admin login: admin
Enter Ambari Admin password:
Do you want to configure Trusted Proxy Support [y/n] (y)? y
The proxy user's (local) username? knox
Allowed hosts for knox (*)?
Allowed users for knox (*)?
Allowed groups for knox (*)?
Add another proxy user [y/n] (n)?
Ambari Server 'setup-trusted-proxy' completed successfully.
```
Note that "The proxy user's (local) username" is set to knox, as knox will be impersonating user accessing Ambari via Knox proxy, 

**Step 3**: Restart Ambari server :
```
# service ambari-server restart
```

#### Configuring Knox Service definition to enabled KTP for ambari service: 

With the above configuration steps, Ambari server is now configured to accept API request coming from knox with doAs=<userName>. For knox to start acting as trusted proxy, ambari service definition should be modified, so that Anonymous authentication policy is disabled. 

**Step 1** : Backup knox ambari service.xml files 

```
# cp /usr/hdp/current/knox-server/data/services/ambari/2.2.0/service.xml /var/tmp/service_ambari.xml
# cp /usr/hdp/current/knox-server/data/services/ambariui/2.2.0/service.xml /var/tmp/service_ambariui.xml
```

**Step 2** : Comment the policies and section and dispatch class name : 

Eg: 
```
# vi /usr/hdp/current/knox-server/data/services/ambariui/2.2.0/service.xml  

<service role="AMBARI" name="ambari" version="2.2.0">

<!--  Start of KTP enable 1
  <policies>
        <policy role="webappsec"/>
        <policy role="authentication" name="Anonymous"/>
        <policy role="rewrite"/>
        <policy role="authorization"/>
    </policies>
End of KTP enable 1 -->

    <routes>
        <route path="/ambari/api/v1/**">
            <rewrite apply="AMBARI/ambari/api/outbound" to="response.body"/>
            <rewrite apply="AMBARI/ambari/api/inbound" to="request.body"/>
        </route>
        <route path="/ambari/api/v1/persist/*?*">
            <rewrite apply="AMBARI/ambari/api/inbound" to="request.body"/>
        </route>
    </routes>
<!--  Start of KTP enable 2 <dispatch classname="org.apache.knox.gateway.dispatch.PassAllHeadersNoEncodingDispatch"/>  End of KTP enable 2 -->
</service>
```

(Update the file /usr/hdp/current/knox-server/data/services/ambari/2.2.0/service.xml same comments as above)

**Step 3**: Modification of service definitaion will only be used when topolgies are redepolyed. A modification on file timestamp on topology xml file will trigger the new depolyment. Use linux command 'touch' to redeploy all the topology)

```
# find /etc/knox/conf/topologies/ -name "*.xml" -exec touch {} \;
# ls -l /etc/knox/conf/topologies/*.xml
```

#### Adding cluster to DPS : 

Follow the same steps as mentioned in docs to configured Knox topologies on both the knox hosts: 

https://docs.cloudera.com/HDPDocuments/DP/DP-1.2.2/installation/content/dp_configure_sso_with_hdp_clusters.html
https://docs.cloudera.com/HDPDocuments/DP/DP-1.2.2/installation/content/dp_configure_proxying_for_wire_encryption.html

Add cluster and select the option to to access ambari behine knox: 

https://docs.cloudera.com/HDPDocuments/DP/DP-1.2.2/administration/content/dp_register_a_cluster_in_dp_platform.html

Provide the URL `https://<LB-FQDN>:<port>/gateway/dp-proxy/ambari`
