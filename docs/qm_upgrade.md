# Customizing a new queue manager on IBM MQ for z/OS
### Audience level
Some knowledge of MQ or z/OS 
### Skillset
z/OS Systems Programming, MQ Administration

### Background
This lab walks users through how to upgrade queue managers on z/OS. This lab assumes the SMP/E work has already been done to install the product. In this lab, we will use MQ v.9.4.2 as an example. As such, our high level qualifier is MQ942CD.

### Overview of exercise
What needs to be done here: 
I. Stop queue manager
II. OPTIONAL: Re-bind queue sharing group 
III. Adjust MSTR and CHIN started tasks
III. Update early code
IV. Restart queue manager

### Lab begin
#### I. Stop queue manager

1\. From the ISPF main menu, type 'SDSF' and press Enter. System Display and Search Facility (SDSF) is a utility that allows you to monitor, control, and view the output of jobs in the system.

2\. On the SDSF menu, type '/' and press Enter to access the MVS command line via the System Command Extension.

3\. On the command line, type 'ZQS1 STOP CHINIT'

4\. Next, type 'ZQS1 STOP QMGR'

#### II. OPTIONAL: Re-bind queue sharing group



