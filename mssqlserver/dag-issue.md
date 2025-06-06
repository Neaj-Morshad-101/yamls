6. CAP Theorem
In the presence of a network partition, a distributed system must choose between Consistency (all nodes see the same data) and Availability (every request receives a response) 
interviewing.io
.

Systems trade off along the CA, CP, or AP axes based on use-case requirements.




I have set up a SQL Server Distributed Availability Group (DAG) in Kubernetes using SQL Server on Ubuntu images. The setup consists of two availability groups (AGs) across two separate clusters:

Setup Details:
Primary Cluster (AG1)
Pods: ag1-0 (Primary), ag1-1, ag1-2. The Primary is Exposed via the LoadBalancer service. Remote Cluster (AG2):
Pods: ag2-0 (The Primary of AG2, Acting as a forwarder of DAG), ag2-1, ag2-2. The Forwarder (ag2-0) is Exposed via the LoadBalancer service.

Distributed AG Configuration:
AG1 and AG2 are part of the DAG. Each AG’s primary is dynamically selected using the pod's label role=primary. LISTENER_URL in the DAG configuration points to the LoadBalancer service of each AG.
Issue: DAG Not Syncing After AG1 Failover

For testing, I triggered a failover in AG1 using:
ALTER AVAILABILITY GROUP [AG1] FORCE_FAILOVER_ALLOW_DATA_LOSS;
The global primary changed from ag1-0 to ag1-1, and I updated the role=primary label accordingly (removed from ag1-0, added to ag1-1. However, AG2 (the forwarder and its replicas) stopped syncing and became unhealthy.

From ag2-0 (forwarder) logs, I only see connection timeouts and disconnections from the global primary. AG2 is not automatically reconnecting to the new primary (ag1-1), even though the LoadBalancer service in LISTENER_URL now points to ag1-1.

Logs from ag2-0 (Forwarder) Shows Like
A connection timeout has occurred while attempting to establish a connection to GLOBAL PRIMARY. Either a networking or firewall issue exists, or the endpoint address provided for the replica is not the database mirroring endpoint of the host server instance
Steps I Tried:
Checked DAG Configuration – The LISTENER_URL is correctly set to the LoadBalancer of AG1, which now points to ag1-1.
Ran the Resume Command: ALTER DATABASE [agtestdb] SET HADR RESUME; This did not resolve the issue.
Verified Network Connectivity

Questions:
1️⃣ What steps are required to ensure AG2 correctly syncs with the new global primary (ag1-1) after an AG1's internal failover?
2️⃣ Is there a specific command that needs to be run on the forwarder (ag2-0) or the new global primary (ag1-1) to reestablish synchronization?
3️⃣ Why isn’t AG2 automatically reconnecting, even though the LoadBalancer service points to the correct primary?
4️⃣ Are there any best practices for handling SQL Server DAG failovers in Kubernetes?

Any insights would be greatly appreciated! 🚀

sql-serverkubernetesfailoveravailability-group










Title: SQL Server Distributed Availability Group (DAG) Not Syncing After AG1 Failover in Kubernetes

I have set up a SQL Server Distributed Availability Group (DAG) in Kubernetes using SQL Server on Ubuntu images. The setup consists of two availability groups (AGs) across two separate clusters:

Setup Details:
Primary Cluster (AG1)
Pods: ag1-0 (Primary), ag1-1, ag1-2
Exposed via LoadBalancer service
Remote Cluster (AG2)
Pods: ag2-0 (Primary for AG2, also acting as a forwarder), ag2-1, ag2-2
Exposed via LoadBalancer service

DAG Configuration:
AG1 and AG2 are part of the DAG.
Each AG’s primary is dynamically selected using the label role=primary.
LISTENER_URL in the DAG configuration points to the LoadBalancer service of each AG.
Issue: DAG Not Syncing After AG1 Failover
For testing, I triggered a failover in AG1 using:


ALTER AVAILABILITY GROUP [AG1] FORCE_FAILOVER_ALLOW_DATA_LOSS; 
The global primary changed from ag1-0 to ag1-1, and I updated the role=primary label accordingly. However, AG2 (the forwarder and its replicas) stopped syncing and became unhealthy.

From ag2-0 (forwarder) logs, I only see connection timeouts and disconnections from the global primary. AG2 is not automatically reconnecting to the new primary (ag1-1), even though the LoadBalancer service in LISTENER_URL now points to ag1-1.

Logs from ag2-0 Forwarder
Error: Connection timeout to primary node ag1-1. Unable to establish a connection.
Error: The remote copy of database 'agtestdb' is not in a recoverable state.
Steps I Tried
✅ Checked DAG Configuration – The LISTENER_URL is correctly set to the LoadBalancer of AG1, which now points to ag1-1.
✅ Ran the Resume Command:

ALTER DATABASE [agtestdb] SET HADR RESUME;
This did not resolve the issue.
✅ Verified Network Connectivity – telnet ag1-1 5022 works from AG2.

Additional Workarounds Tried:

Restarted the ag2-0 forwarder pod.
Ran:
sql
Copy
Edit
ALTER AVAILABILITY GROUP [DAGTest] MODIFY REPLICA ON 'ag1-1' WITH (ENDPOINT_URL = 'tcp://new_primary:5022');
But AG2 still does not sync.
Questions
1️⃣ What steps are required to ensure AG2 correctly syncs with the new global primary (ag1-1) after an AG1 failover?
2️⃣ Is there a specific command that needs to be run on the forwarder (ag2-0) or the new global primary (ag1-1) to reestablish synchronization?
3️⃣ Why isn’t AG2 automatically reconnecting, even though the LoadBalancer service points to the correct primary?
4️⃣ Are there any best practices for handling SQL Server DAG failovers in Kubernetes?

Any insights would be greatly appreciated! 🚀




🔍 Debugging and Fixing the Issue
Step 1: Check the Distributed AG Status on AG2
Run the following query on ag2-0 (forwarder):

sql
Copy
Edit
SELECT ag.name, drs.synchronization_state_desc, drs.is_local, drs.is_primary_replica
FROM sys.dm_hadr_availability_replica_states drs
JOIN sys.availability_groups ag
ON drs.group_id = ag.group_id;
This will show if AG2 thinks it is still connected to the old AG1 primary (ag1-0).
If synchronization_state_desc is NOT SYNCHRONIZED, AG2 has lost connection.
Step 2: Manually Force AG2 to Recognize the New Primary
Try forcing the DAG to recognize ag1-1 as the new primary:

sql
Copy
Edit
ALTER AVAILABILITY GROUP [DAGTest] FAILOVER;
This may allow AG2 to reset and reconnect to AG1.

Step 3: Restart the HADR Transport on AG2
On ag2-0, restart HADR transport to clear any stale connections:

sql
Copy
Edit
ALTER AVAILABILITY GROUP [AG2] OFFLINE;
ALTER AVAILABILITY GROUP [AG2] ONLINE;
Or, restart the SQL Server service in AG2:

bash
Copy
Edit
systemctl restart mssql-server
This forces AG2 to reestablish connections.

Step 4: Ensure AG2 Can See the Correct AG1 Endpoint
Verify that AG2 knows the correct primary endpoint:

sql
Copy
Edit
SELECT * FROM sys.availability_replicas WHERE group_id = (SELECT group_id FROM sys.availability_groups WHERE name = 'DAG');
If the old AG1 primary (ag1-0) is still listed, update it:

sql
Copy
Edit
ALTER AVAILABILITY GROUP [DAGTest] MODIFY REPLICA ON 'ag1-1' WITH (ENDPOINT_URL = 'tcp://ag1-1:5022');
Step 5: Resume Data Movement
If AG2 is disconnected but still part of the DAG, resume data movement:

sql
Copy
Edit
ALTER DATABASE [agtestdb] SET HADR RESUME;
If the issue persists, try removing and re-adding AG2 as a forwarder:

sql
Copy
Edit
ALTER AVAILABILITY GROUP [DAGTest] REMOVE REPLICA ON 'ag2-0';
GO
ALTER AVAILABILITY GROUP [DAGTest] ADD REPLICA ON 'ag2-0'
WITH (ENDPOINT_URL = 'tcp://ag2-0:5022');

🔬 Why Didn’t AG2 Automatically Sync?
The automatic synchronization failure happens due to:
1️⃣ AG2's forwarder still pointing to the old AG1 primary (ag1-0).
2️⃣ AG2 not detecting the new primary (ag1-1) due to stale metadata.
3️⃣ SQL Server's HADR transport service failing to reconnect after the failover.
4️⃣ Potential Kubernetes networking delays (if AG2 reached ag1-0 faster before failover).

🛠️ Best Practices to Prevent This in the Future
Use automatic failover policies with a shorter timeout to force AG2 to detect the change faster.
Ensure AG1's LoadBalancer service refreshes DNS entries immediately after failover.
Regularly monitor sys.dm_hadr_availability_replica_states to catch sync failures early.


How SQL Server DAG Synchronization Works Internally
When a Distributed Availability Group (DAG) is set up, SQL Server manages synchronization using the HADR (High Availability and Disaster Recovery) process. The forwarder (AG2) acts as an intermediary between the global primary (AG1) and its own replicas.

What Happens During a DAG Failover?
AG1 Primary Role Switches

ag1-1 becomes the new primary after failover.
The endpoint URL remains unchanged because it points to the LoadBalancer service.
AG2 (Forwarder) Should Reconnect

AG2 (ag2-0) should detect the new AG1 primary (ag1-1) automatically.
If a disconnection occurs, SQL Server internally attempts to reconnect to the new primary.
It relies on HADR transport services to reestablish the connection.
Failure to Reconnect (Your Issue)

If AG2 stops syncing, it means the HADR transport layer is not detecting the new AG1 primary.
This can happen due to endpoint mismatch, stale session state, or unresolved database status.
🔍 Debugging and Fixing the Issue
Step 1: Check the Distributed AG Status on AG2
Run the following query on ag2-0 (forwarder):

sql
￼Copy
￼Edit
SELECT ag.name, drs.synchronization_state_desc, drs.is_local, drs.is_primary_replica
FROM sys.dm_hadr_availability_replica_states drs
JOIN sys.availability_groups ag
ON drs.group_id = ag.group_id;
This will show if AG2 thinks it is still connected to the old AG1 primary (ag1-0).
If synchronization_state_desc is NOT SYNCHRONIZED, AG2 has lost connection.
Step 2: Manually Force AG2 to Recognize the New Primary
Try forcing the DAG to recognize ag1-1 as the new primary:

sql
￼Copy
￼Edit
ALTER AVAILABILITY GROUP [DAGTest] FAILOVER;
This may allow AG2 to reset and reconnect to AG1.

Step 3: Restart the HADR Transport on AG2
On ag2-0, restart HADR transport to clear any stale connections:

sql
￼Copy
￼Edit
ALTER AVAILABILITY GROUP [AG2] OFFLINE;
ALTER AVAILABILITY GROUP [AG2] ONLINE;
Or, restart the SQL Server service in AG2:

bash
￼Copy
￼Edit
systemctl restart mssql-server
This forces AG2 to reestablish connections.

Step 4: Ensure AG2 Can See the Correct AG1 Endpoint
Verify that AG2 knows the correct primary endpoint:

sql
￼Copy
￼Edit
SELECT * FROM sys.availability_replicas WHERE group_id = (SELECT group_id FROM sys.availability_groups WHERE name = 'DAGTest');
If the old AG1 primary (ag1-0) is still listed, update it:

sql
￼Copy
￼Edit
ALTER AVAILABILITY GROUP [DAGTest] MODIFY REPLICA ON 'ag1-1' WITH (ENDPOINT_URL = 'tcp://ag1-1:5022');
Step 5: Resume Data Movement
If AG2 is disconnected but still part of the DAG, resume data movement:

sql
￼Copy
￼Edit
ALTER DATABASE [agtestdb] SET HADR RESUME;
If the issue persists, try removing and re-adding AG2 as a forwarder:

sql
￼Copy
￼Edit
ALTER AVAILABILITY GROUP [DAGTest] REMOVE REPLICA ON 'ag2-0';
GO
ALTER AVAILABILITY GROUP [DAGTest] ADD REPLICA ON 'ag2-0'
WITH (ENDPOINT_URL = 'tcp://ag2-0:5022');
🔬 Why Didn’t AG2 Automatically Sync?
The automatic synchronization failure happens due to:
1️⃣ AG2's forwarder still pointing to the old AG1 primary (ag1-0).
2️⃣ AG2 not detecting the new primary (ag1-1) due to stale metadata.
3️⃣ SQL Server's HADR transport service failing to reconnect after the failover.
4️⃣ Potential Kubernetes networking delays (if AG2 reached ag1-0 faster before failover).

🛠️ Best Practices to Prevent This in the Future
Use automatic failover policies with a shorter timeout to force AG2 to detect the change faster.
Ensure AG1's LoadBalancer service refreshes DNS entries immediately after failover.
Regularly monitor sys.dm_hadr_availability_replica_states to catch sync failures early.
🚀 Summary
To fix the issue, try the following:
✅ Run sys.dm_hadr_availability_replica_states on ag2-0 to check sync status.
✅ Force failover: ALTER AVAILABILITY GROUP [DAGTest] FAILOVER;
✅ Restart HADR transport: ALTER AVAILABILITY GROUP [AG2] OFFLINE/ONLINE;
✅ Update AG1’s endpoint on AG2 manually.
✅ Resume data movement: ALTER DATABASE [agtestdb] SET HADR RESUME;

This should reestablish AG2’s connection to AG1 after failover. Let me know if you need further debugging! 🚀



Restart ag2-0 
Try failover to ag1-0 again, and see If ag2-0 can connect to ag1-0. 
Ping from ag2-0 to ag1-1 after failover. 
Ask questions on stackover flow. 







If I remove label role=primary from global primary pod (ag1-0), how the ag2-0 knows, and shows disconnection from global primary? 


I am doing a poc for DAG in SQL Server. I have configured a distributed availability group in k8s using sql server ubuntu images, 
In my primary cluster, I have a availability group setup like 3 pods: ag1-0 (primary, exposed by load balancer svc), ag1-1, ag1-2.
In my remote cluster, I have a availability group setup like 3 pods: ag2-0 (primary, exposed by lb svc), ag2-1, ag2-2.  

I have created DAG using ag1 & ag2. both are exposed using lb svc. selecting the primary of each ag cluster using lable `role=primary`.



For testitng, I have done a failover in the primary ag (ag1). and global primary changes from ag1-0 to ag1-1, Now my forwarer & other replicas of second ag are not getting synced. I have removed the primary label from ag1-0 and added primary label in ag1-1. 

how to ensure the forwared of second ag (ag2) sync with the new global primary.


i have a cluster ag1 (ag1-0, ag1-1, ag1-2) for primary ag. and if any failover occurs in primary ag. global primary changed from ag1-0 to ag1-1, then I see my forwarded ag2-0 & ag2-1, ag2-2 is not getting synced. being unhealthy. how to get the second ag (ag2) sync with the new global primary ag1-1.

I am already using load balancer svc in dag's LISTENER_URL . which points to the current primaries of ag1 & ag2. when this failvoer occurs. do I have to run any query on forwarder or new global primary to start synchronizing the dag, How the ag2 will get synced. What is the procedure?

From logs of ag2-0 (forwader), I only see that connection timeout happened, forwarder got diconnected with the global primary. But I don't know How to get this synced with the new global primary. 

I have tried ag database resume command. 
ALTER DATABASE [agtestdb] SET HADR RESUME;


Check Error Logs:


kubectl exec -it ag1-1 -- cat /var/opt/mssql/log/errorlog

Remove / Add ag2 from dag. 



On the New Global Primary (ag1-1):
Temporarily disable automatic seeding:

ALTER AVAILABILITY GROUP [DAG]
MODIFY AVAILABILITY GROUP ON 'ag2' WITH (SEEDING_MODE = MANUAL);
GO
Re-enable automatic seeding:


ALTER AVAILABILITY GROUP [DAG]
MODIFY AVAILABILITY GROUP ON 'ag2' WITH (SEEDING_MODE = AUTOMATIC);
GO


On the Forwarder AG (ag2-0):
Rejoin the DAG:


ALTER AVAILABILITY GROUP [DAG] JOIN;
GO
Resume synchronization:

ALTER DATABASE [agtestdb] SET HADR RESUME;
GO









No, if you are correctly using a Load Balancer service that dynamically points to the current primary of AG1, you should not need to run any manual queries on the forwarding AG (AG2) or the new global primary (which is still within AG1, just a different replica like ag1-1) to restart synchronization after a failover within AG1.



SQL Server Error Logs on AG2 (Forwarding AG):

Critical Information Source: Examine the SQL Server error logs on the primary replica of AG2 (ag2-0) around the time of the failover in AG1. Look for error messages related to:

Connectivity failures to AG1: Errors indicating that AG2 cannot connect to the specified endpoint for AG1.

Distributed AG synchronization issues: Errors related to the Distributed AG mechanism itself.

Network errors: Lower-level network errors that might point to connectivity problems.


















apiVersion: v1
kind: Service
metadata:
  name: ag1
  namespace: dag
  labels:
    app: ag1
spec:
  ports:
  - port: 1433
    name: db
  - port: 5022
    name: mirror
  clusterIP: None
  selector:
    app: ag1
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ag1
  namespace: dag
spec:
  serviceName: ag1
  replicas: 3
  selector:
    matchLabels:
      app: ag1
  template:
    metadata:
      labels:
        app: ag1
    spec:
      containers:
      - name: mssql
        image: neajmorshad/sql-server-2022:latest # official image: mcr.microsoft.com/mssql/server:2022-latest
        imagePullPolicy: IfNotPresent
        env:
          - name: ACCEPT_EULA
            value: "Y"
          - name: MSSQL_SA_PASSWORD
            value: "Pa55w0rd!"
          - name: MSSQL_PID
            value: "Developer"
          - name: MSSQL_AGENT_ENABLED
            value: "True"
          - name: MSSQL_ENABLE_HADR
            value: "1"
        ports:
        - containerPort: 1433
          name: db
        - containerPort: 5022
          name: mirror
        volumeMounts:
        - name: mssql
          mountPath: /var/opt/mssql
  volumeClaimTemplates:
  - metadata:
      name: mssql
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi



apiVersion: v1
kind: Service
metadata:
  name: ag2
  namespace: dag
  labels:
    app: ag2
spec:
  ports:
  - port: 1433
    name: db
  - port: 5022
    name: mirror
  clusterIP: None
  selector:
    app: ag2
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: ag2
  namespace: dag
spec:
  serviceName: ag2
  replicas: 3
  selector:
    matchLabels:
      app: ag2
  template:
    metadata:
      labels:
        app: ag2
    spec:
      containers:
      - name: mssql
        image: neajmorshad/sql-server-2022:latest # official image: mcr.microsoft.com/mssql/server:2022-latest
        imagePullPolicy: IfNotPresent
        env:
          - name: ACCEPT_EULA
            value: "Y"
          - name: MSSQL_SA_PASSWORD
            value: "Pa55w0rd!"
          - name: MSSQL_PID
            value: "Developer"
          - name: MSSQL_AGENT_ENABLED
            value: "True"
          - name: MSSQL_ENABLE_HADR
            value: "1"
        ports:
        - containerPort: 1433
          name: db
        - containerPort: 5022
          name: mirror
        volumeMounts:
        - name: mssql
          mountPath: /var/opt/mssql
  volumeClaimTemplates:
  - metadata:
      name: mssql
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 2Gi


kubectl exec -it -n dag ag1-0 -- bash
root@ag1-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
CREATE AVAILABILITY GROUP [AG1]
WITH (CLUSTER_TYPE = NONE)
FOR REPLICA ON
    N'ag1-0'
        WITH (
            ENDPOINT_URL = N'tcp://ag1-0.ag1:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        ),
    N'ag1-1'
        WITH (
            ENDPOINT_URL = N'tcp://ag1-1.ag1:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        ),
    N'ag1-2'
        WITH (
            ENDPOINT_URL = N'tcp://ag1-2.ag1:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        );
"


-- Grant the ability to create databases in the Availability Group:
root@ag1-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
ALTER AVAILABILITY GROUP [AG1] GRANT CREATE ANY DATABASE;
"
```



kubectl exec -it -n dag ag2-0 -- bash
root@ag2-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
CREATE AVAILABILITY GROUP [AG2]
WITH (CLUSTER_TYPE = NONE)
FOR REPLICA ON
    N'ag2-0'
        WITH (
            ENDPOINT_URL = N'tcp://ag2-0.ag2:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        ),
    N'ag2-1'
        WITH (
            ENDPOINT_URL = N'tcp://ag2-1.ag2:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        ),
    N'ag2-2'
        WITH (
            ENDPOINT_URL = N'tcp://ag2-2.ag2:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            SEEDING_MODE = AUTOMATIC,
            FAILOVER_MODE = MANUAL,
            SECONDARY_ROLE (ALLOW_CONNECTIONS = ALL)
        );
"


-- Grant the ability to create databases in the Availability Group:
root@ag2-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
ALTER AVAILABILITY GROUP [AG2] GRANT CREATE ANY DATABASE;
"
```



## Configure Distributed Availability Group on Kubernetes



First create the first availability group [ag1](../availability-group/ag1/readme.md) and second availability group [ag2](../availability-group/ag2/readme.md) following the steps detailed in readme file. 

Now create load balancer svc for each availability group for inter cluster communication. 

Create load balancer svc for first availability group (ag1) in the first cluster. 
`kubectl apply -f ag1-primary-svc.yaml`
```bash
kubectl get svc -n dag ag1-primary
NAME          TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                         AGE
ag1-primary   LoadBalancer   10.43.238.68   10.2.0.149    1433:32400/TCP,5022:30466/TCP   4h25m
```

Create load balancer svc for second availability group (ag2) in the second cluster.
`kubectl apply -f ag2-primary-svc.yaml`
```bash
kubectl get svc -n dag ag2-primary
NAME          TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
ag2-primary   LoadBalancer   10.43.173.130   10.2.0.188    1433:30752/TCP,5022:31052/TCP   4h24m
```


Verify which one is the primary pod (replica) of each AG. 
Run `SELECT is_local, role_desc from sys.dm_hadr_availability_replica_states` on each replica to see the role. 

Mine is `ag1-0` for ag1, and `ag2-0` for ag2. 



Add lable `role=primary` in ag1-0, ag2-0 (primary pods of each AG).
```
first cluster:
kubectl label pod ag1-0 -n dag role=primary
second cluster:
kubectl label pod ag2-0 -n dag role=primary
```



Now, create the DISTRIBUTED Availability Group named `DAG`.
ag1-primary:  EXTERNAL-IP    10.2.0.149
ag2-primary:  EXTERNAL-IP    10.2.0.188


```bash
# Primary Cluster 
kubectl exec -it -n dag ag1-0 -- bash
root@ag1-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
CREATE AVAILABILITY GROUP [DAG]  
   WITH (DISTRIBUTED)   
   AVAILABILITY GROUP ON  
      'ag1' WITH    
      (   
         LISTENER_URL = 'tcp://10.2.0.149:5022',    
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      ),   
      'ag2' WITH    
      (   
         LISTENER_URL = 'tcp://10.2.0.188 :5022',   
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      );    
GO 
"
```

```bash
# Secondary Cluster
kubectl exec -it -n dag ag2-0 -- bash
root@ag2-0:/# /opt/mssql-tools18/bin/sqlcmd -S localhost -U sa -P "Pa55w0rd!" -No -Q "
ALTER AVAILABILITY GROUP [DAG]
   JOIN   
   AVAILABILITY GROUP ON  
      'ag1' WITH    
      (   
         LISTENER_URL = 'tcp://10.2.0.149:5022',    
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      ),   
      'ag2' WITH    
      (   
         LISTENER_URL = 'tcp://10.2.0.188 :5022',   
         AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,   
         FAILOVER_MODE = MANUAL,   
         SEEDING_MODE = AUTOMATIC   
      );  
GO
"
































