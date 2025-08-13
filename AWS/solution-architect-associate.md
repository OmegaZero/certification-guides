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
    * Repl lag is 1s cross-region (exam q about 1 second cross region, it means probably asking about global aurora)
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
    * Redis Auth
      * Supports SSL
      * Uses user/pass
    * Memcached
      * Supports SASL based auth
  * Redis has sorted sets to allow for data to be sorted automatically (i.e. gaming leaderboard)

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
