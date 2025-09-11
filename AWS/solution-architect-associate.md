# AWS Solutions Architect - Associate Notes

Skipping most basic basic stuff that if you've used AWS for a bit then you probably know or Cloud Practioner stuff

## IAM

* Can have password policy on users (all the standard stuff)
* User MFA use U2F (USB) security key, Virtual MFA (phone), hardware Key fob
* For CLI, SDK, Management console - use Access Key Id and Secret Access Key
* `IAM Credential Report` to review access perms
* User groups cannot belong to other groups

## EC2
* Spot Instance Fleets uses pools of spot instances + (optional) On-Demand to meet target capacity
  * Define pools - instance type, OS, AZ
  * Strategies: 
    * lowestPrice: select lowest price set
    * diversified: Use instances from all pools
    * capacityOptimized: Pool with optimal capacity for # of instances
    * priceCapacityOptimized (recommended) - high capacity pools then cheapest
* Security groups can be used on multiple instances
* Reserved Instances are for 1 OR 3 years
* AMI specific to AZ
* Can have Public, Private and Elastic IP
* Elastic IP 
  * should not be used, expensive and DNS is better
  * Can only have 5 per AWS account (can ask AWS to increase)
  * Maintains same Public IP as long as you have it 
  * can move and attach to one instance at a time
* Placement groups: controls where EC2 instances are made
  * Cluster: single AZ, fast networking
  * Spread: spread instances across AZ's (slower networking but better fault tolerance). Up to 7 instances per AZ
  * Partition: spread instances across MANY EC2 instances (can be 100+) for things like Hadoop

* ENI's can be moved to different instances and keep the same IP addresses but only in same AZ
* Can hibernate which keeps stuff in memory for fast "boot"
  * Not on bare metal
  * root must be encrypted
  * cannot hibernate for more than 60 days
  * for On-Demand, Reserved and Spot Instances
  * Instance RAM must be < 150GB

## Storage

### EBS
* storage types: 
  * gp2, gp3 - max 16,000 IOPS
  * io1, io2 - high io, max 32k and 256k respectively
    * io1 can go to 64k if `Nitro EC2` instance
  * st1 - HDD (max 500 IOPS)
  * sc1 - HDD infrequent cheap (max 250 IOPS)
* cannot use st1 or sc1 as root/boot volume
* gp2 IO and storage size are linked 
* gp3 IO independent of storage size
* io2 CAN attach up to 16 multiple EC2 in same AZ only (`EBS Multi-attach`)
* Can set to delete w/ instance delete (root deleted by default, others not)
* Can be moved to other instances in the same AZ only (use snapshot to move to another AZ)
* To encrypt unencrypted drive, take snapshot and then encrypt it, then create volume
  * can also just create new volume and select encrypt from unencrypted snapshot

### EC2 Instance Store
* WAY faster than EBS (can be millions of IOPs vs io2 (best) EBS which is only 256k)
* Loses all data if instance stopped
* 30GB free comes with an instance

### EFS (3x's more expensive than EBS)
* petabyte scale, automatically grows
* GP and Max I/O (higher throughput, more latency)
* Bursting (IOPS related to storage space), Provisioned (specified read/write in advance), Elastic - It adjusts IOPS based on what you need (default)
* multi-AZ, need sg's for all
* linux only
* has lifecycle rules and storage tiers like S3 (`Standard`, `IA`, `Archive`) also Standard and One Zone

## ELB
* Classic (CLB) [v1]
  * 2009, mix of Layer 7 and Layer 4
  * Depreciated and **NO LONGER ON ARCHITECT EXAM**
* Application (ALB) [v2]
  * 2016, Layer 7/Application (HTTP, HTTPS, DNS, WebSocket, etc.)
  * Route based on URL path, hostname, query string, headers
  * Can route to dynamic ports and redirect to HTTPS
  * Uses target groups like EC2 instances, ECS tasks, Lambda functions, private IP's 
  * Route to multiple target groups and healthcheck is done across the entire target group
  * Application cannot see client IP as it is redirects from the LB, so if you need to see it the LB inserts the IP into the headers `X-Forwarded-For`, the Port to `X-Forwarded-Port` and the proto to `X-Forwarded-Proto` instead
  * Only DNS provided for entrypoint
* Network (NLB) [v2]
  * 2017, Layer 4/Transport (TCP, UDP)
  * Capable of handling millions of requests, very low latency, extreme performance
  * cannot be used with free tier
  * Only have **one** static IP per AZ (so if exam says app only has 1-3 IP's then think NLB)
  * Can be used to point to instances OUTSIDE of AWS with IP address target group
  * Can send to ALB
  * Healthcheck supports TCP, HTTP, HTTPS
  * uses DNS and static IP for entrypoint
* Gateway (GWLB) [v2]
  * 2020, Layer 3/Network (Packets - IP, ICMP, etc.)
  * Basically used for 3rd party security (i.e. firewalls, Intrusion Detection, Deep Packet Inspection, etc.) to analyze traffic (splits traffic among the 3rd party virtual appliance)
  * Uses GENEVE proto on port 6081

* Use v2 LB's as they have more features and CLB will be removed soon
* ALB and NLB can use sticky sessions (using cookie)
  * Application based cookie
    * Custom cookie
      * Generated by your app
      * Specified per target group
      * **NOT ON EXAM** Cannot be named `AWSALB`, `AWSALBAPP` or `AWSALBTG`     
    * Application cookie
      * Generated by LB
      * **NOT ON EXAM** Named `AWSALBAPP`
  * Duration-based Cookie
    * Generated by load balancer
    * **NOT ON EXAM** Named `AWSALB` for ALB, `AWSELB` for CLB

* Cross-Zone LB
  * Distributes load evenly across all AZ's, intead of lopsided if you don't have equal target groups in all AZ's
  * Enabled by default on ALB, no charges for traffic
  * Disabled by default on NLB, charged for traffic
  * Can even set cross zone on tg level

* SSL
  * Managed by `AWS Certificate Manager` (ACM)
  * Server Name Indicator (SNI)
    * Newer proto where client says host it's trying to connect to and server finds correct cert.
    * By doing this it can have multiple certs
    * Only on ALB and NLB, CloudFront
      
* Auto Scaling Groups (ASG) Policies:
   * Predictive scaling: forecasts load and scales accordingly
      * Good when patterns repeat
   * Dynamic Scaling: 
      * Target Tracking: i.e. keep CPU avg at 40%
      * Step Scaling: i.e. when cloudwatch alarm triggered > 70% scale out, < 30% scale int
   * Scheduled Scaling:
      * increase at 5pm on Friday
   * Scaling Cooldown - no more changes during cooldown (default 300 seconds)
     * Use AMI's to serve requests faster so they don't interfere with cooldown
   * Can scale on custom cloudwatch metrics (like # of requests)

# RDS
* Storage auto-scaling
  * Free storage less than 10% of allocated, lasts five minutes, 6 hours since last mod
  * Have to set a maximum storage threshold
* RDS Read-replicas
  * Up to 15
  * Uses Async
  * Within AZ, Cross AZ, [Cross Region (network $)]
* RDS Multi-AZ (DR)
  * Uses Sync
  * Cannot be connected to by apps
  * Auto promo to master
 * You CAN setup a RR with Multi-AZ but it will still be Async and not auto promo to master
   * If you do this you set Multi-AZ on the read replica
* RDS Custom
 * For Oracle and MSSQL only
 * Automated scaling, operation, setup
 * Admin access to OS (SSH or SSM Session Manager) to configure settings, install patches, enable native features
  * Before applying customizations, you should take a snapshot and THEN de-activate automation mode
* Aurora
  * Can have 15 read replicas but repl is faster than RDS
  * Failover is instant
  * 20% more $ than RDS but more efficient
  * storage auto-scales up to 128TB in 10GB increments
  * Aurora stores 6 copies of your data across three AZ's
    * 4 out of 6 for writes (quorum)
    * 3 out of 6 for reads
    * self-healing with peer-to-peer 
    * storage striped across 100s of volumes
  * Custom Endpoints can be used to specify different groups of read replica's to hit for different applications. Typically the main read replica is not used if you use this.  You can also then say different custom endpoint groups have different servers behind them (i.e. r3g.medium and r5g.2xlarge)
  * Serverless - Good for unpredicatable workloads
  * Global Aurora
    * Up to 5 secondary regions (read only)
    * Repl lag is 1s cross-region **exam q about 1 second cross region, it means probably asking about global aurora**
    * RTO is < 1 minute to failover
    * Up to 16 read replicas per region
  * ML
    * You can have ML using SageMaker (any ML) or Comprehend (sentiment analysis)
    * Enables you to run certain queries where you can ask i.e. "recommended products for the user" and aurora will send data to the ML, the ML will return predictions and it will return as the query.  This is instead of doing i.e. where clauses and programmatically finding data
  * Babelfish allows Postgres to understand T-SQL (MSSQL).
* Backups 
  * RDS Backups 
    * Automated Backups
      * stored 1-35 days (or set to 0 to disable)
      * transaction log Every 5 minutes
      * daily full backup
    * Manual Backup
      * Store as long as you want  
    * You still pay for storage on a RDS DB, cheaper to backup and restore if going to be offline for long
  * Aurora Backups
    * stored 1 to 35 days but cannot be disabled
    * point-of-time of any time
    * manual is same as RDS
  * All restores are made in new DB
  * Restoring from on-prem
    * RDS MySQL: Take normal backup, put in S3, restore to RDS instance
    * Aurora MySQL: Take backup w/ Percona XtraBackup, put in S3, Restore to new Aurora MySQL cluster
  * Aurora Cloning is faster and cheaper than snapshot/restore because it initially made on same data volume and THEN data (old and new) is copied to new volume to separate.
* Security
  * At-rest encryption
    * KMS must be definied at launch
    * read replicas cannot be encrypted without master also
    * to encrypt unencrypted, snapshot, restore as encrypted
  * in-flight: uses AWS TLS root cert by default
  * use IAM roles (must enable) instead of user/pass (not on RDS custom)
  * No SSH except RDS Custom
  * Can enable Audit logs, if you want to keep for longer send to CloudWatch Logs
* RDS Proxy
  * connection pooling that reduces CPU, RAM on instances by lowering open connections
  * No code change, just different point
  * MUST use IAM, not user/pass (q on exam about enforcing IAM, probably asking about RDS proxy)
  * Reduces failover time by 66%
  * Cannot use proxy publicly
* Elasticache
  * Can use REDIS (which is more fault tolerant and uses horizontal scaling for perf) or Memcached (multi-node sharding and multi-threaded, but not persistant)
  * Security
    * IAM Auth, but only for AWS API-level
    * KMS At rest
    * Redis Auth
      * Supports SSL
      * Uses user/pass
    * Memcached
      * Supports SASL based auth
  * Key value store, can store session data, cannot use SQL
    * Redis has sorted sets to allow for data to be sorted automatically (i.e. gaming leaderboard)
  * Backup/ Snapshot/ PITR restore
  * Managed and Scheduled Maint
  * Must select an ElastiCache instance type (e.g. cache.m6g.large)
  * Requires a code change so **if exam Q about caching solution that DOES NOT require a code change, it's NOT elasticache**

# Ports
No reason to remember all these (and probably know anyways), just need to know the difference between important ports and DB ports, review right before exam just in case
* Important ports:
  * FTP: 21
  * SSH: 22
  * SFTP: 22 (same as SSH)
  * HTTP: 80
  * HTTPS: 443
* RDS Databases ports:
  * PostgreSQL and Aurora PortgreSQL: 5432
  * MySQL and MariaDB and Aurora Mysql: 3306
  * Oracle RDS: 1521
  * MSSQL Server: 1433

# Route 53
* Only 100% available SLA AWS Service
* Understand TTL
* Must know A, AAAA (same as A but IPv6), CNAME and NS for exam, other records not so much
* Pay $.50/mo per hosted zone (aka not free)
* Public Zone is for the internet, Private zone is inter-VPC (1 or more) only
* Alias records (A/AAAA with Alias option checked) - For pointing to AWS resources
  * free of charge for DNS queries, CNAME is not
  * for root domain and non root, CNAME is only non-root
  * Cannot set alias for EC2 DNS name
  * Cannot set TTL
  * Can have healthcheck
  * Targets:
    * ELB
    * CloudFront
    * API Gateway
    * Elastic Beanstalk env
    * S3 Website
    * VPC Interface Endpoint
    * Global Accelerator accelerator
    * Another record in the same hosted zone
* Routing Policies
  * Simple - single resource or random
    * if using multiple records, all with the same record name, they are all returned and **CLIENT** picks one randomly
    * If using Alias, only one resource can be set
    * Can't be associated with Health Check (without alias)
  * Weighted - percentage of requests to different hosts
    * Use multiple records, all with the same record name, then assign different percentage
    * 0% will receive no traffic, if all 0% then it sends all records (like multi-Simple Routing)
    * Can use healthcheck (without alias)
  * Latency - Send client lowest latency (not necessarily closest!) host
    * Use multiple records, all with the same record name, then assign region
    * Can use healthcheck (without alias)
  * Geolocation - Send client based on location to specific resouce
    * Used for different language apps or something location specific.
  * Geoproximity - Send clients based on their location and bias
    * Set bias for a resource which will shift traffic from one region to another based on bias
    * It's sort of like Weighted except it takes into account the users location
  * Ip-based - Send clients based on their IP address and a CIDR range you specify for a resource
    * If user is not in CIDR range then they receive no response
  * Multi-Value - Allow healthchecks when using multiple resources
    * Basically the same as Simple if you were using multiple resources (random), but with healthchecks
    * Only returns healthy resources
    * Not a replacement for a ELB since it is still client side, not server side
* Healthchecks
  * For public resources only
  * Integrated into Cloudwatch metrics
  * 15 global healthcheckers, 3 (or 18%) (default) for must be healthy for threshold, otherwise unhealthy
  * Checks are every 30 sec default, can be set up to 10 sec ($)
  * Can monitor:
    * Endpoints
      * HTTP, HTTPS, TCP
      * Can select which health checker locations
      * Healthy is 2XX or 3xx
      * Also can set it to check the first 5120 bytes of response for specific text
      * Must set firewall to allow health checkers
    * Calculated Health Checks (Health checks that check other health checks)
      * Can monitor up to 256 child health checks
      * Uses `OR`, `AND` or `NOT`
      * Specify how many to pass to make parent pass
      * Used to i.e. perform maint on website without causing ALL healthchecks to fail
    * CloudWatch Metric
      * Used to check health of private AWS resources sense health checkers are public only
* Route 53 Resolvers
  * Used to do internal lookups of IP's for use across VPN's or Direct Connect between AWS and on-prem
  * (Inbound) Internal DNS servers can forward to a Route 53 resolver if the DNS suffix matches something specific and the resolver will return the internal IP's of the resources instead of having to use public DNS entries
  * (Outbound) same as Inbound but in reverse
  * Allows for more security than putting private IP and DNS entries into public Route 53 (i.e. for connections over the VPN)
  * Allows for split horizon DNS (internal DNS give IP, public gives another for the same resource)
  * Resolver Rules
    * Conditional Forwarding Rules (Forwarding Rules) - Specific domain/sub-domain to target IP addresses
    * System Rules - overrides subdomain set by conditional forwarding rules
      * i.e. conditional forward rules says *.example.com go here, but you can set *.test.example.com goes here
    * Auto-definied System Rules - AWS internal domain names, Private Hosted Zones
    * If multiple matches, Route 53 Resolve picks most specific match
    * Resolver Rules can be shared across accounts using AWS Resource Access Manager (RAM)
      * Allows central management of DNS queries from multiple VPC's to target IP
      
# S3
* Object key is the full path to the files, there is no real directories, it's just that the key is the `prefix` (which can contain `/`) + the file name.
* Naming convention
  * No uppercase, no underscore
  * Not an IP
  * Must start with lowercase letter
  * MUST NOT start with the fredix xn--
  * MUST NOT end with suffix -s3alias
* Max size of object is 5TB
* If more then 5GB then it must be a multi-part upload
* Each object has metadata, tags and version ID
* Bucket Access Security
  * Policies
    * Types
      * User-based - specific user
      * Resource-based
        * Bucket Policies - bucket wide
          * most common
          * Uses JSON that has following fields:
            * Resources - the bucket and objects that the policy applies to
            * Effect - Allow / Deny
            * Principal - The account or user to apply the policy to
          * Used for:
            * Make bucket public
            * Force objects to be encrypted at upload
            * Grant access to another account (cross account)
        * Object Access Control List (ACL) - finer grain (can be disabled)
        * Bucket Access Control List (ACL) - less common (can be disabled)
    * IAM principal can access if user OR resource policy allows AND there is no deny
  * Bucket Setting for `Block Public Access`
    * Can block all public access; overrides bucket policies
    * Can be set at the account level (applies to all buckets on account)
* Versioning
  * Bucket level which uses keys from version (1, 2, 3, etc)
  * Can be enabled/disabled at any time
    * Enabling - if versioning was not on, previous version #'s will be `null`
    * Disabling - safe to do, does not delete previous versions
  * Once versioning is enabled for the bucket:
    * To restore a previous version, you must select the `Show versions` option in the GUI, then select the newer version of the file and permanently delete it, this will make the last verion the current version. 
    * To create a delete marker (file is "hidden" as it is marked for deletion) turn off the `Show versions` option in the GUI and then delete the object (not it will NOT say permanently delete).  To undelete it you must turn on the `Show versions` option and delete the delete marker. Delete marker files will not show up if `Show versions` option is disabled.
* Replication
  * You can do Cross region replication (CRR) or Same region replication (SRR)
  * MUST enabled versioning on both source and destination buckets
  * Buckets can be in diffeent AWS accounts
  * Copy is async
  * Must give permissions to S3 for copying
  * Objects maintain the same Version ID in the destination bucket upon repl
  * Only NEW objects can be replicated after replication is enabled
    * If you need to replicate existing objects, you need to use S3 Batch Replication
  * You can replicate delete markers (optional)
  * Permanent deletions of items with the version id are never replicated
  * You cannot "chain" replication (i.e. Bucket 1 --> Bucket 2 --> Bucket 3, Bucket 3 would not get Bucket 1 objects)
* Storage
  * Durability - all storage classes have the same durability (99.999999999% SLA), very low chance of losing objects
  * Availibility - varies depending on class
  * Should know all the storage classes like on the Cloud Practitioner exam, they are below in case:
    * S3 Standard - For normal high throughput, high avail stuff
    * S3 Standard IA (Infrequent Access) - For less frequent access, use for backups
    * S3 One Zone IA - Only in single AZ, all data lost if AZ is destroyed, use for secondary backup or data that can be easily recreated
    * S3 Glacier Instant Retrieval - millisecond retrieval, must keep data for min 90 days
    * S3 Glacier Flexible Retrieval - 1-5 minutes for expedited retrieval ($$), 3-5 hours for standard ($), 5-12 hours for Bulk (free). Minimum 90 days storage
    * S3 Glacier Deep Archive - Stadard Retrieval ($$) 12 Hours, Bulk ($) 48 hours. Min Storage of 180 days
    * S3 Intelligent-Tiering
      * Moves objects automatically between access tiers based on usage
      * Small monthly monitoring and auto-tiering fee
      * No retrieval charges
      * Tiers
        * Frequent Access (auto) - default
        * IA (auto) - objects not accessed for 30 days
        * Archive Instant Access (auto) - objects not access for 90 days
        * Archive Access (optional) - configurable from 90 - 700+ days
        * Deep Archives Access (optional) - configurable from 180 - 700+ days
  * Lifecycle rules can be configured to move objects between tiers based on days after object creation
    * You can also delete incomplete multi-part uploads, old version of files or access files (after 365 fays)
    * You can do rules by prefix and/or by tag as well
  * S3 Storage Analytics can analyze objects and give recommendations on when to transition objects to different classes. 
    * It only works for `Standard` and `Standard IA`. 
    * Report updated daily
    * 24-48 hours after enabling to start seeing data analysis
* S3 Requester Pays
  * The requester of the object pays to download the file, not the bucket owner
  * Must be authenticated with AWS, not anonymous (so AWS knows how to bill them)
* Event Notifications (fire Lamba, send SNS, send to SQS, send to EventBridge)
  * Object name filtering available (i.e. *.jpg only)
  * Typically event notification in seconds, but sometimes can be a minute or more
  * IAM Permissions must be defined on the event target, not on S3
  * Eventbridge (newer) allows to send to over 18 AWS services
    * Allows further filtering via JSON rules (metadata, object_size, etc.)
    * Can send to multiple destinations at once
    * Can Archive, Replay Events, reliable delivery
* S3 Performance
  * Up to 3,500 request/sec PUT/COPY/POST/DELETE and 5,500 requests/sec GET/HEAD per prefix
    * So if you have four files spread across four prefixes (i.e. /1/*. /2/*, /3/*, /4/*) you could get up to 22,000 GET/HEAD requests/sec
  * Multi-part upload recommended for > 100MB, required for > 5GB
    * Can parallelize
  * S3 Transfer Acceleration
    * Transfer to edger location and then forwarded to bucket region
    * Compatible with multi-part upload
    * Faster as it spends less time going through the internet
  * Byte-Range Fetches
    * Can request GETs by specific byte range to split up file
      * Can parallelize download to speed up download
      * Can help with failures of a specific part (instead of downloading entire thing again)
      * Can just get specific part of file i.e. if you just need header and know header is in first 100 bytes, just request that
* S3 Batch Operations
  * Perform bulk operations on existing S3 objects
    * Modify object metadata, copy objects to other buckets, encrypt un-encrypted objects, modify ACL's and tags, restore objects from S3 Glacier, invoke Lambda for custom function
    * A job consists of list of object, the action to perform and optional parameters
    * Manages retires, tracks progress, sends completion notifications, generates reports,
    * S3 Inventory to get objects --> use Athena to query and filter your objects --> S3 Batch
* S3 Storage Lens
  * Analyze/optimize storage acorss entire AWS Organization
  * Find anomalies, identifiy cost efficiencies, apply data protection best practices,
  * Can aggregate data for Org, specific accounts, regions, buckets or prefixes
  * Default or make your own dashboard
    * Default dashboard 
      * Shows free and advanced metric
      * Can't be deleted but can be disabled
      * Shows Multi-Region and Multi-Account data
  * Can export metrics daily to S3 (CSV, Parquet)
  * Metrics (don't need to know metric names, just examples for what metric is for)
    * Summary
      * i.e. StorageBytes, ObjectCount. 
      * Used to find fast growing or not used buckets/prefixes
    * Cost Optimization
      * i.e - NonCurrentVersionStorageBytes, IncompleteMultipartUploadStorageBytes. 
      * Used to find cost savings
    * Data protection
      * i.e - VersioningEnabledBuckCount, CrossRegionReplicationRuleCount, etc. 
      * Used to identify buckets that aren't following data protection best practices
    * Access management
      * i.e. - ObjectOwnershipBucketOwnerEnforceBucketCount
      * Identify Object Ownership settings for buckets in use
    * Event 
      * i.e. - EventNotificationEnabledBucetCount
      * user case: identify which buckets have event notifications enabled
    * Performance
      * i.e. - TransferAccelerationEnabledBucketCount
      * use case - identify which buckets have S3 Transfer Acceleration enabled
    * Activity 
      * i.e. AllRequests, GetRequests, PutRequests, etc.
      * user case: get insights on how storage is requested
    * Detailed Status Code
      * i.e. - 200OKStatusCount, 403ForbiddenErrorCount, etc.
      * Insights by status code
  * Free vs Advanced
    * Free
      * 28 metrics
      * Last 14 days
    * Advanced
      * Advanced Metrics: Activity, Advanced Cost Optimization, Advanced Data Protection, Status Code
      * CloudWatch Publishing - Can view Lens metrics in CloudWatch (no extra charge)
      * Prefix Aggregation - Collect metrics at prefix level
      * Last 15 months
* Security
  * Encryption
    * Server side (SSE)
      * Amazon managed S3 KMS keys - enabled by default for new objects and new buckets (SSE-S3)
        * AES-256
        * Must set header x-amz-server-side-encryption: "AWS256"
      * KMS Keys stored in AWS KMS (SSE-KMS)
        * Can audit key in CloudTrail and user controls key creation
        * Must set header x-amz-server-side-encryption: "AWS:kms"
        * Limitations
          * KMS Service limitations as every upload called `GenerateDataKey` KMS API
          * 5500/10000/30000 requests/s depending on region
          * Can request increase in the Service Quotes Console
      * Customer Provided Keys (fully managed by customer)
        * Amazon S3 does not store encryption keys
        * HTTPS must be used
        * Encryption key is supplied in HTTP header for every HTTP request
      * DSSE-KMS - Double encryption based on KMS *NEW OPTION, PROBABLY NOT ON TEST YET*
    * Client side 
      * Client has to encrypt/decrypt data before/after sending to S3
      
    * Encryption in transit is done via SSL/TLS. You can disable HTTP by setting the policy `aws:SecureTransport: "false"`
    * You can set bucket policies to refuse API calls to S3 unless they have encryption headers for SSE-KMS or SSE-C, this will "force encryption"
    * Bucket policies are evaluated BEFORE S3 encryption
  * CORS **Most likely will be on the test, very popular question for the exam**
    * Bucket level setting (permissions tab)
    * Origin = protocol + host + port
      * Same origin: http://www.example.com/app1 and http://www.example.com/app2
      * different origin: http://www.example.com/ and http://other.example.com
    * Web browser security to allow content from one website (i.e. http://www.example.com/index.html) access to content on another website (http://pics.example.com/image.png)
      * This occurs by a preflight request to the "other" website, that website will send back it's CORS headers and if the first origin is allowed then the browser will be allowed
    * You can specify an origin or ALL origins with `*`
  * MFA Delete
    * Force users to MFA before doing important operations in S3. Import operations:
      * Permanently delete an object versions
      * SuspendVersioning on the bucket
    * Versioning MUST be enabled on the bucket to enabled MFA Delete
    * Only the root account can enable/disable MFA Delete
    * Currently can only enable/disable and do deletes via CLI
  * Logging
    * Access logs will log all authorized or denied access from any account into another S3 bucket
      * The target for logging bucket must be in the same AWS region
    * Never set the logging bucket to the same bucket as you want access logged as it will infinitely loop!
  * Pre-signed URLs - temporary access to files
    * URL expiration
      * S3 Console - 1 minute up to 720 mins (12 hours)
      * AWS CLI - defalt 3600 seconds, max 604800 seconds (168 hours)
    * When you generate a URL, the URL will grant the permissions of the user that generated the URL for Get/PUT
    * Examples:
      * Allow only logged-in users access to download a premium video
      * Allows constantly changing user list to download files
      * Temporarily allow a user to upload a file to your private bucket
  * S3 Glacier Vault Lock
    * Write once, read many (WORM) model
    * Once you set a lock policy, an object that is put into the glacier vault cannot be deleted by ANYBODY
    * versioning must be enabled
    * is set per object, set object to not be able to be deleted for a specified amount of time
    * Retention mode - Compliance
      * Objects cannot be overwritten or deleted by any user, including root
      * Retention mode can't be changed, retention period can't be shorted
    * Retention mode - Governance
      * Some users have permissions to change retention or delete object
    * Retention periods CAN be extended
    * Legal Hold
      * protect object indefinitely, independent from retention period
      * User has S3:PutObjectLegalHold permissions to place or remove
  * S3 Access Points
    * Can set specific users/groups to have access to a prefix of a S3 bucket with specific permissions
    * Simplifies security management
    * Each access point has it's own DNS name and access policy
    * You can set the access point to only be accessible within the VPC
      * Must create a VPC endpoint to access the access point
      * VPC Endpoint policy must allow access to the target bucket AND access point
  * S3 Object Lambda - by using a S3 Access point you can use lambda to create a different version of the file (i.e. redact certain info, add to file) and then have a S3 Object Lambda Access Point to retrieve that changed file
    
# CloudFront
* About
  * CloudFront is a CDN (so if you hear CDN on exam, it's CloudFront)
  * DDoS protection (because it's worldwide) and has Shield and AWS WAF
  * 216 or more edge locations globally
* Origins
  * S3 bucket which is secured with Origin Access Control (OAC)
  * VPC - applications hosting inside VPC's (ALB, NLB, EC2 instances)
  * Custom Origin (HTTP) - S3 bucket as static website OR any public http backend
* How it works
  * Client connects to edge location
  * Origin sends data to edge location through private AWS connection where it is cached locally
* CloudFront vs S3 CRR
  * Cloudfront 
    * Works with ALL Edge locations
    * Files are cached with a TTL
    * Great for static content
  * S3 CRR
    * Must be setup per region
    * Files updated in near real time
    * Read only
    * Grant for dynamic content that needs low-latency in a few regions
* In order to allow CloudFront access to a VPC app, you should use a VPC Origin (new, more secure) which allows traffic to go through all private connections, not make the instance public (old way, less secure)
* Geo Restriction is used to allow or block specific countries
* Pricing
  * Cost of data out per edge location varies
  * Price class - All: All regions
  * Price class - 200 - Most regions, skip most expensive regions
  * Price class - 100 - Only least expensive regions
* CloudFront Invalidations
  * If back-end updates, CloudFront will not update until TTL expires
  * Can force an entire (*) or partial (/images/*) CloudFront invalidation to bust cache

# AWS Global Accelerator
* Uses Anycast Public IP's (all servers have the same IP, but client is sent to closest one)
* 2 Anycast Ip's are created for your app
* Traffic is sent to Edge locations then through private AWS connection to your app
* Works with Elastic IP, EC2 Instances, ALB, NLB, public or private
* Health Checks - global health check fails over in less than 1 minute (makes it great for DR)
* Security 
  * Only 2 external IP's to be whitelisted
  * DDoS protection with AWS Shield
* Great for gaming servers, VoIP, IoY, things that require static IP, fast failover of faster performance of TCP/UDP

### Difference between CloudFront and AWS Global Accelerator is that CloudFront serves from the edge location, GA is sending data through AWS backbone to your app directly which improves performance

# AWS Snowball (aka Snow family)
* Overview
  * For edge computing or data migrations into the cloud
  * Devices have same amount of RAM and vCPU
    * Storage Optimized is 210TB
    * Compute Optimized is 28TB
  * Recommendation is that if it's going to take over a week to transfer data to AWS, you should use Snowball
  * Can also be used for cases where there is no or low internet (i.e. - ship, underground)
  * Can run EC2 instances or Lambda functions on the device to do computing at the edge.
    * This allows you to do things like preprocess data, transcode media, etc.
  * Snowball always imports directly into S3 once it is received by AWS.
    * If you want to import into Glacier you need to set a S3 lifecycle policy on the S3 bucket that data is imported into

# Amazon FSx
* Allows third-party file systems on AWS
* FSx for Windows
  * Supports SMB, NTFS, Active Directory, ACLs, user quotes
  * Can be mounted to Linux EC2 instances
  * Can use MS Distributed File System (DFS) to group files across multiple FS (i.e. make visible on prem)
  * Scale 10s of GB/s, millions of IOPS, 100s of PB of data
  * SSH or FDD
  * Can be configured for Multi-AZ for HA
  * Can be configured to connect to in-prem with VPN or Direct Connect)
  * Data backed up to S3 daily
* FSx for Lustre
  * Parallel distributed file system (Lustre = Linux + cluster) used for Machine Learning and **High Performance Computing (HPC)** *(if you see HPC on exam, it's talking about this)*
  * Also used for video processing, financial modeling, electronic design automation
  * SSD and HDD
  * **Seamless integration with S3** (<--- may be on exam)
  * Can "read S3" as a file system
  * Can write the output of computations back to S3 (through FSx)
  * Deployment Options:
    * Scratch File System
      * Temporary Storage
      * Data not replicated (lost if FS fails)
      * High speed (6x's faster)
      * Use: short term processing or to optimize costs
    * Persistent File System
      * Long term storage
      * Data replicated within same AZ - Replace failed files in minutes
      * Usage: Long term processing, sensitive data
* FSx on NetApp ONTAP
  * compatible with NFS, SMB, iSCSI
  * for workloads running on ONTAP or NAS
  * Broad compatibility - Works with Windows, Linux, MacOS, VMWareCloud on AWS, Amazon Workspaces & AppStream 2.0, EC2, ECS, EKS
  * storage auto grows and shrinks
  * snapshots, replication, low-cost, comporession and data de-dupe
  * **Point-in-time instantaneous cloning** (helpful to testing new workloads)
* FSx for OpenZFS
  * compatible with NFS (v3, v4, v4.1, v4.2)
  * move ZFS workloads to AWS
  * Broad compatibility like FSX NetApp
  * Up to 1 million IOPS with < .5ms latency
  * Snapshots, compression, low-cost 
    * DOES NOT have data de-dupe
  * **Point-in-time instantaneous cloning** (helpful to testing new workloads)

# Storage Gateway
* Bridge between on-prem and cloud data
* All gateways are install on-prem  in VM or in EC2 (except Volume, can't use EC2)
* S3 File Gateway
  * Allows NFS or SMB on prem to be translated to HTTPS calls to S3 buckets (can use whatever S3 type but not Glacier - use lifecycle rules for that)
  * Most recently used data is cached in the file gateway
  * GW bucket access via IAM permissions
  * SMB can use Active Directory user auth
* Volume Gateway
  * block storage using iSCSI protocol backed by S3
  * Volume types
    * Cached - low latency to most recent data
    * Stored - entire dataset on-prem, scheduled backup to S3
  * Can make EBS snapshots to restore on-prem volumes
* Tape Gateway
  * backs up tapes using iSCSI Virtual Tap Library (VTL) to S3
  * Backup data using existing tap infra, then data is copied to S3 
  * works with leading backup software vendors

# AWS Transfer Family
* Used to FTP to AWS (all types)
* Pay per provisioned endpoint per hour + data transfers in GB
* Store and manage user credentials within service OR integrate with existing auth system (i.e. MS AD, LDAP, Okta, Cognito, custom)
* Highly Available and Scalable
* FTP to S3 or EFS

# AWS DataSync
* **Appearing a lot on exam recently**
* For moving large amounts of data
  * To AWS <--> on-prem or other clouds - needs agent
  * AWS <--> AWS - no agent needed
* Can sync data between S3, EFS, FSx
* Repl tasks scheduled hourly, daily, weekly
* **File permissions and metadata are preserved (NFS POSIX, SMB)** *Unique to this service if asked on exam!*
* Agent can use up to 10Gbps, but bandwidth limit an be set
* Can use AWS Snow Family to assist which has agent pre-installed on device

### Differences when dealing with data syncing *(can be on exam)*
* Data integration --> `Storage gateway` ( caching frequently accessed data on premises and stores archives in AWS)
* Data migration --> `Data Sync` ( out and out a migration service used to migrate active and archive data in a scheduled manner)
 
# Quick AWS Storage Overview
* S3: Object Storage
* S3 Glacier: Object Archival
* EBS volume: Network storage for one EC2 instance at a time
* Instance Storage: Physical storage for EC2 instance (high IOPS, low durability)
* EFS: Network FS for Linux instances
* FSx for Windows: Network FS for Windows
* FSx for Lustre: High Performance Computing Linux FS
* FSx for NetApp ONTAP: High OS Compability
* FSx for OpenZFS: Managed ZFS FS
* Storage Gateway: S3 and DSx File Gateway, Volume Gateway (cache & stored), Tape gateway
* Transfer Family: FTPS interface on top of S3/EFS
* DataSync: Schedule data sync from somewhere to AWS or AWS to AWS
* Snow Family: Move large amounts of data to cloud through physical device
* Database: for specific workloads, usually with indexing and querying

# SQS
* Attributes
  * Default retention:4 days, max 14 days
  * Limit of 256KB/message, but unlimited messages, unlimited throughput
  * Can have duplicate messages and out of order messages
  * Messages are persisted until consumed and deleted OR retention period passes
  * Up to 10 messages at a time when polling
  * Use ASG with the CloudWatch metric `ApproximateNumberOfMessages` to trigger a CloudWatch Alarm to scale out/in ASG - **Common integration on exam**
  * No need to provision throughput
* Security (same as SNS)
  * Encryption
    * In-flight using HTTPS API (default)
    * At rest using KMS (must be enabled)
    * Client-side if client wants to perform encrypt/decrypt 
  * IAM policies can be used for access control
  * SQS Access Policies (similar to S3 bucket policies)
    * Use for cross account access and/or other services (SNS, S3, Lambda) to write to SQS
* Message Visibility
  * By default messag visibility is for 30 seconds, during this a message is not visible to other consumers once polled
  * If the client needs more time to process the message, the `ChangeMessageVisibility` API call can be used to give that message more time
* Long Polling
  * When polling the client can wait for messages to appear. As soon as a message(s) appears, it will be sent to the client.
  * Long polling wait time can be 1 - 20 seconds by adjusting `WaitTimeSeconds` on the queue or the API level
  * Preferrable to short polling as it reduces API calls AND reduces latency as messages are consumed RIGHT when they appear, as opposed to the next API call
* FIFO Queues
  * Messages are sent in order, but limited throughput (300 msg/s without batching, 3000 msg/s with)
  * Exactly once send capability (by removing duplicates using Deuplication ID)
  * Ordered by Message Group ID (messages in the queue are put into groups) (mandatory parameter)
  * Queue MUST be created with the suffix `.fifo`

# SNS
* Attributes
  * Up to 12.5 Million subscriptions per topic
  * All subscribers will get all messages (although there is new feature to filter)
  * 100,000 topics limit
  * No need to provision throughput
* Security (same as SQS)
  * Encryption
    * In-flight using HTTPS API (default)
    * At rest using KMS (must be enabled)
    * Client-side if client wants to perform encrypt/decrypt 
  * IAM policies can be used for access control
  * SQS Access Policies (similar to S3 bucket policies)
    * Use for cross account access and/or other services (SNS, S3, Lambda) to write to SQS
* Can send to email, lambda, HTTP Endpoint, FireHose, SMS/Mobile, SQS
* Data is not persisted (lost if not delivered)

# SNS and SQS
* Your app --> SNS which can send to multiple SQS which can be picked up by app
  * This allows for no data loss do to throughput limitations
  * You can add more SQS subscribers over time
  * Make sure your SQS access policy allows for SNS to write
  * Can send to SQS in other regions
* For S3, for any event type (i.e. object create) and prefix (i.e. images/) you can only have one S3 Event rule
  * By fanning out you can allow multiple things to consume a single event
* SNS can be FIFO
  * Must have `.fifo` suffix
  * Same as SQS (ordering my message ID, dedupe by dedupe ID)
  * Can have a Standard or FIFO SQS Subscriber
  * Limited through put to same as SQS
* Subscriber Message filtering
  * JSON policy to filter out messages per subscriber

# Kinesis Data Streams
* Collect and store streaming data in real time
  * Great for click streams, IoT devices, metrics & logs (things that input millions of records)
* Retention up to 365 days
  * data cannot be deleted, it has to expire
  * ability to reprocess data (replay) by consumers 
  * can have infinite consumers in parallel
  * BIG difference vs SQS where the data can only have one consumer per queue as messages are deleted
* Data up to 1MB (but really you want to use a lot of "small" real time data)
* data ordering guarantee with same `Partition ID`
  * This can group data across multiple shards
* at rest KMS encryption, HTTPS in-flight
* Capacity
  * Provisioned Mode:
    * Choose # of shards
    * Pay per shard provisioned per hour and you can scale them manually
    * Each shard gets 1MB/s in (or 1000 records/sec) and 2MB/s out
    * Can use shard estimator tool to help determine how many shards you should have
    * When increasing shards you can only double shards in one operation (1 --> 2, 2 --> 4. You can't do 1 --> 5)
  * On-demand Mode:
    * No need to provision or manage capacity
    * Default capacity provisioned (4MB/s in or 4000 records per second)
    * Scales automatically based on observed throughput peak during last 30 days
    * Pay per stream per hour & data in/out per GB
    * Max write 200 MiB/sec & 200K records/sec
    * Max read 400 MiB/second (per consumer). Up to 2 default consumers or 20 with EFO
 
# Amazon Data Firehose (formally Kinesis Data Firehose)
  * Takes data from a producer with records up to 1MB
  * Can transform record with a lambda
  * Can put all or failed data into a S3 backup bucket
  * Batches data together and writes to:
    * AWS Destinations like S3, RedShift or OpenSearch **Remember these three for exam**
    * Third-parties (i.e. - DataDog, MongoDB, etc.)
    * Custom HTTP Endpoint
  * Fully Managed Service, serverless
  * Automatic scaling, pay for what you use
  * Near Real-Time with buffering capability based on size/time (**Near Real-Time on exam typically meanins Firehose**)
  * Supported CSV, JSON, Parquet, Avro, Raw Text & Binary Data
  * Conversions to Parquet/ ORC, compressions with gzip/snappy/zip
  * Does not store data, just batches and writes data to somewhere
  * By default, Firehose waits for 5 MiB buffer size or 300 seconds interval before delivering records

# Amazon MQ
* Since SQS/SNS are "cloud-native" proprietary AWS services you may not want to re-architect traditional on-prem protocols like:
  * MQTT, AMQP, STOMP, OpenWire, WSS
* Managed message broker service for:
  * RabbitMQ
  * ActiveMQ
* Amazon MQ doesn't "scale" as much as SQS/SNS (which are basically infinite)
* Runs on servers, can run in Multi-AZ with failover
* has queue feature (SQS) and topic features (SNS)
* In HA you use EFS to store messages and then a failover can occur to a different AZ without losing data

# ECS
* EC2 Launch Type
  * You must provision and maintain the EC2 instances
  * Each instance must run the RCS Agent to register in the ECS Cluster
  * AWS takes care of starting/stopping containers
* Fargate - (we already know this) - exam will supposedly harp on how you should use Fargate over EC2 instances (for obvious reasons)
* IAM Roles
  * EC2 Instance Profiles
    * Used by the ECS Agent
    * Makes API calls to ECS Service
    * Send container logs to CloudWatch Logs
    * Pull Docker image from ECR
    * Retrieve data from Secrets Manager or SSM Parameter Store
  * Task Role
    * Allows each task to have a specific role
    * Defined in task definition
* Can use load balancers - use ALB normally, only use NLB for high throughput/high performance use cases or to pair with AWS Private Link
* Use EFS to have the same filesystems across AZ's for all tasks
  * Fargate + EFS = completely serverless
* Auto Scaling
  * AWS Application Auto Scaling uses Avg. ECS Service CPU Util, Memory Util and ALB Request Count Per Target
  * Kinds
    * Target Tracking - scaled based on CloudWatch metric
    * Step Scaling - Scale based on CloudWatch Alarm
    * Scheduled - Scale based on specified date/time
  * Remember that ECS Service Auto Scaling (task level ) != EC2 Auto Scaling (EC2 instance level)
    * Obviously not an issue if using fargate
  * Auto scaling EC2 Instances
    * Auto Scaling Group - Scale based on CPU Util 
    * EC2 Cluster Capacity Provider - automatically provisions and scales infra when EC2 instances are missing capacity (CPU, RAM, etc.)
      * Pairs with ASG
      * Smarter than plain ASG and should be used instead

# ECR
* Store Docker images on AWS (private and public)
* Fully integrated with ECS, backed by S3
* Access controlled through IAM

# EKS
* Open source container platform that uses Kubernetes. Similar to ECS
* Cloud-agnostic - Can use K8's oon-prem or on any cloud provider (unlike ECS which is closed-source and AWS only)
* Uses pods instead of ECS Services (if you see pods on exam, it's talking about EKS)
* Node Types
  * Managed Node Groups
    * Creates and manages Nodes (EC2 instances) for you
    * Nodes are part of ASG managed by EKS
    * Supported on On-Demand or Spot Instances
  * Self-Managed Nodes
    * Nodes created by you and registered to the EKS cluster and managed by an ASG
    * You can use prebuild AMI - Amazon EKS Optimized AMI
    * Supports On-Demand or Spot Instances
  * AWS Fargate
    No maitenance, no nodes managed
* Data Volumes
  * Need to specify StorageClass manifest on EKS cluster
  * Leverages Container Storage Interface (CSI) compliant driver
  * Support for:
    * EBS
    * EFS
    * FSx for Lustre
    * FSx for NetApp ONTap

# AWS App Runner
* Fully managed service to deploy web applications and API's at scale
* No infra experiance required
* Start with source code or container images, configure settings (vCPU, RAM, Auto Scaling, Health Check, etc.)
* Automatically builds and deploys the web app
* Comes with automatic scaling, highly available, load balancer, encryption
* VPC access support
* Connect to database, cache and message queue
* Used for web apps, API's microservices, rapid production deploys

# AWS App2Container
* CLI tool for migrating Java and .NET web apps into Docker containers
* Lift and shift apps running on bare metal into AWS without code changes
* Generates CloudFormation templates (compute, network, etc.)
* Registers generated Docker containers to ECR
* Deploys to ECS, EKS or App Runner
* Supports pre-build CI/CD pipelines
* Just need to remember for quiz that if you want to migrate a Java or .NET app into container on AWS, this is what you use

# AWS Lambda
* Languages (don't need to remember all of these, just remember is has a lot of support for languages especially Node.js and Python)
  * Node.js (JavaScript)
  * Python
  * Java
  * C# (.NET Core) / PowerShell
  * Ruby
  * Customer Runtime API (community supported i.e. Rust, Golang)
* Container Images
  * Container image must implement Lambda Runtime API
  * ECS / Fargate preferred for running Docker images
* Example of Serverless CRON job - CloudWatch Events EventBridge triggers a Lambda every hour
  * No longer need to run a EC2 service constantly just for it to run the job every hour
* Pricing
  * Pay per request and compute time
  * Free tier 1,000,000 AWS Lambda requests and 400,000 GB of compute time  
    * $.20 per 1 million requests after 1M
    * 400,000GB-seconds = 400,000 seconds if 1GB of RAM, 3.2M if 128GB RAM
  * $1 for 600,000GB-seconds after
* Limits (**exam likes to test your knowledge on these!**)
  * Execution
    * Memory: 128MB 0 19GB
    * Max execution time: 900 seconds (15 minutes)
    * Env variables - 4KB
    * /tmp (disk capacity) 512MB to 10GB
    * Concurrency executions: 1000 (can be increased)
  * Deployment
    * Deployment size: 50MB
    * Size of uncompressed deployment: 250MB
    * /tmp to load other files at startup
    * Size of env vars: 4KB
  * Exam likes to test on "We need 30 minutes of run time" or "We need 30GB of RAM" etc. so you should know Lambda cannot work
* Concurrency
  * 1000 requests per region per account
  * Can set a "reserved concurrency" at the function level (which is a limit of how many can run at a time)
  * Each invocation over the concurrency limit will trigger a "Throttle"
  * Throttle Behavior
    * sync invoke: return ThrottleError - 429
    * async invoke: retry automatically then go to DLQ
      * will retry for up to 6 hours if receiving 429 or 500 errors
      * returns event to internal event queue
      * Each retry interval increases exponentially from 1 second after the first attempt to max of 5 minutes      
  * If you need more than the default 1000 invocations, open support ticket
  * If you don't set a reserved concurrency and one lambda function gets hit with 1000 invocations then all the other lambda functions will throttle
  * Cold Start
    * New instance - code is loaded and code outside the handler is run (init)
    * If init is large (code, dependencies, SDK) this process can take some time
    * First request served by new instances has higher latency than the rest
  * Provisioned Concurrency
    * Concurrency is allocated before the function is invoked and cold start never happens
    * Low latency
    * Application Auto Scaling can manage concurrency (schedule or target util)
  * Keep in mind that provisioned AND reserved concurrency will eat into your total concurrency per region. (if you set it to 300 for a specific function you will only ever have 700 left for other functions).
* Lambda SnapStart
  * Improves Lambda function performance up to 10x at no extra cost for Java, Python, .NET
  * Function is invoked from pre-initialized state (no function initialization from scratch)
  * When you publish new version Lambda inits function, takes snapshot of memory and disk state of initialized function and snapshot is cached
* Lambda@Edge & CloudFront Functions
  * Edge Function: code that you write and attach to a CloudFront distrubution
    * Allows you to run functions close to your users to reduce latency
  * Runs close to your users to minimize latency
  * Fully serverless, pay only for what you use
  * Can help with:
    * Website Security and Privacy
    * Dynamic Web Application at Edge
    * SEO
    * Intelligently Route Across Origins and Data Centers
    * Bot Mitigation at the Edge
    * A/B Testing
    * Real-time Image Transformation
    * User Authentication and Authorization
    * User Prioritization
    * User Tracking and Analytics
  * CloudFront Functions
    * Lightweight functions written in JS
    * high scale latency sensitive CSN customizations
    * Sub-ms startup times, millions of requests/sec
    * Used to change Viewer requests (after CF receives requests from a viewer) and responses (before CF receives request)
    * Native feature of CF (manage code entirely within CF)
    * max execution time < 1ms
  * Lambda@Edge
    * Written in NodeJS or Python
    * Scales to thousands of requests/second
    * max exeuction time 5-10 seconds
    * Used to change CF requests and responses
      * Viewer Request, Origin Request (before CF forwards request to origin), Origin Response (after CF receives response from origin), Viewer Response
    * Author function in on AWS Region, CF distributes to all it's locations
  * Use cases
    * Cloudfront Functions:
      * Cache key normalization (transform requests attributes - headers, cookies, query strings, URL to create a cache key
      * header manipulation - insert/modify.delete HTTP headers in request or response
      * URL rewrites or redirects
      * Request authentication and Authorization (create and validate a JWT) to allow/deny requests
    * Lambda@Edge
      * Longer execution time (several ms)
      * Adjustable CPU/memory
      * Your code depends on 3rd party libraries or other AWS services
      * Network access to use external services
      * File system access or access to the body of HTTP requests
* Lambda in VPC
  * By default, Lambda is not inside a VPC and does not have access to the resources in it
  * In order to access resources in yoru VPC, you must create an ENI inside the VPC
  * If you want to use an RDS Proxy with your Lambda (recommended as many Lambdas can overwhelm as RDS server) then the Lambda must be deployed in your VPC as RDS proxy is NEVER publicly accessible **remember for exam, important**
* Invoke Lambda from within RDS & Aurora
  * Allows you to process data events from inside the database
  * Supported by RDS for Postgres and Aurora MySQL
  * Must allow outbound traffic from DB instance to Lambda
  * DB instance must have required permissions to invoke Lambda function
* RDS Event Notifications
  * Notifications that tell you about DB instance itself (does not talk about data) i.e. created, stopped, started
  * Subscribe to event categories: DB instance, DB snapshot, DB Parameter Group, DB Security Group, RDS Proxy, Custom Engine Version
  * Near real time events (up to 5 minutes)
  * Send notification to SNS or subscribe to events using EventBridge

# DynamoDB
* NoSQL DB w/ transactions that is HA across multiple Az's
* Millions of request per second, trillons of rows, 100s of TB's of storage
* single-digit millisecond performance
* Integrated with IAM for security, authorization and administration
* auto scaling, always available
* Standard and Infrequent Access (IA) table classes
* Basics
  * Made of Tables
  * Each Table has a Primary Key that must be decided at creation time
  * Table can have infinite items (rows)
  * Each item has attributes (can be added to over time, can be null)
  * Max size of an item is 400KB
  * Data types supported:
    * ScalerTypes (String, Number, Binary, Bool, Null)
    * DocumentTypes (List, Map)
    * SetTypes (String Set, Number Set, Binary Set)
  * DynamoDB is a great choice for rapidly evolving schemas (so if you see rapidly evolving schemas on the exam, it's talking about DynamoDB)
* Read/Write Capacity Modes (Don't worry about HOW to calculate capacity, its not on this exam)
  * Provisioned Mode (default)
    * Specified read/writes /sec
    * Capacity needs to be planned beforehand
    * Pay for provisioned Read Capacity Unity (RCU) and Write Capacity UInits (WCU)
    * Can add auto-scaling RCU/WCU
  * On-Demand Mode
    * Read/writes scale automatically, no capacity planning needed
    * Pay for what you use but it's more $$$
    * Great for unpredictable workloads, steep sudden spikes or extremely low activity
* DynamoDB Accelerator (DAX)
  * Microseconds latency with in-memory cache
  * 5 minutes TTL on cache (default)
  * Doesn't require app logic modification
  * DAX vs Elasticache - DAX is for individual records vs Elasticache is more for storing things you already computed from DynamoDB
* Stream Processing (DynamoDB Streams)
  * Ordered stream of item-level modifications (create/update/delete) in a table
  * 24 Hours retention, Limited # of consumers, Processing using AWS Lamabda Triggers or DynamoDB Stream Kinesis adapter
  * Use cases:
    * Insert into derivitive tables
    * Implement cross-region repl
    * Invoke AWS Lambda on changes to DynamoDB table
    * Real time usage analytics
  * Can also use Kinesis Data Streams (newer)
    * 1 year retention
    * High # of consumers
    * Processing using lots of services..
* Global tables
  * Active-Active global tables - apps can read and write to table in any region
  * Must enable DynamoDB Streams as a pre-req
  * Low latency in multiple regions
* TTL - Automatically delete items after expiry timestamp
* Continuous Backups
  * Optional enabled for the last 35 days
  * PITR to any time in the window
  * recovery process make a new table
* On-demand backups
  * Full backups for long term retention until deleted
  * does effect performance or latency
  * Can be configured and managed in AWS Backup
  * recovery process create new table
* Export to S3 (must enable PITR)
  * Can export any point of time in the last 35 days
  * JSON or ION format
  * Doesn't consume read capacity
* Import from S3
  * Import CSV, DynamoDB, JSON or ION
  * Doesn't consume write capacity
  * Creates new table
  * Import errors logged in CloudWatch Logs

# API Gateway
* Supports WebSocket Protocol
* Handles API versioning (v1, v2..)
* Handle different environments (dev, test, prod)
* Handle security (Authentication and Authorization)
* Create PAI Keys, handle request throttling
* Swagger/Open API import to quickly define APIs
* Transform and validate requests and responses
* Generate SDK and API specifications
* Cache API responses
* Integrations:
  * Lambda (easy REST API)
  * HTTP such as on-prem or ALB
    * Why? Add rate limiting, caching, user auth, API keys, etc.
  * AWS Service (i.e. step function workflow, post message to SQS)
    * Why? Add auth, deploy duplicly, rate control
* Endpoints:
  * Edge-Optimized (default): For global clients
    * routed through cloudfront edge locations but API gateway still lives in only one region
  * Regional:
     * For client in same region, but can still manually combine with CloudFront
  * Private
     * Can only be access from your VPC using an ENI (using resource policy to define access)
* Security:
  * User Authentications through IAM Roles, Cognito, Custom Authorizer (your own logic)
  * Custom Domain Name HTTPS through ACM
    * If using Edge Optimized endpoint then cert must be in us-east-1
    * if using regional endpoint then cert must be in API Gateway region
    * Must setup CNAME or A-alias record in Route 53
* Default timeout of 29 seconds but can be changed

# AWS Step Functions
* Visual workflow for Lambda functions
* Can integrate with EC2, ECS, on-prem servers, API Gateway, SQS queues, etc.
* Possibility of implementing human approval feature
* Features: sequence, parallel, conditions, timeouts, error handling
   
# Amazon Cognito
* Gives users an identity to interact with web or mobile application
  * ** COMMON QUESTION ON EXAM is to ask how to store credentials for mobile users - do not say "store on mobile device", say "Cognito"**
* Cognito User Pools
  * Sign in functionality for app users using a serverless database
  * Simple user/password but can have MFA
  * Can also use Federated Identities (Google, FB, SAML, etc.)
  * Integrates with API Gateway & ALB
* Cognito Identity Pools (used to be called Federated Identity)
  * Provides AWS credentials to users so they can access AWS resources directly
  * Can integrate with a IdP
  * IAM policies applied to the credentials are defined in Cognito
  * Default IAM roles for guest users
  * Can set row level security in DynamoDB **Important to remember, frequently on exam**
* Cognito vs IAM: It's for "outside" users (web and mobile) so on the test looks for "hundreds of users" or "mobile users" or "authenticate with SAML" to know whether they want IAM or Cognito

# DocumentDB
* Is MongoDB
* store, query and index JSON data
* Similar "deployment concepts" as Aurora
* HA with rep across 3 AZ's
* storage auto grows in increments of 10GB
* Automatically scales to millions requests/sec
* On exam is you see "MongoDB" it's probably talking about DocumentDB, if you see "NoSQL" it's probably taling about DocumentDB or DynamoDB

# Neptune
* Fully managed graph database (extremely interconnected data like Wikipedia)
* HA across 3AZ's with up to 15 read replicas
* Store up to billions of relations and query the graph with milliseconds latency
* Streams
  * Real time ordered sequence of every change of graph data
  * Changes available immediately after writing
  * No duplicates, strict order
  * Steams data accessible in HTTP Rest API
  * Use cases:
    * Use notifications when certain changes are made
    * maintain graph data synchronized in another data store (S3, OpenSearch, ElastiCache)
    * Replicate data across region in Neptune

# Keyspaces
* Managed Apache Cassandra which is open source NoSQL distributed DB
* Serverless, Scalable, HA
* Scales based on application traffic
* Tables are replicated 3 times across multiple AZ's
* Uses Cassandra Query Language (CQL)
* Single-digit millisecond latency at any scale, 1000s of rquests per second
* Capacity: On-demand mode or provisions mode with auto-scaling (same as DynamoDB)
* Encryption, backup, PITR up to 35 days
* Use cases: store IoT data, time-series data
* **Just remember Keyspaces if `Apache Cassandra` is on test**

# Timestream
* Serverless time series database
* automatically scales to adjust capacity
* stores trillions of events per daya
* 1000s time faster & 1/10th the cost of relations databases for time related data
* Scheduled queries, multi-measure records, SQL compatibility
* Encryption in transit and at rest
* recent data kept in memory and historical data kept in cost-optimized storage
* Built-in time series analytics functions

# Athena
* Serverless query service to analyze data in S3
* Uses standard SQL language to query files (built on Presto)
* CSV, JSON, ORC, Avro, Parquet
* Price $5/TB scanned
* Commonly used with Amazon QuickSight
* Use cases: Analyze VPC flow logs, CloudTrail trails, ELB logs, etc.
* **Exam Q on analyzing data in S3 using serverless = Athena**
* Performance Improvements
  * Use columnar data type for cost savings (less scans)
    * Parquet or ORC is recommended
    * Huge performance improvement
    * Use Glue to convert your data to Parquet or ORC
  * Compress data for smaller retrievels
  * Partition datasets in S3 for easy querying on virtual columns
    * e.g. s3://athena-examples//flight/parquert/year=1991/month=1/day=1/
  * Use larger files (> 128 MB) to minimize overhead
* Federated Query
  * Can run SQL queries across relational, non-relational, and customer data sources (AWS or on-prem)
  * Uses Data Source Connectors that run on AWS Lambda to run Federated Queries (e.g. CloudWatch Logs, DynamoDB, RDS, and much more)
  * Can store results back to S3

# Redshift
* based on Postgres, but it's not OLTP, it's OLAP
* 10x better performance than other data warehouses
* Columnar storage of data & Parallel Query engine
* Provision cluster OR serverless cluster
* SQL interfaces for queries
* BI Tools like QuickSight or Tableua integrate with in
* vs Athena: Faster Queries, joins and aggregations thanks to indexes
* Cluster:
  * Leader node: For query planning, results aggregation
  * Computer node: for performance queries, send results to leader
  * Provisioned mode:
    * choose instance types in advances
    * Can reserve instances for cost savings
* Has Multi-AZ for some clusters
* Snapshots are PITR backups stored in S3
  * Incremental (only changes what is saved)
  * Can restore snapshot into a new cluster
  * Automated every 8 hours, every 5 GB or on schedule, set retention
  * Manual: snapshot is retained until deletion
  * You can configure Redshift to automatically copy snapshots (automated or manual) to another AWS region
* Loading data:
  * Large inserts are MUCH better
  * Kinesis Data Firehouse --> Redshift (through S3 copy)
  * S3 --> Redshift
    * Put data into S3 and then run copy command in redshift
    * Can route data through the internet (doesn't have enhanced VPC Routing) or through VPC (Enhanced Routing)
  * EC2 Instance --> Redshift
    * Should insert in batches, not RBAR as that's extremely inefficient for Redshift
* Redshift Spectrum
  * Launch query in redhsift which kicks off the query on many redshift spectrum worker nodes which queries data in S3 and sends the data back to redshift.
  * Allows you to query S3 without putting data into Redshift first
  
# OpenSearch (used to be called ElasticSearch)
* In DynamoDB, you can only query by primary key or indexes, OpenSearch searches any field, even partial matches
* Common to use OpenSearch as complement to another DB
* Managed cluster or serverless cluster
* Does not natively support SQL (but can be enabled via plugin)
* Ingest from Kinesis Data Firehose, AWS IoT, CloudWatch Logs
* Security through Cognito, IAM, KMS, TLS
* Comes with OpenSearch Dashboards for visualization
* So for instance your data goes into DynamoDB, DynamoDB Streams uses a Lambda to load just item data into OpenSearch, and then in open Search it searches for a specific item and then sends it back to DynamoDB to get the full item record

# EMR
* Elastic MapReduce
* Hadoop clusters (Big Data) to analyze and process vast amount of data
* clusters can be made of hundreds of EC2 instances
* EMR comes bundled with Big Data tools like Apache Spark, HBase, Presto...
* EMR takes care of all provisioning and configuration
* auto-scaling and integration with spot instances
* use cases: data processing, ML, web indexing, big data
* Node types
  * Master Node - Manages cluster, coordinates, manages health - long running
  * Core Node - Run tasks and store data - long running
  * Task Node (optional) just to run tasks - usually Spot
* Purchases options:
  * On-demand: reliable, predictable, won't be terminated
  * Reserved (min 1 year): cost savings (EMR will automatically use if avail)
  * Spot INstances: cheaper, can be terminated, less reliable
* Can have long running cluster or transient (temporary) cluster

# QuickSight
* Serverless ML powered BI service to create interactive dashboards
* per-session pricing
* Integrates with S3, Aurora, Athena, Redshift, S3...
* In-memory computation using SPICE (only works if you import data into QuickSight, not if you query a different DB)
* Enterprise edition: Column level security (CLS)
* Can use AWS Services, Third party datasources (on-prem, JIRA, salesforce) or imported data
* Define users (standard users) or groups (enterprise version)
  * QuickSight users, not IAM users!
* Dashboard is a read only snapshot of an analysis that you can share
  * Preserved the config of the analysises (filtering, parameters, controls, sort)
  * You can share the analysis or the dashboard with Users and Groups
  * To share a dashboard you must first publish it
  * Users how can see a dashboard can also see underlying data

# AWS Glue **Exams like to ask about Glue so this part is important**
* Serverless ETL service
* Can extract and convert data into parquet (columnar) format for Athena
  * S3 PUT notifications fire a lambda or EventBridge to kick off Glue job, Glue imports CSV from S3, converts to Parquet, saves to a different bucket which is analyzed by Athena **Important concept for exam**
* Glue Data Catalog - Uses Glue Data Crawler to inspect DB's (RDS, DynamoDB, etc.) and creates metadata databases/tables in the Glue Data Catalog which is used by other services (Glue (ETL), Athena, RedShift..)
* Glue Job Bookmarks - prevents re-processing old data
* Glue Datarew - clean and normalize data using pre-built transforms
* Glue Studio - new GUI to create, run and monitor ETL Glue jobs
* Glue Streaming ETL (built on Apache Spark) - Compatible with streaming like Kinesis Data Streams, Kafka, MSK

# AWS Lake Formation
* Data Lake - central place to have all your data for analytics purposes
* Fully managed service that makes it easy to setup data lake in days
* Discover, cleanse, transform and ingest data into your Data Lake
* Automates complex manual steps (collecting, cleansing, moving, cataloging data) and de-duplicate
* Combine structured and unstructed data
* Out of the box blueprints - S3, RDS, relations and NoSQL DB
* Fine grained access controlf ro applications (row and column level)
* Built on top of AWS Glue
* **Important for exam** - You can do column AND row level security in lake formation, which means any service which then pulls the data from your data lake will only see what you want it to see, you don't have to manage permissions in each service

# Amazon Managed Service for Apache Flink (previously Kinesis Data Analytics for Apache Flink)
* Flink (Java, Scala, or SQL) is is a framework for processing data streams
* Reads from Kinesis Data Streams or Amazon MSK (Apache Kafka)
  * CANNOT read from Firehose which is an **exam trick**
* Run any flink app on a managed cluster on AWS
  * Provisioned compute resources, parallel compute, auto scaling
  * Automated backups (checkpoints and snapshots)
  * Uses flink programming features to transform data
  
# Amazon MSK
* Alternative to Amazon Kinesis
* Fully managed Apache Kafka on AWS
  * Allow you to create, update and delete clusters
  * MSK creates & manages Kafka broker nodes & Zookeeper nodes
  * Deploys MSK cluster in your VPC
  * Automatic recovery from common Apache Kafka failures
  * Data is stored on EBS volumes for as long as you want
* MSK Serverless
  * Run Apache Kafka on MSK without managin capacity
  * MSK automatically provisions resources and scales compute & storage
* MSK vs Kinesis Data Streams:
  * Kinesis Data Streams: 
    * 1MB message size limit
    * Data Streams with shards
    * Shard Splitting & Merging
    * TLS in-flight encryption
    * KMS at-rest encryption
    * Only keep data for 1 year
  * MSK
    * 1MB default, configure for higher
    * Kafka Topics with Partitions
    * Can only add partition to a topic
    * PLAINTEXT or TLS In-flight encryption
    * KMS at rest encryption
    * Can keep data for as long as you want
* Consumers:
  * Amazon Managed Service for Apache Flink
  * Glue
  * Lambda
  * Applications running EC2 or ECS

# IOT Core
* Service that collects data from IOT devices

# Amazon Rekognition
* Uses AI to find objects, people, text, scenes in images and videos
* Can do user verification, people counting, content moderation, celebrities, object flagging (mountains, dogs, etc.)
* Content Moderation
  * Detect content that is inappropriate, unwanted or offensive
  * Can be used to moderate image content for a website
  * Set threshold confidence
  * Can manually review flagged items

# Amazon Transcribe
* Automatically convert speech to text
* Uses deep learning process called automatic speech reconginition (ASR) to recognize speech to text 
* Can automatically remove PII using redaction
* Supports automatic language identification for multi-lingual audio

# Amazon Polly
* Text to speech (many voices)
* customize the pronunciation of words with Pronunciation lexicons
  * e.g. If you want it to say "Amazon Web Services" everytime it reads "AWS" or how to say a certain word
* Upload lexicons and use them in SynthesizeSpeech operation
* Generate speech from plain text or from documents marked up with Speech Synthesis Markup Language (SSML) - this enables more customization
  * emphasizing specific words or phrases
  * Using phonetic pronunciation
  * including breathing sounds, whispering
  * using newscaster speaking style

# Amazon Translate
* Translates language from one language to another, allowing you to localize content

# Amazon Lex & Connect
* Lex
  * Same tech that powers alexa
  * Used for chatbots and call center bots
  * Recognizes the intent of text/callers
* Connect
  * receive calls, create contact flows, cloud-based virtual contact center
  * Can integrate with other CRM's
  * No upgront payments, 80% cheaper than traditional contant center solutions

# Amazon Comprehend
* Natural Language Processing (NLP)
* Fully managed serverless service
* Uses ML to find insights and relationships in text
  * Language of text
  * Extract key phrases, places, people, brands, events
  * Understand how positive or negative the text is
  * Automatically organizes collection of text files by topic
* Can be used to review customer emails for positive or negative experiances
* Create or group articles by topic

# Amazon Comprehend Medical
* Detects and returns useful information in unstructured clinical text
* Uses NLP to detect Protected Health Information (PHI)
* Can be used to categorize health information (prescriptions, client issues)

# Amazon SageMaker AI
* Full managed service for developers / data scientists to build ML models
* Makes it easy to label, build ML model, train, tune, apply the model, which is normally hard to do and has many different parts - all these parts are in SageMaker

# Amazon Kendra
* Fully managed document search service **the important part**
* Extracts answers from documents, builds Knowledge Index and then uses NLP with a search
  *  e.g. user can search "Where is IT support desk", it searches and returns "1st Floor"
* Learns from user interfactions/feedback to promot preferred results
* Ability to manually fine-tune search results (importance of data, freshness, custom)

# Amazon Personalize
* Build apps with real-time personalized recommendations
* Same tech used on Amazon.com
* Can make personalized product recommendations, customized direct marketing

# Amazon Textract
* Extract text, handwriting and data from any scanned documents using ML
* Read any type of document or image
* Extract data from forms and tables
* Good for healthcare, financial services, public sector (tax forms, ID documents, passports)

# CloudWatch
* Metrics
  * Up to 30 dimensions per metric
  * All metrics have timestamps
  * Can create custom CloudWatch Metric
  * Can stream metrics via Firehose
    * Can filter subset of metrics instead of all of them
* Logs
  * log group - arbitrary name, usually representing an app
  * log stream - instances within app, log files, containers
  * can define log expiration policies
  * logs are encrypted by default
  * can setup KMS with your own keys
  * Insights lets you write a query and select a timeframe and you can query your logs and see visualization.
    * Can add to dashboard
    * Can query multiple Log groups (even in different AWS accounts)
    * It's a query engine, not a real time engine
  * Can export to S3 but it can take up to 12 hours
  * Log subscriptions
    * real-time log events that sends to Kinesis Data Streams, Firehose or Lambda
    * Can use subscription filter to filter which logs are sent
    * By using this you can aggregate different logs from multiple regions or accounts into a single  destination (e.g. Kinesis Data Streams)
      * Need to setup IAM access policy in source and destination accounts for cross-account
* Log Tail allows you to see logs come in as they occur
* EC2 instances will not send logs to CloudWatch by default, you must use an agent
  * Ensure you have IAM permissions for the EC2 to send logs to CloudWatch
  * Can be used on-prem as well
* Agents
  * CloudWatch Logs Agent
    * Older - can only send to CloudWatch Logs
    * Unified
      * Can collect additional system-level metrics and at a more granular level
      * Centralized config using SSM Parameter Store
* Alarms
  * Can start, terminate, reboot or recover an EC2 instance
  * Trigger Auto-scaling action
  * Send notification to SNS (which could send to Lambda to do pretty much anything)
  * Alarms are single metric only
  * Composite alarms can monitor the states of multiple other alarms using AND and OR conditions
    * Helps to remove alarm noise by creating complex composite alarms
  * Alarms can be made off a metric filter
* Container Insights
  * collects metrics and logs from containers (ECS, Fargate, K8s on EC2)
  * In EKS and K8s, Insights uses a containerized version of CloudWatch Agent to discover containers
* Lambda Insights is a lambda layer which collects system-level metrics and diagnostic info (cold starts, worker shutdowns, etc.)
* Contributer Insights **See metrics about top-N contributors**
  * Analayzes log data and creates time series that displays contributor data
  * Works with any AWS generated logs
  * Helps you find bad hosts, heaviest netowrk users, etc.
* Application Insights
  * Provides automated dashboards that shwo potential problems with monitored apps
  * app runs on EC2 with select technologies only (.NET, Java, etc.)
  * links to other AWS resources (e.g. Lambda, EBS, RDS, etc.)
  * Shows issues with application related to some services
  * Uses SageMaker
  * All findings and alerts are send to EventBridge and SSM OpsCenter


# EventBridge (formerly CloudWatch Events)
* Can run Scheduled (cron jobs) or use Event Pattern which can react to a service doing something (eg. IAM root login)
* trigger Lambda, send SQS/SNS..
* AWS Services send to `Default Event Bus`, AWS SaaS Partners (Datadog, Zendesk) send to `Partner Event Bus` and your own Custom apps can write to the `Custom Event Bus`
* You can access event busses across accounts using resource based policies
* You can archive events (all/filter) sent to an event bus (indefinitely, set period)
* You can replay archived events
* Schema Registry
  * Can analyze the events in your bus and infer a schema
  * Allows you to generate code for application that follows the eventbridge bus schema
  * Can version schemas
* Resource based Policy
  * Manage permissions for specific Event Bus
  * allow/deny events from other AWS accounts or regions

# CloudTrail
* Provides governance, compliance and audit
* Enabled by default
* Get history of events/ API calls made within your AWS Account
* Trail can be applied to all regions (default) or single region
* Can put logs into CloudWatch or S3 (can do if you want to keep events for longer than 90 days)
* If a resource is deleted, investigate CloudTrail first!
* Logs Management Events by default, Data Events (S3 object activity, Lambda function) are not logged by default
* Insights:
  * Detects unusual activity such as hitting service limits, inaccurate resource provisioning, bursts of IAM actions, etc.)
  * CloudTrail Insights analyzes normal events to create a baseline
  * and then continuously analyzes write events to detect unusual patterns
* You can have CloudTrail send events to EventBridge which can then send notifications

# AWS Config
* Auditing and recording compliance of AWS resources
* Records configuration and changes over time
* You can receive alerts for changes
* Per-region service
* Can be aggregated across regions and accounts
* Can store data into S3
* Rules
  * Can use AWS managed config rules (over 75)
  * Make custom config rule
  * can be eval/triggered for each config change or at regular time intervals
* DOES NOT prevent actions from happenings (no deny), just reports
* No free tier, $.0003 per configuration item recorded per region, $.001 per config rule evaled per region (aka $$)
* Can show you compliance, configuration and CloudTrail API calls to a resource over time
* Remediations using SSM Automation Documents
  * AWS Managed or custom Automation Documents can be trigged to take remediation against non-compliant resources (e.g. remove unused IAM access keys).
    * Can make it activate a Lambda to do whatever

  * Can set Remediation retries (up to five times) if the resource is still non-compliant after auto remediation
* Examples:
  * Can send to EventBridge to trigger notifications to SNS when resources are non-compliant
  * Can send notifications directly to SNS (notifications can then be filtered and sent to admins)

# AWS Organizations
* Consolidated billing across all accounts
* Member accounts can only belong to one organization
* Shared reserved instances and Savings Plans discounts across accounts
* Can enabled cloudtrail on all accounts and send logs to central S3 account
* Establish cross account roles for admin purposes
* Service Control Policies
  * IAM policies applied to OU or Accounts to restrict Users and Roles
  * They do not apply to the management account
* Tag Policies
  * Standardizes tags across resources in AWS Organization
  * Ensure consistent tags, audit tagged resources, prevent non-compliant tagging on specified resources
  * Use EventBridge to monitor tags
  * **"How to ensure consistent tags account accounts" on test - think organizations**

* IAM Conditions
  * `aws:SourceIp` - restricts client IP <ul>from</ul> which API calls are being made
  * `aws:RequestedRegion` - restricts the region the API calls are made <ul>to</ul>
    * e.g. if you put eu-central-1, you won't be able to hit from eu-central-1
  * `ec2:ResourceTag` - restrict based on tags
    * e.g. can limit `StartInstance` to user tags with `aws:DevOps`
  * `aws:MultiFactorAuthPresent` - used to force MFA
    * e.g. can limit `StopInstance` only to users that have MFA'ed
  * `s3:ListBucket` is a bucket level permission so it needs `Resource:<bucket arn>`
  * `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` apply to objects IN the bucket so they need `Resource:<bucket arn>/*` **WILL PROBABLY BE ON TEST**
  * `aws:PrincipalOrgID` can be used to restrict access to accounts that are member of the AWS Organization

* IAM Roles vs Resource Based Policies
  * You can access a S3 cross account by assuming an IAM role OR by connecting directly to the bucket and giving access in the bucket policy.
  * The difference is that if you assume an IAM role, you give up the original permissions and take on the permissions assigned to the role. With resource based policies, the principal does not give up their permissions
  * Resource based policies should be used when for instance a user in account A needs to scan a DynamoDB table in Account A and then dump into S3 bucket in Account B.  If you used roles you may have to go back and forth assuming roles
  * More and more services are implementing resource based policies in AWS e.g. S3, SNS, SQS, etc.
  * Eventbridge rules will implement a resource based policy to connect to some things like S3, SQS, SNS, etc. but other things that don't support resource based policies it will use a IAM role.  
    * Some services (like Kinesis) DO support resource based roles but they are still using IAM roles. In order to figure out if it will use a resource policy or the IAM role you need to look at the EventBridge rule

* IAM Permission Boundaries
  * Supported for users and roles (not groups)
  * Advanced feature to use a managed policy to set the maximum permissions and IAM entity can get
  * You can set that e.g. a user only has access to S3 in the boundary. Then even if they give themselves `AdministratorFullAccess` they still will only be able to access S3.
  * This is helpful because you can set boundaries and then allow non-administrators (e.g. a project manager over some developers) to give access to things but they still can't grant more access than you want them to have. You could also allow e.g. developer to manage their own permissions
  * Can also be used to restrict a specific user instead of applying a SCP that applies to the whole account

* IAM Identity Center (used to be AWS Single Sign-On (SSO))
  * Can use for all AWS account in AWS Organizations, Business accounts (Microsoft 365, Salesforce, etc.), SAML2.0-enabled applications, EC2 Windows Instances
  * Identity providers
    * Built-in provider
    * 3rd party Active Directory, Okta, etc.
  * Can use Permission Sets to grant access to groups of users to different OU's within AWS accounts
  * Application Assignments allows user to access SAML 2.0 business apps
  * Attribute-Based Access Control (ABAC) - uses fine-grained permissions based on user attributes stored in IAM Identity Center Identity Store
    * example: title, locale
    * Define permissions once, then modify AWS access by changing attributes on the user

* AWS Directory Service - Provides a way to create Windows Active Directory in AWS
   * AWS Managed Microsoft AD
     * Create your own AD in AWS, manage users locally, supports MFA
     * Establish "trust" connections with your on-prem AD
   * AD Connector
     * Acts as a proxy back to your on-prem AD by using a Directory Gateway, supports MFA
     * All AD resources managed in on-prem AD
   * Simple AD
     * AD compatible managed directory on AWS
     * Cannot be joined with on-prem AD, standalone
       * Can by used for EC2 instances running Windows to join a domain
   * **Exam may ask you VERY high level questions on this stuff like "We need to proxy AD user" - use AD connector, "We need to manage users on cloud and use MFA" - Managed Microsoft AD, "Just need to have windows domain for EC2, no MFA" - Simple AD**
   * How to integrate IAM Identity center to AD
     * If using AWS Managed Microsoft then you just connect to that and that's it
     * If you have your own AD then either:
       * Setup AWS Managed Microsoft AD (if you want to manage AD users in the cloud) by setting up a two way trust between your AD and the AWS Managed Microsoft AD
       * Setup AD Connector to connect requests back to your AD (if you don't want to manage AD users on cloud)

* AWS Control Tower
  * Uses AWS Organizations  to secure and make compliant multi-account AWS environments
  * Benefits:
    * Automate setup of environments in a few clicks
    * Automate ongoing policy management using guardrails
    * Detect policy violations and remediate them
    * Monitor compliance through an interactive dashboard
  * Guardrails:
    * Preventive Guardrail - uses SCP's (e.g. restrict regions to only specific ones account all accounts)
    * Detective Guardrail - uses AWS Config to find non-compliance (i.e. identify untagged resources)
      * Can fire SNS to notify or invoke Lambda which could then go remediate

# KMS
* Can auto key usage through CloudTrail ** may be on exam**
* Two methods of Keys
  * Symmetric (AES-256)
    * Never get access to he KMS key unencrypted (you must call KMS API to use)
  * Asymmetric (RSA & ECC key pairs)
    * Public (encrypt) and Private Key (Decrypt) pair
    * Used for Encrypt/Decrypt or Sign/Verify operations
    * Public Key is downloadable but cannot access the private key unencrypted
    * Use for encryption outside of AWS by users who can't call KMS API
* Types of Keys
  * AWS Owned Keys (free): SSE-S3, SSE-SQS, SSE-DDB (default key)
  * AWS Managed Keys (free): aws/service-name, example: aws/rds or aws/ebs). Created to encrypt objects in services by default
  * Customer Managed Keys created in KMS: $1/month
  * Customer Managed Keys imported into KMS: $1/month
  * + pay for API to KMS ($.03 / 10000 calls)
* Automation Key rotation
  * AWS-managed KMS key: automatic every 1 year
  * Customer-managed KMS Keys(must be enabled): automatic and on-demand
  * Imported KMS Keys: only manual rotation possible using alias
* A KMS key cannot live in two regions so if you take an encrypted ebs volume, snapshot it into a S3 bucket (which uses the same KMS key), and then that bucket replicated to another region, KMS will re-encrypt with the new key in the new region, and then when the snapshot is restored in the new region it will be using the same new key.
* Key Policies
  * Similar to S3 bucket policies except you cannot control access to the key without the Key Policy
  * Default Key Policy:
    * Created if you don't provide specific KMS Key Policy
    * Complete access to the key to the root user = entire AWS account
  * Custom KMS Policy
    * Define users/roles that can access the KMS key
    * Define who can admin the key
    * Useful for cross account access of the key
* Copying snapshots across accounts:
  * You make a snapshot using KMS and attached a Key Policy to authorize cross-account access, then share the encrypted snapshot.  In the target account you create a copy of the snapshot, encrypt it with a CMK in the target account and then create volume from the snapshot.
* Multi-region Key
  * Key is created in one region and sync'ed to other regions
  * Use key id, key material, automatic rotation
  * Encrypt in one region and decrypt in other regions
  * They are NOT global (they are primary + replicas)
  * each multi-region key is managed independently with it's own key policy
  * Not recommended to use multi-region keys due to cost, compliance and management of replicating keys
  * unless using for specific use cases like:
    * Global client side encryption
    * encryption on Global DynamoDB
    * Global Aurora
  * For instance you could have fields in DDB that are encrypted so that only the client can access them, not the DBA, and then that DDB table is a global table accessible from many regions
* S3 Replication with Encryption
  * Unencrypted objects and objects encrypted with SSE-S3 are replicated by default
  * Objects encrypted with SSE-C can be replicated
  * For objects encrypted with SSE-KMS you need to enable the option:
    * Specify which KMS Key to encrypt objects with in target bucket
    * Adapt KMS Key Policy for target key
    * Create IAM Role is kms:Decrypt for the source KMS Key and kms:Encrypt for the target KMS Key
    * You may get KMS throttling errors if you have a lot of objects to replicate, in which case you can ask for a service quote increase
  * If you use multi-region keys, S3 will still treat them as single region and decrypt/re-encrypt when replicating
* KMS Encrypted AMI Sharing **could be on test**
  * AMI in Source account is encrypted with KMS key in source account
  * modify image attribute to add `Launch Permission` which corresponds to the specified target AWS Account
  * Must share the KMS Keys used to encrypt the snapshot the AMI references with target account/IAM role
  * In target account, create IAM Role/User in target account must have permissions to `DescribeKey`, `ReEncrypt*`, `CreateGrant`, `Decrypt`
  * When launching an EC2 instance from the AMI, optionally the target account can specify a new KMS key on it's own account to re-encrypt volumes

# SSM Parameter Store
* Serverless and scalable secure storage for config and secrets using KMS with version tracking
* Notifications with EventBridge and integrates with CloudFormation
* Can store params with a hierarchy
* Tiers -  
  * Standard - Free
  * Advanced - $.05 per advanced param/month
    * more parameters
    * bigger size of parameters
    * use parameter policies - e.g. assign TTL to expire/delete a param or send notifications to EventBridge on change or other details

# AWS Secrets Manager
* Newer service that lets you rotate secrets
* Secrets are encrypted using KMS
* **If you ever see "Secrets" and "RDS" on exam think "Secrets Manager"**
* Secrets manager can replication secrets from one region to another and then promote a read replica secret to standalone

# AWS Certificate Manager (ACM)
* Private and public certificates
* Free public TLS certificates
* Automatic TLS cert renewal
* Can use with ELB's, CloudFront, API Gateway 
* CANNOT use with EC2 instances (certs can't be extracted)
* Takes a few hours to verify a certificate (DNS or Email validation)
* Public cert will be enrolled for automatic renewal 60 days before expiry
* You can import a certificate created outside ACM, but these do not automatically renew, you must re-import new
  * ACM sends daily expiration events starting 45 (configurable) data prior to expiration to EventBridge
  * AWS Config has a managed rile `acm-certificate-expiration-check` to check for expiring certificates
* If using with API Gateway the certificate must be:
  * in the same region as CloudFront for edge-optimized
  * in the same region as the API Gateway for Regional


# Web Application Firewall (WAF)
* Protects against Layer 7 exploits (HTTP)
* Deploy on:
  * ALB
  * API Gateway
  * CloudFront
  * AppSync GraphQL API
  * Cognito User Pool
* **Common Exam trick is to try to get you to deploy to a NLB but that's not possible since that is lower layer**
* Define Web ACL (Web Access Control List) Rules:
  * IP Set - Up to 10k IP addresses, can use multiple rules for more
  * HTTP headers, HTTP body, URI strings (SQL injection and Cross-Site Scripting XSS)
  * Size contraints, geo-match (block countries)
  * Rate based rules (to count occurrences of events) - for DDoS protection
* Web ACL's are regional except for CloudFront (Global)
* A rule group is a reusable set of rules you can add to web ACL
  * Can be set to eval in specific order
  * Can be set to allow/block for request that don't match rules
* Fixed IP while using WAF with a Load Balancer 
  * Since you can't use a NLB with a Fixed IP, you can use Global Accelerator to get a Fixed IP and WAF on ALB
  
# AWS Shield
* Standard - Free and provides protection from SYN/UDP floods, Reflection attacks and other layer 3/layer 4 attacks
* Advanced - Optional DDoS mitigation service ($3,000 per month per organization)
  * Protects again more sophisticated attacks on EC2, ELB, CF, Global Accelerator & Route 53
  * 24/7 AWS DDoS response team (DRP)
  * Protect again higher fees due to DDoS
  * automatically creates, evals and deploys WAF layer 7 rules

# AWS Firewall Manager
* Manage rules in all AWS Org accounts
* Security policy: groups of security rules for:
  * WAF
  * Shield Advanced
  * Security Groups for EC2, ALB and ENI
  * Network Firewall (VPC Level)
  * Route 53 Resolver DNS Firewall
  * Policies are at region level
* Rules are applied to new resources as they are created for future accounts in your Org
* Price ($100) per policy
* WAF vs Firewall Manager vs Shield
  * Web ACL's defined in WAF
  * For granular protection, WAF is your choice
  * For WAF across multiple accounts, accelerate WAF config, automate protection of new resources, use Firewall Manager with WAF
  * Shield is for additional feature on top of WAF for dedicated support and advanced reporting
  * Shield Advanced to help against frequent DDoS

# DDoS Best Practices (grouped by AWS into BP# resources)
* BPs
  * BP1 - CloudFront or Global Accelerator
  * BP2 - AWS WAF
  * BP3 - Route 53
  * BP4 - API Gateway
  * BP5 - VPC rules
  * BP6 - ELB
  * BP7 - ASG
  * BP8 - Regions
* Edge Location Mitigation (BP1, BP3) - Use Edge to distribute
* Infrastructure layer defense (BP1, BP3, BP6) - provides layers to protection before it even gets to the backend
* EC2 Auto Scaling (BP7) - Scale with sudden traffic increase
* Load Balancing (BP6) - Distribute and scale with sudden traffic
* Detect and filter malicious web requests (BP1, BP2) 
  * CF blocks specific geographies, distributes
  * WAF blocks IP's and filters requests
* Shield Advanced (BP1, BP2, BP6) - automatic deploys WAF rules for layer 7 attacks
* Obfuscate AWS Resources (BP1, BP4, BP6) - Hide backend resources
* Security Groups and Network ACL's (BP5) 
  * Security Groups and NACLs filter traffic by specific IP at subnet/ENI level
  * Elastic IP protected by AWS Shield Advanced
* Protect API endpoints (BP4) - header filter, API keys, hides backend Lambda/EC2, edge-optimized or regional CF mode

# AWS GuardDuty
* Uses ML to search logs for unusual things
  * CloudTrail Event Logs - unusual API calls, unauthorized deployments
  * CloudTrail Management Events - create VPC subnet, create trail
  * CloudTrail S3 Data Events - get object, list object, delete object
  * VPC Flow Logs - unusual internal traffic, IP addresses
  * DNS Logs - compromised EC2 instances sending encoded data within DNS queries
  * Optional Features - EKA Audit Logs, RDS/Aurora, EBS, Lambda, S3 Data Events, etc.
* Can notify via EventBridge (which can target Lambda or SNS)
* **May be on exam** has rules to protect against Cryptocurrency attacks

# Amazon Inspector
* Automated Security Assessments
  * For EC2 it uses the AWS System Manager (SSM) agent which can analyze unintended network accessibility and OS vulnerabilities
  * For container images push to ECR - Analyzes container images as they are pushed
  * For Lambda Functions - Identifies software vulnerabilities in function code and package dependencies as they are deployed
* Reports and integrates into AWS Security Hub
* Send findings to EventBridge
* Uses a CVE database for vulnerability checks
* A risk score is associated with all vulnerabilities for prioritization

# Amazon Macie - Data security and privacy service (PII)
* Uses ML learning and pattern matching to discover and protect sensitive data in AWS
* Can analyze a S3 bucket and then notify to EventBridge









# Disaster Recovery in AWS
* RPO - The maximum acceptable amount of data loss measured in time
* RTO - The maximum acceptable downtime after a disaster
* Disaster Recovery Strategies: The following are RPO and RTO decreases but money increases
  * Backup And Restore - Cheap but high/long RPO
    * Using AWS Snow or AWS Storage Gateway to copy to cloud
    * Regular snapshots of RDS/RedShift/EBS 
  * Pilot Light
    * Small version of the app is always running in the cloud
    * Faster than Backup and Restore as critical systems (e.g. RDS is already sync'ed and up) are already up (e.g. just have to turn on EC2/ECS instances and connect to RDS, change Route 53 records and you're up
  * Warm standby
    * Full system is up but at minimum size
    * RDS is up and synced, EC2/ECS systems are up but set low with auto scaling
    * ELB running to accept records
    * Just update Route53 and the app will scale out to handle load
  * Multi-Site/Host Site Approach
    * Full production scale running and Active/Active between data centers so if one fails the other just picks up all the traffic
  * All AWS Multi-Region
    * Use Aurora Global and full production running in multiple regions
* DR Tips:
  * Backup
    * EBS Snapshots, RDS automated backups
    * Regular pushes to S3
    * on-premise use AWS Snow or Storage Gateway
  * High Availability
    * Use Route 53 to migrate DNS over from Region to Region
    * RDS Multi-AZ, ElastiCache Multi-AZ, EFS, S3
    * Site to Site VPN as a recovery from Direct Connect
  * Replication
    * RDS Replication (Cross Region), AWS Aurora + Global DB
    * Storage Gateway
    * DB repl from on-prem to RDS
  * Automation
    * CloudFormation/Elastic Beanstalk to re-create a whole new environment
    * Recover/Reboot EC2 instances with CloudWatch if alarms fail
    * AWS Lambda function for customized automations
  * Chaos
    * Netflix has a "simian-army" randomly terminating EC2

# Database Migration Service (DMS)
* Source DB stays online during migration
* Supports
  * Homogeneous migrations e.g. Oracle to Oracle
  * Heterogenous migrations e.g. SQL Server to Aurora
* Continuous Data Replication using CDC
* Must have a DMS (EC2) instance to perform repl tasks
* AWS Schema Conversion Tool (SCT)
  * Converts database schema from one engine to another (e.g. Oracle --> MySQL)
  * Do not need to use if migrating same DB engine (on-prem postgres --> RDS postgres, it's not needed as it's same engine, just different platform) **part you need to know for the test**
* If you want to have continuous repl of a on-prem DB to a different engine on AWS
  * Implement a server on-prem with AWS SCT installed which syncs schema to the RDS
  * then implement DMS to sync data from on-prem to RDS
* Multi-AZ DMS deployment
  * DMS instances in different regions which DMS maintains and synchronizes in case of AZ failure
  * Used for Data Redundancy, eliminate I/O freezes, minimize latency spikes
* RDS and Aurora MySQL Migrations
  * RDS to Aurora Migration
    * Option 1: DB Snapshot from RDS restored as Aurora DB (downtime)
    * Option 2: Create Aurora RR from RDS and promote to it's own cluster when repl lag is 0 ($$)
  * External to Aurora 
    * MySQL
      * Option 1: Percona XtraBackup to create file backup --> S3 then restore
      * Option 2: Create Aurora MySQL DB and use mysqldump to migrate into Aurora (slower)
    * PostgreSQL
      * Create backup --> S3 then import with aws_S3 Aurora extension
  * Use DMS if both databases are up and running

# AWS On-prem strategy **Just need to know things related to on-prem for test**
* Can download Amazon Linux 2 AMI as a VM (.iso)
* VM Import/Export
  * Migrate existing applications into EC2
  * Create DR repo strategy for on-prem VM's
  * Can export back the VM's from EC2 to on-prem
* AWS Application Discovery Service
  * Gather info about on-prem servers to plan migration
  * Server util and dependency mappings
  * Track with AWS Migration Hub
* DMS
* AWS Server Migration Service (SMS)
  * Incremental repl of on-prem live server to AWS

# AWS Backup
* Fully managed service
* No need to create custom scripts and manual processes
* Supports cross-region AND cross-account
* EC2/EBS, S3, RDS, Aurora, DynamoDB, Document DB, Neptune, EFS, FSx, Storage Gateway and more
* Supports PITR for supported services
* On-demand and scheduled backups
* Tag based backup policies
* Create backup policies known as Backup Plans
  * Backup frequency (every 12 hours, daily, weekly, monthly, cron expression)
  * Backup window
  * Transaction to Cold Storage (Never, Days, Weeks, Months, Years)
  * Retention Period (Always, , Days, Weeks, Months, Years)
* You set up the service, select resources to backup and it backs up to S3
* Backup Vault Lock Policy
  * Enforce a WORM (Write Once Read Many)
  * Protects against inadvertent or malicious delete operations
  * Updates that shorten or alter retention periods
  * Even root user cannot delete backups when enabled!
* You can backup specific resources or all resources (and then filter by tags if desired)

# AWS Application Discovery Service
* Used to plan migration project from on-prem --> cloud
* Agentless Discovery
  * VM inventory, config, performance history such as CPU, memory, disk usage
* Agent-base 
  * System config, system performance, running processes, details of network connections between systems
* Resulting data can be viewed within AWS Migration Hub

# AWS Application Migration Service (MGN)
* Lift-and-shift solution to convert physical, virtual and cloud-based servers to run natively on AWS
* Install agent on-prem/any cloud and then it performans continuous replication from your apps and disks to EC2 instances and EBS volumes as a staging env, then you cut over and make the staging env production
* Minimal downtime, reduced costs

# Transferring large amounts of data into AWS
* **On the test you may be asked how to transfer data the most reliably, the fastest, or the most quickly to setup:**
* Site-to-Site VPN - can setup very quickly but transfer is the slowest (can take months for large data)
* Direct Connect - Setup takes 1 month but then faster connection (few weeks compared to months of Site-to-site)
* Snowball - 1 week to order, receive device, load it, send back
* On-going repl: site-to-site VPN or DX with DMS or DataSync

# VMWare Cloud on AWS
* If you use VMWare Cloud to manage VM's you migrate to VMWare Cloud on AWS to keep using VMWare Cloud
* You can run production workloads across VMWare vSphere based private, public and hybrid cloud environments
* Has disaster recovery strategy
* Once VMWare Cloud on AWS is in place, you can take advantage of other AWS resources

# Event processing in AWS
* SQS --> Lambda - try to send, SQS retries, set up DLQ in SQS to remove after x retries since it will infinitely loop
* SQS FIFO --> Lambda - try to send, SQS retries, set up DLQ in SQS to remove after x retries since it will block other messages (since it's in order)
* SNS --> Lambda --> SQS - SNS sends async to Lambda, if lambda fails it retires, if x fails Lambda sends to SQS DLQ
* Fan out - Instead of sending the same message to three SQS queues (which is subject to problems), send to SNS and that can send to multiple SQS
* S3 Event Notifications - Send to SNS, SQS, Lambda on S3 Event (e.g. use Lambda to generate thumbnails) - set up per event
* S3 Event Notifications to EventBridge - ALL events are sent - can filter with json rules, multiple destinations (FireHose, Kinesis Streams, etc), archive, replay, etc.
* EventBridge Intercept API Calls - When API Call occurs CloudTrail triggers event to EventBridge which can send notification
* AWS Service Integration with Kinesis Data Streams - API Gateway --> Kinesis Data Streams --> FireHose --> S3

# Caching Strategies on AWS
* You can cache on the edge in CloudFront, in the region with API Gateway, or back in your app logic with RDS/Redis (for example) - for dynamic content
* You can cache on the edge in CloudFront from your S3 bucket - for static content
* The farther in you go, the more the cost goes up, the more latency to the user
* So you need to decide the best aching, TTL, Network, Computation, Cost and Latency for your needs
* Basically there is a lot of ways to do caching on AWS but it's up to the scenario for what is going to be best use

# Blocking IP Address in AWS
* With Public Subnets, the NACL is your first line of defense, but you can optionally run a firewall on your EC2 instance (but this will incur CPU cost on the instance(s))
* With Public Subnet with an ALB/NLB and EC2 is in private subnet - You have NACL on Public Subnet, filtering on the ALB/NLB, and then optional firewall on EC2
  * You can also set up WAF on the ALB to extra filtering
* For CloudFront --> Public Subnet w/ ALB --> Private Subnet with EC2 - Set up WAF on CloudFront, Geo Restrictions can also be on CloudFront but NACL's *CANNOT* be used because CloudFront sends all the traffic with the public IP's into your ALB, so you need to setup security on the ALB instead of NACL's

# High Performance Computing (HPC)
* Cloud is great for HPC because you can create a large number of resources quickly and speed up time to results by adding more, then only pay for what you have used
* EC2 - Optimize for CPU, memory, use clusters in same rack, same AZ and spot instances for cheaper
* Networking 
  * EC2 Enhanced Networking (SR-IOV)
    * Higher bandwidth, higher packets per second (PPS), lower latency
    * Option 1 - Elastic Network Adapter (ENA) up to 100 Gbps
    * Option 2 - Intel 82599 VF up to 10 Gbps (LEGACY)
  * Elastic Fabric Adapter (EFA)
    * Improved ENA for HPC, only works with Linux
    * Great for inter-node communications, tightly coupled workloads
    * Leverages Message Passing Interface (MPI) standard
    * Bypasses underlying Linux OS to provide low-latency, reliable transport
  * **It's quite common on exam to need to distinguish between ENI, ENA, EFA**
* Review storage types (EBS, Instance Store, EFS, etc.) (FSx for Lustre is built for HPC!)
* Automation and Orchestration
  * AWS Batch
    * multi-node parallel jobs which enable you to run single jobs that span multiple EC2 instances
    * Easily schedule jobs and launch EC2 instances accordingly
  * AWS ParallelCluster
    * Open-source cluster management tool to deploy HPC on AWS
    * Configure with text files
    * Automate creation VPC, Subnet, cluster type and instance types
    * Ability to enable EFA on the cluster (improves network performance) **Might be on the exam for what it does it do to enable EFA in the text files - it improves network performance**

# EC2 High Availability
* This section assumes that app cannot be spread across multiple instances (in which case a ALB would be used)
* Elastic IP Address is attached to Public EC2 instance, cloudwatch event/alarm is set up to execute a lambda that can bring up another instance and move the elastic IP
* We can also use a ASG and set it min 1, max 1 desired 1 in >= 2 AZ's and the EC2 instance has EC2 User Data (start script) that will attach the elastic IP (instance role must have allow API calls to attach elastic IP )
* If the data is needed from the EBS volume, we can tell the ASG terminate lifecycle hook to tag a EBS snapshot + tags, the ASG will bring up a new instance, then the EBS volume is created from the snapshot and attached to the replacement instance

# CloudFormation
* If you use a service role for CloudFormation, it will use that for permissions instead of your own permissions. However in order to grant service role permissions to CloudFormation, the user needs to have the "iam:PassRole" permission, otherwise they cannot assign a role to CF

# SES
* Send and receive email for transactional, marketing and bulk email
* Can use DomainKeys Identified Mail (DKMI) and Sender Policy Framework (SPF)
* Flexible IP deployment: shared, dedicated and customer-owned IPs
* Send emails using your application via AWS Console, APIs or SMTP
* Provides stats such as email deliveries, bounces, feedback loop results, email open

# Amazon Pinpoint
* Inbound/outbound marketing communications service
* Supports email, SMS, push, voice, and in-app messaging
* Possibility to receive replies and process through SNS, FireHose, CloudWatch Logs
* Scales to billions of messages per day
* vs SNS & SES
  * You have to manage each message audience, content and delivery schedule in SNS & SES
  * In Pinpoint, you create message templates, delivery schedules, highly-targeted segments and full campaigns

* SSM Session Manager
* Instance access
  * Can secure shell into EC2 and on-prem
  * Port 22, SSH access, bastion hosts or SSH keys are not needed
  * Supports Linux, macOS and Windows
  * Send session log data to S3 or CloudWatch Logs
  * Must put IAM role on the instance that allows SSM
  * Other methods of connection: Open port 22 for SSH (need SSH keys) or use EC2 Instance Connect (no keys needed but still open port 22)
* Run command
  * Can run a command (script) against multiple instances (using resource groups) without need for SSH
  * Command output can be shown in AWS Console, Sent to S3 bucket or CloudWatch Logs
  * Send notifications to SNS about command status (In progress, Success, Failed)
  * Integrated with IAM & CloudTrail
  * Can be invoked by EventBridge
* Patch Manager
  * Automates patching of instances (OS updates, application updates, security updates)
  * Supports EC2 and on-prem
  * Linux, macOS, Windows
  * patch on-demand or on schedule
  * Scan instances and generate patch compliance report
* Maintenance Windows - Set up schedules to perform patching, install software, reboot instance, etc.
* Automation
  * Create Automation runbooks which are SSM Document to define actions preformed on your EC2 instance or AWS resources
  * Can be triggered using AWS Console, AWS CLI, EventBridge, on maintenance windows schedule or by AWS Config rule remediation

# Cost Explorer
* Can see breakdown of usage by hour in relation to cost
* Get recommendations for savings plans with reserved instances
* Forecast usage for up to 12 months

# AWS Cost Anomaly Detection
* Continuously monitors cost and usage using ML to detect unusual spends
* Learns unique, historic spend patterns to detect one-time cost spikes and/or continuous cost increases (no need to set thresholds)
* Monitors AWS services, member accounts, cost allocation tags or cost categories
* Sends you anomaly detection report with root cause analysis
* Get notified with individual alerts or daily/weekly summary (using SNS)

# AWS Outposts
* For companies that run Hybrid cloud, they have to have the skills to manage on-prem and cloud infra
* In order to alleviate this, AWS Outposts is a server rack that can be brought to the on-prem location to leverage AWS services in your on-prem
* You are responsible for physical security
* Benefits:
  * Low-latency access to on-prem systems
  * Local data processing
  * Data residency
  * Easier migration from on-prem to the cloud
  * Fully managed service
* Services that work on Outposts
  * EC2
  * EBS
  * S3
  * EKS
  * ECS
  * RDS
  * EMR

# AWS Batch
* Batch processing (job with start and end, not continuous) at any scale
* Batch will dynamically launch EC2 instances or spot instance
* AWS Batch provisions the right amount of compute/memory
* batch jobs defined as Docker images and run on ECS
* Batch vs Lambda
  * Lambda
    * Time limit
    * limited disk space
    * serverless
  * Batch
    * No time limit, any runtime as Docker images
    * Relies on EC2 (can be managed by AWS)
    * Relies on EBS/instance store for disk space

# AWS AppFlow
* Allows transfer of data from SaaS and AWS
* Sources: Salesforce, SAP, Zendesk, Slack and ServiceNow
* Destinations: AWS services like S3, RedShift, or non-AWS like SnowFlake or SalesForce
* Frequency: on a schedule, in response to events, on-demand
* Data Transformation: filtering and validation
* Encrypted: over public internet or privately over AWS PrivateLink
* Don't spend time writing integration and leverage APIs immediately

# AWS Amplify
* Set of tools to help you develop and deploy scalable full stack web and mobile applications
* Authentication, Storage, API (RES, GraphQL), CI/CD, PubSub, Analytics, AI/ML Predictions, Monitoring
* Connect your source code from GitHub, AWS CodeCommit, BitBucket, GitLab, or upload directly
* Think of Amplify has the Elastic Beanstalk for Mobile and Web applications - a one stop shop

# AWS Instance Scheduler
* Deployed through CloudFormation (it's not a service, it's a CloudFormation template that you find on AWS website)
  * Google "AWS Instance Schedule" for the template
* Automatically stop/start AWS services to reduce costs (up to 70%)
* Supports: EC2 instances, EC2 Auto Scaling Groups, RDS instances
* Can do all sorts of stuff like take snapshot of RDS before shutting down, tag items, only during maintenance windows, etc.
* Schedules are managed in DynamoDB Tables
* Resource tags and Lambda to start/stop instances
* Supports cross-account and cross-region resources

# Well Architected Framework
* Stop guessing your capacity needs
* Test systems at production scale
* Automate to make architectural experimentation easier
* Allow for evolutionary architectures
  * Design based on changing requirements
* Drive architectures using data
* Improve through game days
  * Simulate applications for flash days
* Pillars - not something to balance or trade-offs, they're synergy
  * Operational Excellence
  * Security
  * Reliability
  * Performance Efficiency
  * Cost Optimization
  * Sustainability
* AWS Well-Architected Tool
  * Free tool to review your architectures against the 6 pillars Well-Architected Framework and adopt architectural best practices
  * You answers a bunch of questions about how you want to run your business and it provides recommendations at different priority levels

# Trusted Advisor
* No need to install anything, high level AWS account assessment
* Analyze your AWS account(s) and provides recommendations on 6 categories
  * Cost optimization
  * performance
  * Security
  * Fault tolerance
  * Service limits
  * Operational Excellence
* Business & Enterprise Support Plan
  * Full Set of Checks
  * Programmatic Access using AWS Support API


# More information about solution architecture (resources to use when actually doing solution architecture)
* These are some guides on how to setup architecture for different solutions, you can search for what you are trying to do and find solution instructions, diagrams and even cloudformation templates
* [https://aws.amazon.com/architecture](https://aws.amazon.com/architecture)
* [https://aws.amazon.com/solutions](https://aws.amazon.com/solutions)
