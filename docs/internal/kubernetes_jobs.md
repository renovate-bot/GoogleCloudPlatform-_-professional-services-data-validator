# Scaling Data Validation with Kubernetes Jobs 

## Nature of Data Validation
Data Validation by nature is a batch process. We are presented with a set of arguments, the validation is performed, the results are provided and Data Validation completes. Data Validation can also take time (multiple secs, minutes) if a large amount of data needs to be validated. 

Data Validation has the `generate-table-partitions` function that partitions a row validation into a specified number of smaller, equally sized validations. Using this feature, validation of large tables can be split into row validations of many partitions of the same tables. See [partition table PRD](partition_table_prd.md) for details on partitioning. This process generates a sequence of yaml files which can be used to validate the individual partitions. 

## Kubernetes Workloads
Kubernetes supports different types of workloads including a few batch workload types. The Job workload is a batch workload that retries execution until a specified number of them successfully complete. If a row validation has been split into `n` partitions, then we need to validate each partition and merge the results of the validation. Using Kubernetes Jobs we need to successfully run `n` completions of the job, as long as we guarantee that each completion is associated with a different partition. Since release 1.21, Kubernetes provides a type of job management called indexed completions that supports the Data Validation use case. A Kubernetes job can use multiple parallel worker processes. Each worker process has an index number that the control plane sets which identifies which part of the overall task (i.e. which partition) to work on. Cloud Run uses slightly different terminology for a similar model - referring to each job completion as a task and all the tasks together as the job. The index is available in the environment variable `JOB_COMPLETION_INDEX` (in cloud run the environment variable is `CLOUD_RUN_TASK_INDEX`). An explanation of this is provided in [Introducing Indexed Jobs](https://kubernetes.io/blog/2021/04/19/introducing-indexed-jobs/#:~:text=Indexed%20%3A%20the%20Job%20is%20considered,and%20the%20JOB_COMPLETION_INDEX%20environment%20variable).

Indexed completion mode supports partitioned yaml files generated by the `generate-table-partitions` command if each worker process runs only the yaml file corresponding to its index. The`--kube-completions` or `-kc` flag when running configs from a directory indicates that the validation is running in indexed jobs mode on Kubernetes or Cloud Run. This way, each container only processes the specific validation yaml file corresponding to its index. 
### Passing database connection parameters
Usually, DVT stores database connection parameters in the `$HOME/.config/google-pso-data-validator` directory. If the environment variable `PSO_DV_CONFIG_HOME` is set, the database connection information can be fetched from this location. When running DVT in a container, there are two options regarding where to locate the database connection parameters.
1. Build the container image with the database connection parameters. This is not recommended because the container image is specific to the databases being used OR
2. Build a DVT container image as specified in the [docker samples directory](../../samples/docker/README.md). When running the container, pass the `PSO_DV_CONFIG_HOME` environment variable to point to a GCS location containing the connection configuration. This is the suggested approach.

## Future Work
### Clean up the way DVT works so that config files can be stored either in the file system or GCS.
In some cases, the command `configs run` can take a GCS location where the yaml files can be found. In other cases, this has to be specified by setting the `PSO_DV_CONFIG_HOME` environment variable to point to the GCS location. This inconsistent behavior is challenging and should be fixed.   
### Use GCP secret manager to store database connection configuration
DVT currently uses the GCP secret manager to secure each element of the database connection configuration. The current implementation stores the name of the secret for each element (host, port, user, password etc) in the database configuration file. When the connection needs to be made, the values are retrieved from the secret manager and used to connect to the database. While this is secure, database configurations are still stored in the filesystem.

Another way to use the secret manager is to store the complete connection configuration as the value of the secret. Then while specifying the connection, we can also indicate to DVT to look for the connection configuration in the secret manager rather than the file system. Since most DVT commands connect to one or more databases, nearly every DVT command can take optional arguments that tells DVT to look for the connection configuration in the secret manager. 