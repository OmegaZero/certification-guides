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

