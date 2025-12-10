How do you upgrade your queue managers on z/OS once a new version of the product is installed?

Assuming the SMP/E process has gone correctly, you will be able to access a HOLDDATA file. This is essentially a readme with the appropriate instructions for any changes that need to be made.
 
If you have a queue-sharing group, you will need to customize and run JCL for the following samples in the queue manager’s SCSQPROC: 
 
CSQ45BPK – update the binds for DB2
CSQ45GEX – update the grants for DB2 
Once you have successfully run the JCL, you can go ahead and make updates to the SYS1.PROCLIB. 
 

Here, you will adjust the MSTR and the CHIN for your target queue manager. You will need to ensure that the appropriate libraries are pointed at the version of MQ you’d like to use. In our case, we had to change our high-level qualifiers from MQ933CD to MQ935CD since we’d like to run version IBM MQ 9.3.5.

If you are upgrading a private queue manager running on z/OS, you can bypass the bind and grant steps. All that needs to be done is updating your MSTR and CHIN libraries in SYS1.PROCLIB. 

EARLY CODE UPDATE
/34Y
SETPROG LPA,ADD,MODNAME=(CSQ3INI,CSQ3EPX),DSNAME=thlqual.SCSQLINK
SETPROG LPA,ADD,MODNAME=(CSQ3ECMX),DSNAME=thlqual.SCSQSNLx
 REFRESH QMGR TYPE(EARLY)
