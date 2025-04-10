# Using SSL with MQ

Exchanging digital certificates between MQ and MQExplorer requires the use of a trusted certificate authority (CA). The certificate authority’s role is to sign (or create) a personal certificate that can be used by an application to identify itself to another application and verify the personal certificate presented at connection time can be trusted. The CA also provides a public CA digital certificate that can be used by application to validate personal certificate issued by that CA.
RACF commands will be used to create a CA signer certificate (known as a CERTAUTH certificate in RACF) that will be used later  sign client certificates used by clients and to create a personal certificate for the MQ channel initiator (CHIN) address space.

The first step to enable digital certificate exchanges between MQExplorer and the queue manager is to create a Certificate Authority (CA) certificate (also known as signer certificates), which will be used to create or sign personal certificates. When a personal certificate is presented to RACF, this CA certificate will be used to ensure the validity of the personal certificate.

1. Use the TSO command RACDCERT CERTAUTH GENCERT command to create a signer
certificate with label 'MQ CA'.

    ```
    racdcert certauth gencert subjectsdn(CN('MQ CA') OU('ATS')
    O('IBM') C('US')) withlabel('MQ CA') keyusage(certsign)
    notafter(date(2025/3/31)) 
    ```

    | Parameter | Meaning |
    | ------- | -------- |
    | subjectsdn(CN('common-name') OU('organizational-unit-name1') O('organization-name') C('country')) | |
    | withlabel | Specifies the label assigned to this certificate. If specified, this must be unique to the user ID with which the certificate is associated. If not specified, it defaults in the same manner as the WITHLABEL keyword on the RACDCERT ADD command. |
    | signwith | Specifies the certificate with a private key that is signing the certificate. If SIGNWITH is specified, it must refer to a certificate that has a private key associated with it. If no private key is associated with the certificate, an informational message is issued and processing stops. | 
    | notafter | Specifies the local date and time after which the certificate is no longer valid. |

2. Use the RACDCERT CERTAUTH GENCERT command to create a personal certificate for the
MQ channel initiator (CHIN) with a label of 'MQCHIN'.

    ```
    racdcert id(dquincy) gencert subjectsdn(CN('MQ CHIN') OU('ATS')
    O('IBM') C('US')) withlabel('MQ CHIN') signwith(certauth
    label('MQ CA')) notafter(date(2025/3/31) 
    ```

| Parameter | Meaning |
| ------- | -------- |
| subjectsdn | |
| withlabel | Specifies the label assigned to this certificate. If specified, this must be unique to the user ID with which the certificate is associated. If not specified, it defaults in the same manner as the WITHLABEL keyword on the RACDCERT ADD command. |
| signwith | Specifies the certificate with a private key that is signing the certificate. If SIGNWITH is specified, it must refer to a certificate that has a private key associated with it. If no private key is associated with the certificate, an informational message is issued and processing stops. | 
| notafter | Specifies the local date and time after which the certificate is no longer valid. |

> ***Tech-Tip:** START1 is the MQ channel initiator region’s RACF identity under which the CHIN started task is executing.*

3. Run the command 

        racdcert certauth list(label('MQ CA'))

    Your output should look like this:

    ```                                                        
    Digital certificate information for CERTAUTH:           
                                                            
    Label: MQ CA                                          
    Certificate ID: 2QiJmZmDhZmjgdTYQMPB                  
    Status: TRUST                                         
    Start Date: 2025/03/10 00:00:00                       
    End Date:   2025/03/31 23:59:59                       
    Serial Number:                                        
        >00<                                             
    Issuer's Name:                                        
        >CN=MQ CA.OU=ATS.O=IBM.C=US<                     
    Subject's Name:                                       
        >CN=MQ CA.OU=ATS.O=IBM.C=US<                     
    Signing Algorithm: sha256RSA                          
    Key Usage: CERTSIGN                                   
    Key Type: RSA                                         
    Key Size: 2048                                        
    Private Key: YES                                      
    ```

> ***Tech-Tip:** The common name (CN), organization unite(OU), organization(O) country (C) , labels and aliases are case sensitive. Subsequent RACF or keytool commands referencing a label, an alias or common name, etc. in any command must use the exact same case and spacing when referring to the values of these fields in the certificate (i.e. be consistent).*

*For example, command racdcert certauth list(label(‘MQ CA’)) would display this certificate while command racdcert certauth list(label(‘mq CA’)) would not. Command racdcert certauth delete(label(‘MQ CA’)) could be used to delete this certificate while command racdcert certauth delete(label(‘mq CA’)) could not.*

4. Use the RACDCERT ADDRING command to create a RACF Key Ring for the MQ CHIN region’s RACF identity. The signer certificates and WMQ’s personal certificates will be connected to this key ring.

        racdcert id(user1) addring(MQCHIN.KeyRing) 

> ***Tech-Tip:** Tech-Tip: RACF key rings are case sensitive so be sure to enter the key ring name, e.g. MQ.KeyRing in exactly the same case in subsequent commands.*

> ***Tech-Tip:** START1 needs READ access to FACILITY resource IRR.DIGTCERT.LISTRING in order to access this key ring.*

5. Use the RACDCERT CONNECT command to connect the MQ signer certificate that was used to sign the ‘MQ CA’ client’s personal certificate to the MQ CHIN’s key ring.

        racdcert id(start1) connect(ring(MQCHIN.KeyRing) label('MQ CA') certauth usage(certauth)) 


> ***Tech-Tip:** A RACF key ring is unique to the owning user. A key ring labeled “MQ.KeyRing” owned by user START1 is not the same key ring owned by user START2. So connecting a signer certificate labeled “MQ CA” with command RACDCERT ID(START1) CONNECT(RING(MQCHIN.KeyRing) has no effect on a key ring labeled MQCHIN.KeyRing owned by user START2. The significance of this is more apparent during key ring maintenance when a key ring is updated with new information but the application is actually using another key ring with the same name but owned by a different user.*