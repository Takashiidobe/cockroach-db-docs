A successful locality-aware backup job requires that each node in the cluster has access to each storage location. This is because any node in the cluster can claim the job and become the [_coordinator_ ](backup-architecture.html#job-creation-phase) node.