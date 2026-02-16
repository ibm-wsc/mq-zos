# Implementing basic authentication using SAF for the MQ Web Console

#### **Audience level**
Some knowledge of MQ, z/OS, Liberty

#### **Skillset**
MQ Administration, z/OS systems programming

### **Overview of exercise**

I. Generate tokens

II. Add and connect keyrings

III. Edit the mqwebuser.xml

### **Exercise** 

#### I.	Generate certificates

1\. Create a certificate authority (CA) certificate, which will be used to sign the server certificate. For example, enter the following RACF command:

```
RACDCERT GENCERT -
  CERTAUTH -
  SUBJECTSDN(CN('MQ CA') -
    O('IBM') -
    OU('MQ')) -
  SIZE(2048) -
  WITHLABEL('mqwebCertauth')
  ```

2\. Create a server certificate, signed with the CA certificate created in step 1, by entering the following command:

```
RACDCERT ID(SYSPROG) GENCERT -
  SUBJECTSDN(CN('MQW942') -
   O('IBM') -
   OU('MQ')) -
   SIZE(2048) -
   SIGNWITH (CERTAUTH LABEL('SYSPROG')) -
  WITHLABEL('mqwebServerCert')
  ```

  where mqwebUserId is the mqweb server started task user ID, and hostname is the host name of the mqweb server.

#### II. Add and connect keyrings

1\. Create a key ring using the following command:

```
RACDCERT ID(SYSPROG) ADDRING(MQW942.KeyRing) 
```

2\. Connect the CA certificate and server certificate to a SAF key ring by entering the following commands:

```
RACDCERT ID(SYSPROG) CONNECT(RING(MQW942.KeyRing) LABEL('mqwebCertauth') CERTAUTH)
RACDCERT ID(SYSPROG) CONNECT(RING(MQW942.KeyRing) LABEL('mqwebServerCert'))
```

3\. Validate everything looks good with the following command:

> racdcert id(SYSPROG) listring(MQW942.KeyRing)

![key ring](assets/safkeyring/saf1.png)

4\. FTP the exported CA certificate in binary to your workstation, and import it into your browser as a certificate authority certificate.

```
sftp user1@zos
cd //’USER1’
ls /+mode=text
mget USER1.MQ.CERT.MQWEBCA
keytool –import –v –trustcacerts –alias “MQ CA” –file USER1.MQ.CERT.MQWEBCA –keystore USER1.jks
```
    
#### Edit the mqwebuser.xml

1\. Edit the file WLP_user_directory/servers/mqweb/mqwebuser.xml, where WLP_user_directory is the directory that was specified when the crtmqweb script ran to create the mqweb server definition.

2\. Make the following changes to configure the mqweb server to use a RACF key ring:

<sslDefault sslRef="mqDefaultSSLConfig"/>

3\. Add to the mqwebuser.xml the following lines:

```
<keyStore id="defaultKeyStore" filebased="false"
          location="safkeyring://mqwebUserId/keyring"
          password="password" readOnly="true" type="JCERACFKS" />
<ssl id="thisSSLConfig" keyStoreRef="defaultKeyStore" sslProtocol="TLSv1.2"
          serverKeyAlias="mqwebServerCert" clientAuthenticationSupported="true" />
<sslDefault sslRef="thisSSLConfig"/>
```

where:
mqwebUserId is the mqweb server started task user ID.
keyring is the name of the RACF key ring.
mqwebServerCert is the label of the mqweb server certificate.

4\. Restart the mqweb server by stopping and restarting the mqweb server started task.




