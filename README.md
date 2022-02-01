# Running TPC-C Test on CockroachDB

## Prerequisites 

### accessing the cockroach-client pod (optional)

The steps outlined here will only cover starting the interactive SQL pod to adjust the preserve_downgrade _option. Detailed steps for updating cockroachdb versions can be found [here](https://github.com/waynedovey/example-bank/tree/1623487141edb403c1c9b4b637f3445ca66ed973/database-init/cockroachdb/charts/cockroachdb-multicluster/charts/cockroachdb)

In the current cockroachDB environment that has been configured on aws-cluster-shared-1: 

* In our current environment, there is an SSL secure cockroach-client pod in the test-cockroachdb namespace.
* This pod can be accessed using: `oc rsh cockroach-client -n test-cockroachdb`

* Start the sql client: `./cockroach sql --insecure --host cockroachdb-sample-public`

* Set cluster.preserve_downgrade_option, where $current_version is the CockroachDB version currently running (e.g., 19.2):

``` 
SET CLUSTER SETTING cluster.preserve_downgrade_option = '$current_version';
```

* Exit the shell and delete the temporary pod using: `\q`

Alternately, you could launch a fresh interactive pod and start the built in sql client using: 
```
oc run cockroachdb --rm -it \
--image=cockroachdb/cockroach \
--restart=Never \
-- sql --insecure --host=my-release-cockroachdb-public
```
> If you are running in secure mode, you will have to provide a client certificate to the cluster in order to authenticate, so the above command will not work. See [here](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/client-secure.yaml) for an example of how to set up an interactive SQL shell against a secure cluster or [here](https://github.com/cockroachdb/cockroach/blob/master/cloud/kubernetes/example-app-secure.yaml) for an example application connecting to a secure cluster.

After you have entered the SQL session, run the following step: 
``` 
SET CLUSTER SETTING cluster.preserve_downgrade_option = '$current_version';
```

* Exit the shell and delete the temporary pod using: `\q`


### updating the max-disk-temp-storage to execute the TPC-C test 

*these steps must be performed on each cluster that is part of the cockroachDB deployment*

* When running the TPC-C test you may see an error similar to 
```
disk budget exceeded: 1048576 bytes requested, 0 currently allocated, 0 bytes in budget
```
* This error is due to the `max-disk-temp-storage` value being set to 0 by default 

* If you deployed using the Helm chart
    * Change the `max-disk-temp-storage` value from 0 to 256MiB in   
    ```
    /cockroachdb/charts/cockroachdb-multicluster/charts/cockroachdb/values.yaml
    ```
    * Redeploy the Helm chart on each cluster that is part of the cockroachDB deployment 

* If you deployed CockroachDB manually:
    * You will need to edit the StatefulSet of the cockroachdb deployment. 
    
    ``` - name: db
          image: 'cockroachdb/cockroach:v21.1.7'
          args:
            - shell
            - '-ecx'
            - >-
              source /topology-conf/topology.conf && exec /cockroach/cockroach
              start
            
            <omitted for length>

              --http-port=8080 
              --port=26257 
              --cache=8GiB
              --max-disk-temp-storage=256MiB
              --max-offset=500ms
              --max-sql-memory=8GiB 
              --locality=region=${region},zone=${zone}
    ```
    * This could be done by:  
         * Editing the YAML directly and applying it
         * Editing the stateful set inside the OpenShift CLI 
         * Using an `oc patch` command
    * Choose whichever is most suitable for your deployment. 
