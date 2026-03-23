# IBM MQ for z/OS Queue Sharing Group Best Practices

### At a glance
This resource is meant to be used as customers are planning out their queue-sharing group environments. Queue-sharing involves connecting 2 or more queue managers on 2 seperate LPARs with coupling facility storage and Db2 data-sharing to achieve high availability.

### IBM MQ for z/OS considerations

- [ ] Are my applications on private queues well suited for queue-sharing?

What makes applications good candidates for queue-sharing?

    - [ ] Do my applications have loose or zero affinities? Check for strict message ordering or grouping as an indicator of tight affinities
    - [ ] Are the private queues they're connected processing messages efficiently/not letting the messages hang around? Check for message age.
