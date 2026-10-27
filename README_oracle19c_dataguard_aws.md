# Oracle Database 19c Data Guard on AWS — Build, Disaster Recovery, and Operations

This repository documents the end-to-end Oracle Database 19c Data Guard project on AWS: infrastructure provisioning, Oracle database preparation, RMAN physical-standby creation, redo transport and apply, Data Guard Broker, role-aware client connectivity, backup/recovery, controlled switchover and failover, Flashback-based reinstate, Snapshot Standby, Active Data Guard validation, and operational checks.

> **Accuracy and reproducibility note**
>
> This README combines project details and successful outcomes recorded during implementation with reproducible command templates for the steps. The primary/standby identities, private IPs, region, architecture, Data Guard tests, and recorded final health checks reflect the project. Some one-off AWS CLI, Terraform, Oracle installation, and configuration commands were not retained verbatim in the conversation. Those sections are marked as examples/templates where appropriate. Replace placeholders with values from the actual AWS account and Oracle homes before running commands. Never commit credentials, private keys, wallet contents, or secret-bearing files.

## 1. Project overview

### Objectives

- Provision an isolated AWS environment for Oracle Database 19c using Terraform.
- Use Red Hat Enterprise Linux 8 and encrypted EBS storage.
- Install and configure Oracle Database 19c, including Multitenant capabilities.
- Configure a primary database and physical standby across separate Availability Zones.
- Create the physical standby using RMAN `DUPLICATE FROM ACTIVE DATABASE`.
- Configure redo transport, standby redo logs, managed recovery, and Data Guard Broker.
- Validate transport/apply status, archive gaps, and lag.
- Configure role-aware client connectivity and the `ORCLCDG_HA` service.
- Test planned switchovers, controlled failover, Flashback Database recovery, former-primary rebuild/reinstate, Snapshot Standby, and Active Data Guard.
- Establish RMAN backup and recovery validation.
- Record patching and upgrade work separately from completed milestones.

### Confirmed architecture

| Component | Project value |
|---|---|
| AWS Region | `us-east-1` |
| Operating system | Red Hat Enterprise Linux 8.10, x86-64 |
| Oracle version | Oracle Database 19c |
| Primary host | DB01 |
| Primary private IP | `10.20.2.4` |
| Primary `DB_UNIQUE_NAME` | `ORCLCDG` |
| Standby host | DB02 |
| Standby private IP | `10.20.3.143` |
| Standby `DB_UNIQUE_NAME` | `ORCLCDG_STB` |
| Standby type | Physical standby |
| Infrastructure as Code | Terraform |
| Storage | Encrypted EBS volumes; persistent Oracle storage |
| Network | VPC, private networking, security groups, IAM, and supporting AWS networking |
| Redo transport | Asynchronous (`ASYNC`) |
| Data Guard management | Data Guard Broker (`DGMGRL`) |
| HA client service | `ORCLCDG_HA` |
| Original primary at final validation | `ORCLCDG` / DB01 |
| Original standby at final validation | `ORCLCDG_STB` / DB02 |

> **Important:** Role is dynamic. `ORCLCDG` and `ORCLCDG_STB` are database identities, not permanent role labels. Always query `V$DATABASE` and Broker before executing a role transition.

### High-level architecture

```text
                         AWS Region: us-east-1
                 ┌─────────────────────────────────┐
                 │              VPC                │
                 │                                 │
                 │  Availability Zone A            │
                 │  ┌───────────────────────────┐  │
Client ──────────┼─▶│ DB01: 10.20.2.4           │  │
                 │  │ Oracle 19c                │  │
                 │  │ DB_UNIQUE_NAME=ORCLCDG    │  │
                 │  │ PRIMARY / READ WRITE      │  │
                 │  └─────────────┬─────────────┘  │
                 │                │ ASYNC redo     │
                 │                │ transport      │
                 │  Availability Zone B            │
                 │  ┌─────────────▼─────────────┐  │
                 │  │ DB02: 10.20.3.143        │  │
                 │  │ Oracle 19c                │  │
                 │  │ DB_UNIQUE_NAME=ORCLCDG_STB│  │
                 │  │ PHYSICAL STANDBY          │  │
                 │  └───────────────────────────┘  │
                 │                                 │
                 │  Encrypted EBS / IAM / SGs      │
                 └─────────────────────────────────┘
```

## 2. Repository and workstation prerequisites

The project uses AWS CLI for account/environment inspection and Terraform for infrastructure provisioning.

On the Mac workstation, check the installed tools:

```bash
aws --version
terraform version
ssh -V
```

Confirm the AWS CLI identity and region:

```bash
aws sts get-caller-identity
aws configure get region
aws configure list
```

Set the project region for the current shell if needed:

```bash
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION="$AWS_REGION"
```

Do not place AWS access keys directly in scripts or README files. Use an approved AWS CLI profile, IAM Identity Center, or another supported credential method.

## 3. AWS pre-flight checks

Before provisioning, review quotas, networking, AMI selection, and security-group rules. The original AMI ID and all resource IDs are account-specific and should be read from the actual Terraform variables/state or AWS CLI output rather than guessed.

### 3.1 Check account and region

```bash
aws sts get-caller-identity
aws ec2 describe-availability-zones \
  --region us-east-1 \
  --query 'AvailabilityZones[].{Zone:ZoneName,State:State}' \
  --output table
```

### 3.2 Inspect available RHEL 8.10 images

Use the approved RHEL 8.10 x86-64 AMI for the account/region. AMI ownership and names can change; select and verify the exact AMI before applying Terraform.

```bash
aws ec2 describe-images \
  --region us-east-1 \
  --owners 309956199498 \
  --filters \
    'Name=architecture,Values=x86_64' \
    'Name=state,Values=available' \
    'Name=name,Values=RHEL-8.10*' \
  --query 'Images | sort_by(@,&CreationDate)[-10:].{ImageId:ImageId,Name:Name,Created:CreationDate,Architecture:Architecture}' \
  --output table
```

Review service quotas and available resources before provisioning:

```bash
aws service-quotas list-service-quotas \
  --service-code ec2 \
  --region us-east-1 \
  --output table
```

### 3.3 Networking and security

The database nodes require private communication for Oracle Net/Data Guard. Restrict access to the approved administrative sources and the peer database security group.

At minimum, verify:

- SSH administration is restricted to the approved source/bastion/SSM path.
- Oracle listener TCP port `1521` is reachable between DB01 and DB02.
- Database listener access is not exposed to `0.0.0.0/0`.
- EBS volumes are encrypted.
- IAM permissions follow least privilege.
- Private subnet routing and package-repository access work as designed.
- The security groups permit primary-to-standby and standby-to-primary Oracle Net communication needed by role transitions and validation.

Inspect resources once the IDs are known:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters 'Name=private-ip-address,Values=10.20.2.4,10.20.3.143' \
  --query 'Reservations[].Instances[].{Id:InstanceId,State:State.Name,PrivateIP:PrivateIpAddress,AZ:Placement.AvailabilityZone,Type:InstanceType,Image:ImageId}' \
  --output table
```

```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --output table
```

The instance type, AMI ID, subnet IDs, security-group IDs, and exact Terraform variable names are account-specific. Use the actual project configuration as the source of truth.

## 4. Provision AWS infrastructure with Terraform

The infrastructure phase provisions the EC2 database hosts, VPC/networking, security groups, IAM permissions, encrypted EBS storage, and supporting resources.

### 4.1 Review Terraform files and variables

From the project directory:

```bash
cd /path/to/oracle19c-dataguard-aws
find . -maxdepth 3 -type f -print | sort
```

Review the Terraform configuration and variable files before applying:

```bash
grep -RniE 'region|ami|instance_type|subnet|security_group|ebs|encrypt|iam|10\.20\.2\.4|10\.20\.3\.143' \
  --include='*.tf' --include='*.tfvars' .
```

Do not commit actual `.tfstate` files, credentials, private keys, or secret-bearing `.tfvars` files. Use a protected remote backend for team/production use.

### 4.2 Initialize, format, and validate

```bash
terraform init
terraform fmt -recursive
terraform validate
terraform plan
```

Review the plan for the correct region, RHEL 8.10 image, subnet placement, security-group rules, IAM, encrypted EBS volumes, and cost. Only apply after reviewing the plan:

```bash
terraform apply
```

If the repository uses a named variable file, use the actual project file, for example:

```bash
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

The filename above is an example; use the real variable filename in the repository.

### 4.3 Validate provisioned resources

```bash
terraform output
terraform state list
```

Check AWS instance placement and EBS encryption:

```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters 'Name=private-ip-address,Values=10.20.2.4,10.20.3.143' \
  --query 'Reservations[].Instances[].{Id:InstanceId,PrivateIP:PrivateIpAddress,AZ:Placement.AvailabilityZone,State:State.Name}' \
  --output table
```

```bash
aws ec2 describe-volumes \
  --region us-east-1 \
  --filters 'Name=status,Values=available,in-use' \
  --query 'Volumes[].{VolumeId:VolumeId,SizeGiB:Size,Encrypted:Encrypted,Type:VolumeType,State:State,AZ:AvailabilityZone}' \
  --output table
```

Do not infer that an EBS volume is attached to the intended database host from the volume list alone; verify its attachment and mount/device mapping.

## 5. Connect to DB01 and DB02

Use the approved SSH, bastion, or Systems Manager access path configured for the project. Do not expose SSH or Oracle Net publicly to make access easier.

On each host, confirm OS and resources:

```bash
hostname
hostname -I
cat /etc/redhat-release
uname -m
df -h
free -h
```

Check attached storage:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
sudo blkid
```

The project used persistent Oracle storage on encrypted EBS. The exact NVMe device names can vary across EC2 starts and instances, so identify devices by EBS mapping/serial and size rather than assuming a device name.

## 6. Linux preparation and Oracle installation

> **Command note:** The exact Oracle media filenames, patch bundle names, Oracle home path, and all one-off installation commands were not retained in the project transcript. The commands below are a reproducible outline. Use the approved Oracle 19c media and the actual Oracle home paths for this deployment.

### 6.1 OS prerequisites

On both DB01 and DB02, confirm the required packages and Oracle prerequisites according to the approved Oracle 19c installation procedure for RHEL 8.10.

```bash
sudo dnf makecache
sudo dnf repolist
```

Check Oracle account and limits:

```bash
id oracle
ulimit -a
```

If the Oracle software owner or groups are not yet configured, create them according to the approved installation standard. Do not recreate users/groups blindly on an already configured host.

### 6.2 Oracle environment

As the Oracle software owner, confirm:

```bash
echo "$ORACLE_BASE"
echo "$ORACLE_HOME"
echo "$ORACLE_SID"
which sqlplus
sqlplus -v
```

Set environment values using the actual installation paths and SID:

```bash
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=/u01/app/oracle/product/19c/dbhome_1
export PATH="$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH"
```

These paths are examples; verify the actual Oracle home before using them.

### 6.3 Verify Oracle software and patch inventory

```bash
"$ORACLE_HOME/bin/sqlplus" -v
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lspatches
```

Install the database software on both hosts using the approved Oracle 19c media and installation procedure. Keep the Oracle homes and patch levels compatible between primary and standby.

## 7. OS storage and Oracle directories

Confirm the Oracle data and recovery filesystems are mounted and persistent on each host:

```bash
df -h
lsblk -f
mount | grep -E '/u01|oracle'
```

Example directory checks/creation, adjusted to the actual Oracle layout:

```bash
sudo install -d -o oracle -g oinstall -m 775 /u01/app/oracle
sudo install -d -o oracle -g oinstall -m 775 /u01/app/oracle/oradata
sudo install -d -o oracle -g oinstall -m 775 /u01/app/oracle/fast_recovery_area
```

Do not change ownership or permissions on an existing Oracle home or data directory without confirming the intended owner/group and operational impact.

## 8. Primary database identity and baseline

The primary database uses:

- `DB_NAME`: `ORCLCDG` (same `DB_NAME` on the physical standby)
- Primary `DB_UNIQUE_NAME`: `ORCLCDG`
- Standby `DB_UNIQUE_NAME`: `ORCLCDG_STB`

The physical standby must share the primary `DB_NAME`, but each database must have a distinct `DB_UNIQUE_NAME`.

Connect to DB01 as SYSDBA:

```bash
sqlplus / as sysdba
```

Check baseline role, open mode, logging, and force logging:

```sql
SET LINESIZE 200
COLUMN NAME FORMAT A15
COLUMN DB_UNIQUE_NAME FORMAT A20
COLUMN OPEN_MODE FORMAT A20
COLUMN DATABASE_ROLE FORMAT A20

SELECT name, db_unique_name, open_mode, database_role,
       log_mode, force_logging, flashback_on
FROM v$database;

SELECT instance_name, status FROM v$instance;

SHOW PARAMETER db_name
SHOW PARAMETER db_unique_name
SHOW PARAMETER db_recovery_file_dest
SHOW PARAMETER compatible
```

Enable ARCHIVELOG if not already enabled. This requires a controlled restart and should be done only after confirming the database state and maintenance window:

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;
ARCHIVE LOG LIST;
```

Enable FORCE LOGGING:

```sql
ALTER DATABASE FORCE LOGGING;
SELECT force_logging FROM v$database;
```

Enable Flashback Database if required by the configured recovery area, retention target, and operational plan:

```sql
ALTER SYSTEM SET db_recovery_file_dest_size = <approved_size> SCOPE=BOTH;
ALTER SYSTEM SET db_recovery_file_dest = '<approved_FRA_path>' SCOPE=BOTH;

SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE FLASHBACK ON;
ALTER DATABASE OPEN;

SELECT flashback_on FROM v$database;
```

The size and path are placeholders; use the actual allocated recovery storage. Do not run the placeholder statements verbatim.

## 9. Password file and Oracle Net configuration

Data Guard redo transport and Broker require reliable Oracle Net connectivity and compatible SYS password-file authentication.

### 9.1 Password file

Check the password file on both hosts:

```bash
ls -l "$ORACLE_HOME/dbs"/orapw*
```

If creating or updating a password file, follow the approved Oracle procedure and synchronize the appropriate file to the standby. Never include the password in shell history, source control, or README text.

### 9.2 Listener

On each host:

```bash
lsnrctl status
lsnrctl services
```

Start or reload the listener only when appropriate:

```bash
lsnrctl start
lsnrctl reload
```

Confirm DB01 and DB02 can reach each other's listener on TCP 1521 using the private addresses. For example, from DB01:

```bash
nc -zv 10.20.3.143 1521
```

From DB02:

```bash
nc -zv 10.20.2.4 1521
```

### 9.3 Oracle Net aliases

The project used separate aliases for Data Guard transport/connection paths, including `ORCLCDG_DB01` and `ORCLCDG_STB`/`ORCLCDG_DB02` patterns during role transitions. The exact final aliases must match the current `tnsnames.ora`, listener services, and Broker properties.

Example pattern for `tnsnames.ora`:

```text
ORCLCDG =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.20.2.4)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCLCDG))
  )

ORCLCDG_STB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.20.3.143)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCLCDG_STB))
  )

ORCLCDG_DB01 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.20.2.4)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCLCDG))
  )

ORCLCDG_DB02 =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = 10.20.3.143)(PORT = 1521))
    (CONNECT_DATA = (SERVER = DEDICATED)(SERVICE_NAME = ORCLCDG_STB))
  )
```

These are connection-string examples, not a substitute for the actual final project file. For role-aware HA service access, use the dedicated `ORCLCDG_HA` service described later.

Test aliases:

```bash
tnsping ORCLCDG
tnsping ORCLCDG_STB
tnsping ORCLCDG_DB01
tnsping ORCLCDG_DB02
```

`tnsping` verifies name resolution/reachability but does not prove that database authentication succeeds. Test with SQL*Plus too:

```bash
sqlplus 'sys@ORCLCDG as sysdba'
sqlplus 'sys@ORCLCDG_STB as sysdba'
```

Enter credentials at the prompt.

## 10. Configure primary Data Guard parameters

On DB01 as SYSDBA, inspect existing values before changing anything:

```sql
SHOW PARAMETER db_unique_name
SHOW PARAMETER log_archive_config
SHOW PARAMETER log_archive_dest
SHOW PARAMETER log_archive_dest_state
SHOW PARAMETER fal_server
SHOW PARAMETER standby_file_management
```

Set the Data Guard configuration membership:

```sql
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG =
  'DG_CONFIG=(ORCLCDG,ORCLCDG_STB)' SCOPE=BOTH;
```

Configure asynchronous redo transport from primary to standby. Use the alias that resolves to the current standby service:

```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2 =
  'SERVICE=ORCLCDG_STB ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=ORCLCDG_STB'
  SCOPE=BOTH;

ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH;
```

On DB02, set the same DG_CONFIG membership and standby file management:

```sql
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG =
  'DG_CONFIG=(ORCLCDG,ORCLCDG_STB)' SCOPE=BOTH;

ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO SCOPE=BOTH;
ALTER SYSTEM SET FAL_SERVER='ORCLCDG_DB01' SCOPE=BOTH;
```

The exact `SERVICE=` value should be aligned with the final alias/transport configuration used in the environment. If Broker is managing the configuration, use Broker properties for Broker-managed settings and avoid competing manual edits.

## 11. RMAN baseline backup

Before creating/rebuilding a standby or performing a risky maintenance operation, verify a usable RMAN backup and archive-log coverage.

```bash
rman target /
```

At the RMAN prompt:

```rman
SHOW ALL;
REPORT SCHEMA;
LIST BACKUP SUMMARY;
```

Example baseline backup (confirm backup destination, retention, and free space first):

```rman
CONFIGURE CONTROLFILE AUTOBACKUP ON;
BACKUP DATABASE PLUS ARCHIVELOG;
BACKUP CURRENT CONTROLFILE;
BACKUP SPFILE;
LIST BACKUP SUMMARY;
```

Validate recoverability:

```rman
RESTORE DATABASE VALIDATE;
VALIDATE DATABASE;
```

RMAN command details and backup destinations should match the actual configured backup strategy. A successful backup is not sufficient by itself; a restore/validation exercise is required.

## 12. Create the physical standby with RMAN

The project created the standby using RMAN active duplication.

### 12.1 Prepare DB02

Confirm the standby host has compatible Oracle software, listener connectivity, password-file authentication, storage, and a minimal initialization configuration. `DB_NAME` matches the primary, while `DB_UNIQUE_NAME` is `ORCLCDG_STB`.

Example minimal init file pattern for a fresh standby (use the actual SID and paths):

```text
DB_NAME=ORCLCDG
DB_UNIQUE_NAME=ORCLCDG_STB
```

Start the auxiliary instance in NOMOUNT as appropriate:

```sql
STARTUP NOMOUNT;
```

### 12.2 Run RMAN active duplicate

From an Oracle environment that can connect to both databases:

```bash
rman target 'sys@ORCLCDG' auxiliary 'sys@ORCLCDG_STB'
```

At the RMAN prompt, use the duplicate form validated for the actual file layout:

```rman
DUPLICATE TARGET DATABASE
  FOR STANDBY
  FROM ACTIVE DATABASE
  DORECOVER
  NOFILENAMECHECK;
```

> **Caution:** `NOFILENAMECHECK` is only appropriate when the primary and standby are on separate hosts/storage and the identical filenames cannot overwrite the primary's files. Verify the file layout before using it. If paths differ, use the appropriate file-name conversion or OMF configuration instead.

After duplication, validate DB02:

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;
```

Expected role: `PHYSICAL STANDBY`.

## 13. Standby redo logs and managed recovery

### 13.1 Inspect online and standby redo logs

Run on the primary and standby:

```sql
SET LINESIZE 200
SELECT group#, thread#, bytes/1024/1024 AS size_mb, status
FROM v$log
ORDER BY group#;

SELECT group#, thread#, bytes/1024/1024 AS size_mb, status
FROM v$standby_log
ORDER BY group#;
```

The standby redo log design should account for each redo thread and generally provide at least one more standby redo log group per thread than the online redo log group count, with appropriately matched sizes. Use the actual online redo log size and thread count.

Example only:

```sql
ALTER DATABASE ADD STANDBY LOGFILE SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 500M;
ALTER DATABASE ADD STANDBY LOGFILE SIZE 500M;
```

Do not execute these sample `ADD STANDBY LOGFILE` statements without checking existing groups and the actual log size.

### 13.2 Start managed recovery

On DB02, mount the physical standby if required and start managed recovery:

```sql
ALTER DATABASE MOUNT STANDBY DATABASE;
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT;
```

During later role transitions, the exact startup/recovery command depends on the current state and whether Broker is controlling the database. Avoid starting duplicate recovery processes or mixing manual and Broker control.

### 13.3 Validate redo receipt and apply

On DB02:

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;

SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;

SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;

SELECT * FROM v$archive_gap;
```

On DB01, inspect destinations:

```sql
SELECT dest_id, status, destination, error, target, db_unique_name
FROM v$archive_dest
WHERE status <> 'INACTIVE'
ORDER BY dest_id;
```

The project recorded `MRP0 APPLYING_LOG`, no archive gap, and zero transport/apply lag at final validation.

## 14. Data Guard Broker configuration

Data Guard Broker was configured and used to validate the configuration and perform role transitions.

### 14.1 Enable Broker

On both databases:

```sql
ALTER SYSTEM SET DG_BROKER_START=TRUE SCOPE=BOTH;
```

Confirm the static listener services required by Broker are configured on both hosts. Listener static service names, `ORACLE_HOME`, and `SID_NAME` must match the actual installation and instance names.

### 14.2 Connect with DGMGRL and configure

Example command pattern (the exact connection identifiers must match the final listener/TNS setup):

```bash
dgmgrl 'sys@ORCLCDG'
```

At the DGMGRL prompt, a fresh configuration is typically created using:

```text
CREATE CONFIGURATION 'orclcdg_dr'
  AS PRIMARY DATABASE IS 'ORCLCDG'
  CONNECT IDENTIFIER IS 'ORCLCDG';

ADD DATABASE 'ORCLCDG_STB'
  AS CONNECT IDENTIFIER IS 'ORCLCDG_STB'
  MAINTAINED AS PHYSICAL;

ENABLE CONFIGURATION;

SHOW CONFIGURATION;
SHOW DATABASE VERBOSE 'ORCLCDG';
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
```

> **Do not re-create an existing Broker configuration.** These commands document the initial setup pattern. If the configuration already exists, inspect it and use `EDIT DATABASE` / `EDIT CONFIGURATION` as appropriate.

### 14.3 Project-specific Broker connectivity fix

A Broker connectivity issue was resolved by using an explicit host:port/service identifier rather than relying on an alias that was not resolving as expected.

The successful primary identifier recorded in the project was:

```text
10.20.2.4:1521/ORCLCDG_DGMGRL
```

The property command used was:

```text
EDIT DATABASE ORCLCDG
  SET PROPERTY DGConnectIdentifier = '10.20.2.4:1521/ORCLCDG_DGMGRL';
```

The configuration was then refreshed:

```text
DISABLE CONFIGURATION;
ENABLE CONFIGURATION;
SHOW CONFIGURATION;
```

This sequence was used after correcting the connection identifier. Do not disable a healthy configuration as a routine action. If Broker cannot reconnect, first test the exact static service using SQL*Plus and inspect the listener.

### 14.4 Validate Broker and network connectivity

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW DATABASE VERBOSE 'ORCLCDG';
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
VALIDATE DATABASE VERBOSE ORCLCDG;
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
VALIDATE NETWORK CONFIGURATION FOR ALL;
```

Run validation before and after role transitions and patching.

## 15. Role-aware HA client connectivity

The project included an HA service named `ORCLCDG_HA`, TAF, startup and `DB_ROLE_CHANGE` triggers, and separate Data Guard transport aliases. The purpose is to keep application connection routing aligned with the database role after a switchover or failover.

### 15.1 Verify the service

On the database where the service should be active, inspect registered services:

```sql
SELECT name, network_name
FROM v$services
ORDER BY name;
```

Check listener registration:

```bash
lsnrctl services
```

### 15.2 Service and trigger validation

The exact service-management command and trigger body depend on the final implementation in the project. Inspect the deployed service and triggers rather than blindly recreating them:

```sql
SELECT name, network_name
FROM v$services
WHERE UPPER(name) LIKE '%ORCLCDG_HA%';

SELECT owner, trigger_name, status, triggering_event
FROM dba_triggers
WHERE UPPER(trigger_name) LIKE '%ROLE%'
   OR UPPER(triggering_event) LIKE '%DB_ROLE_CHANGE%';
```

For RAC/Clusterware-managed databases, use `srvctl` for service management; for a standalone Oracle Restart/non-Clusterware environment, use the service-management approach appropriate to the actual deployment. Do not mix service-management mechanisms without confirming the architecture.

### 15.3 Test client aliases

From an approved client environment, test the HA alias after confirming its actual `tnsnames.ora` descriptor:

```bash
tnsping ORCLCDG_HA
sqlplus '<application_user>@ORCLCDG_HA'
```

After each planned role transition, verify that the service is offered by the current primary and that a new client connection reaches a read/write primary. Existing sessions may behave according to the configured TAF/application behavior; validate the intended client experience rather than assuming all in-flight transactions survive.

## 16. Planned switchover — DB01 to DB02

A planned switchover was tested. Before switching, confirm both databases are healthy and synchronized.

### 16.1 Pre-switchover checks

In SQL*Plus on both hosts:

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;
```

Check Broker:

```text
SHOW CONFIGURATION;
VALIDATE DATABASE VERBOSE ORCLCDG;
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
```

Check lag and gaps on the standby:

```sql
SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;

SELECT * FROM v$archive_gap;
```

Do not proceed if the configuration is unhealthy, redo is not caught up, a gap remains unresolved, or the required role-transition prerequisites are not met.

### 16.2 Execute Broker switchover

Use the current primary's DGMGRL endpoint:

```text
SWITCHOVER TO 'ORCLCDG_STB';
SHOW CONFIGURATION;
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
SHOW DATABASE VERBOSE 'ORCLCDG';
```

The project also tested manual role-transition/recovery operations in addition to Broker-managed switchover. For routine controlled transitions, prefer the validated Broker workflow when Broker is managing the configuration.

### 16.3 Validate after switchover

On both hosts:

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;
```

Verify the new primary is `PRIMARY` / `READ WRITE`, the former primary is a physical standby, redo apply is active, and the Broker configuration is healthy.

## 17. Controlled failover and former-primary reinstate

The project tested a controlled failover, promoted the standby, then used Flashback Database to recover/rebuild the former primary and return it to the standby role. These operations are disruptive and must be performed only under a documented DR decision and recovery plan.

> **Safety note:** A real unplanned failover is not the same as a planned switchover. Confirm the original primary is truly unavailable or fenced before promoting a standby. Avoid split brain by ensuring the former primary cannot continue accepting writes.

### 17.1 Capture role and recovery state

On the current primary and standby:

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;

SELECT flashback_on FROM v$database;

SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;
```

On the standby:

```sql
SELECT * FROM v$archive_gap;
```

### 17.2 Failover

The exact Broker command depends on the failure state and validation result. When Broker is available, inspect the configuration and validate the target before issuing the role transition:

```text
SHOW CONFIGURATION;
SHOW DATABASE VERBOSE 'ORCLCDG';
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
```

For a Broker-managed failover, use the command supported by the current Broker state and the approved DR procedure; do not copy a failover command blindly into an unrelated state.

For a manual failover, use Oracle's documented role-transition sequence appropriate to the observed database state. The project exercised manual failover and promoted the standby, but the exact one-off command sequence was not retained verbatim in the available conversation.

### 17.3 Rebuild/reinstate the former primary

The project used Flashback Database to the SCN at which the standby became primary, then reconverted/reinstated the former primary as a physical standby and restored redo apply.

A safe operational outline is:

1. Confirm the new primary is stable and accepting writes.
2. Fence the former primary so it cannot accept writes.
3. Record the role-transition SCN and verify Flashback Database/recovery files are available.
4. Flash back or reinstate the former primary using the exact SCN and supported Oracle/Broker procedure.
5. Convert the former primary to physical standby if required by its resulting state.
6. Start managed recovery or let Broker manage recovery.
7. Validate redo transport, apply, and absence of gaps.
8. Confirm the Broker configuration is healthy and the intended roles are restored.

Inspection commands:

```sql
SELECT current_scn FROM v$database;
SELECT flashback_on FROM v$database;
SELECT oldest_flashback_scn, oldest_flashback_time
FROM v$flashback_database_log;
```

After the former primary has been returned to standby, validate:

```sql
SELECT name, db_unique_name, open_mode, database_role
FROM v$database;

SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;

SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;

SELECT * FROM v$archive_gap;
```

> The exact failover/reinstate command sequence and SCN from the original hands-on session were not retained in the conversation. This section deliberately documents the tested workflow and validation commands without inventing an exact historical SCN or claiming the template is a verbatim transcript.

## 18. Snapshot Standby validation

Snapshot Standby testing was completed to validate temporary read/write testing on a standby and subsequent return to physical standby. Snapshot Standby is not a replacement for backup or a permanent writable DR target.

Before conversion, confirm the current role and Broker health:

```sql
SELECT name, db_unique_name, open_mode, database_role
FROM v$database;
```

```text
SHOW CONFIGURATION;
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
```

Use the supported Broker conversion command for the current state, and confirm the required Flashback/recovery prerequisites before converting. The command pattern is:

```text
CONVERT DATABASE 'ORCLCDG_STB' TO SNAPSHOT STANDBY;
```

After testing, convert it back:

```text
CONVERT DATABASE 'ORCLCDG_STB' TO PHYSICAL STANDBY;
```

Then verify the physical standby role, recovery, and Broker status:

```sql
SELECT name, db_unique_name, open_mode, database_role
FROM v$database;
```

```text
SHOW CONFIGURATION;
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
```

These are Broker command patterns; confirm the actual target name and current state before execution.

## 19. Active Data Guard validation

The project validated Active Data Guard `READ ONLY WITH APPLY`.

On the standby, inspect the role/open mode:

```sql
SELECT name, db_unique_name, open_mode, database_role
FROM v$database;
```

Expected Active Data Guard standby state:

- `DATABASE_ROLE = PHYSICAL STANDBY`
- `OPEN_MODE = READ ONLY WITH APPLY`

Confirm apply activity and lag:

```sql
SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;

SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;
```

Return the standby to the normal state required by the operating procedure before any further role transition.

## 20. RMAN backup, recovery, and validation

RMAN backup/recovery is part of the DR implementation, not an optional add-on. Use the approved backup destination, retention policy, and storage lifecycle.

### 20.1 Inspect RMAN configuration and backups

```bash
rman target /
```

```rman
SHOW ALL;
LIST BACKUP SUMMARY;
LIST BACKUP OF DATABASE;
LIST BACKUP OF ARCHIVELOG ALL;
REPORT SCHEMA;
REPORT OBSOLETE;
```

### 20.2 Example database and archive backup

```rman
CONFIGURE CONTROLFILE AUTOBACKUP ON;
BACKUP DATABASE PLUS ARCHIVELOG;
BACKUP CURRENT CONTROLFILE;
BACKUP SPFILE;
LIST BACKUP SUMMARY;
```

### 20.3 Validate restore capability

```rman
RESTORE DATABASE VALIDATE;
VALIDATE DATABASE;
```

For an actual recovery exercise, use a controlled test database/restore target or a documented maintenance window. Do not run destructive restore/recovery commands against the active production database without an approved plan.

### 20.4 Recovery validation checklist

- Backup completes without errors.
- Control file and SPFILE backups are available.
- Required archived redo is retained.
- Restore validation succeeds.
- Recovery steps and recovery point are documented.
- Backup copies are stored in the intended durable, access-controlled location.
- Restore is tested, not merely assumed based on backup completion.

## 21. July 2026 Release Update patching and 21c upgrade planning

Patching and a future upgrade are separate change phases and should be clearly represented in the project record. The project notes indicate that Oracle patching and the upgrade to Oracle 21c were planned for later implementation; do not describe either as complete unless the actual post-change inventory proves it.

### 21.1 Capture pre-patch inventory on both hosts

```bash
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lspatches
```

```sql
SELECT banner_full FROM v$version;

SELECT name, db_unique_name, open_mode, database_role
FROM v$database;

SELECT comp_id, version, status
FROM dba_registry
ORDER BY comp_id;

SELECT owner, object_type, COUNT(*) AS invalid_count
FROM dba_objects
WHERE status = 'INVALID'
GROUP BY owner, object_type
ORDER BY owner, object_type;
```

### 21.2 Pre-patch readiness

- Confirm the exact approved July 2026 RU package and README.
- Confirm OPatch prerequisites and inventory on both Oracle homes.
- Confirm a usable RMAN backup and recovery path.
- Confirm Data Guard is synchronized and Broker validation is healthy.
- Confirm sufficient free space in the Oracle home and recovery destinations.
- Document the patch order and role-transition strategy.
- Use the current Oracle instructions for Data Guard rolling patching, if supported for the selected RU and configuration.

### 21.3 Post-patch validation

After the patch is actually applied, repeat:

```bash
"$ORACLE_HOME/OPatch/opatch" lspatches
```

```sql
SELECT banner_full FROM v$version;

SELECT name, db_unique_name, open_mode, database_role
FROM v$database;

SELECT comp_id, version, status
FROM dba_registry
ORDER BY comp_id;
```

Also validate Data Guard Broker, redo transport/apply, archive gaps, and application connectivity. Do not claim the July 2026 RU is installed until the output verifies it.

### 21.4 Oracle 21c upgrade — planned

Upgrade to Oracle Database 21c is a future phase. Before starting, confirm Oracle support/lifecycle considerations, target architecture, upgrade method, application compatibility, backup/recovery, Data Guard strategy, and rollback plan. Capture pre-upgrade inventory and execute the approved upgrade procedure in a separate change window.

## 22. Day-to-day monitoring commands

### Database role and mode

```sql
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;
```

### Redo transport destinations

```sql
SELECT dest_id, status, destination, error, target, db_unique_name
FROM v$archive_dest
WHERE status <> 'INACTIVE'
ORDER BY dest_id;
```

### Apply lag and transport lag

```sql
SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;
```

### Archive gaps

```sql
SELECT * FROM v$archive_gap;
```

### Redo receipt/apply process

```sql
SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;
```

For 19c environments where process views differ by release or state, also consult the supported Data Guard process views and Broker validation output.

### Recent archived logs

```sql
SELECT thread#, sequence#, first_time, next_time, archived, applied
FROM v$archived_log
ORDER BY thread#, sequence# DESC
FETCH FIRST 30 ROWS ONLY;
```

### Broker status

```text
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW DATABASE VERBOSE 'ORCLCDG';
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
VALIDATE DATABASE VERBOSE ORCLCDG;
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
VALIDATE NETWORK CONFIGURATION FOR ALL;
```

## 23. Troubleshooting notes and lessons learned

### Broker cannot connect to a database

1. Check `SHOW CONFIGURATION VERBOSE`.
2. Test the exact `DGConnectIdentifier` host, port, and service with SQL*Plus.
3. Check `lsnrctl status` and `lsnrctl services` on the destination.
4. Confirm TCP 1521 reachability and security-group rules.
5. Correct the Broker property only after the actual service is verified.
6. Re-enable/refresh the Broker configuration only when appropriate.

The project-specific primary identifier that resolved the recorded issue was:

```text
10.20.2.4:1521/ORCLCDG_DGMGRL
```

### Redo transport error

```sql
SELECT dest_id, status, destination, error
FROM v$archive_dest
WHERE dest_id IN (1,2);
```

Review the destination service, listener, password-file authentication, `DB_UNIQUE_NAME`, network path, and `LOG_ARCHIVE_DEST_STATE_n`.

### Standby not applying

```sql
SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;

SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;

SELECT * FROM v$archive_gap;
```

Check whether the standby is mounted/open in the intended mode, whether recovery is active, and whether required archived redo is available.

### Switchover blocked

- Inspect both database roles and `SWITCHOVER_STATUS`.
- Run Broker `VALIDATE DATABASE VERBOSE` for both databases.
- Confirm transport/apply are healthy and no gap remains.
- Confirm the client service and static listener endpoints are valid.
- Do not force a transition simply to bypass an unexplained warning.

### Listener works but Broker fails

A listener being up does not prove the exact static service used by Broker is valid. Test the precise host:port/service with SQL*Plus and align it with `DGConnectIdentifier` and static service configuration.

### Flashback/reinstate issue

Before relying on Flashback Database for reinstate, verify that Flashback is enabled, the required flashback logs still exist, and the target SCN is available. Do not invent or guess the SCN.

## 24. Final validation results recorded for the project

At the final validation recorded in the project:

- DB01 / `ORCLCDG` was `PRIMARY` and `READ WRITE`.
- DB02 / `ORCLCDG_STB` was `PHYSICAL STANDBY`.
- Managed recovery showed `MRP0 APPLYING_LOG`.
- Transport lag was `+00 00:00:00`.
- Apply lag was `+00 00:00:00`.
- No archive gap was reported.
- Planned switchover was validated in both directions.
- Controlled failover and Flashback-based former-primary rebuild/reinstate were tested.
- Snapshot Standby conversion/testing and return to Physical Standby were validated.
- Active Data Guard `READ ONLY WITH APPLY` was validated.
- The role-aware `ORCLCDG_HA` client connectivity path, TAF, and role-change automation were tested.

These are the results recorded for the project at the time of testing; rerun the commands in this README to establish the current state before making operational decisions.

## 25. Production-readiness checklist

- [ ] Terraform plan and AWS resource configuration reviewed and stored.
- [ ] EC2 instances are in the intended Availability Zones and private subnets.
- [ ] EBS encryption and storage persistence verified.
- [ ] Security groups restrict SSH and Oracle Net access.
- [ ] Oracle software versions and patch inventory recorded on both hosts.
- [ ] ARCHIVELOG and FORCE LOGGING verified on the primary.
- [ ] Flashback Database configuration and recovery area reviewed.
- [ ] Password files and Oracle Net connectivity verified.
- [ ] Standby redo logs checked against online redo logs and thread count.
- [ ] RMAN backup and restore validation completed.
- [ ] Broker configuration healthy.
- [ ] `DGConnectIdentifier` and static listener services tested directly.
- [ ] Redo transport healthy and no archive gap.
- [ ] Apply/transport lag within the agreed threshold.
- [ ] `ORCLCDG_HA` and client connectivity validated after role transitions.
- [ ] Planned switchover completed and documented.
- [ ] Controlled failover and former-primary reinstate completed and documented.
- [ ] Snapshot Standby test completed and returned to Physical Standby.
- [ ] Active Data Guard read-only apply validated.
- [ ] July 2026 RU patching status explicitly verified and documented.
- [ ] Oracle 21c upgrade remains marked planned until completed.
- [ ] Runbook, command output, screenshots, and change records are stored with the repository.

## 26. Evidence to keep in the GitHub repository

Recommended non-secret evidence:

- Architecture diagram and resource summary.
- Terraform module/input/output documentation (without state or secrets).
- `V$DATABASE` role/open-mode output from primary and standby.
- `V$ARCHIVE_DEST`, `V$DATAGUARD_STATS`, `V$ARCHIVE_GAP`, and managed-recovery output.
- `DGMGRL SHOW CONFIGURATION` and `VALIDATE DATABASE VERBOSE` output.
- RMAN backup summary and restore-validation output.
- Planned switchover and failover/reinstate evidence.
- Snapshot Standby and Active Data Guard validation evidence.
- Pre-/post-patch OPatch inventory once patching is completed.
- A short incident note documenting the `DGConnectIdentifier` connectivity fix.

Never commit passwords, private keys, Oracle wallet secrets, AWS credentials, Terraform state, or production data.

## Appendix A. Quick command reference

```sql
-- Database role and mode
SELECT name, db_unique_name, open_mode, database_role, switchover_status
FROM v$database;

-- Redo destinations
SELECT dest_id, status, destination, error
FROM v$archive_dest
WHERE status <> 'INACTIVE';

-- Lag
SELECT name, value, unit, time_computed
FROM v$dataguard_stats
ORDER BY name;

-- Archive gap
SELECT * FROM v$archive_gap;

-- Managed recovery
SELECT process, status, thread#, sequence#
FROM v$managed_standby
ORDER BY process;
```

```text
-- DGMGRL
SHOW CONFIGURATION;
SHOW CONFIGURATION VERBOSE;
SHOW DATABASE VERBOSE 'ORCLCDG';
SHOW DATABASE VERBOSE 'ORCLCDG_STB';
VALIDATE DATABASE VERBOSE ORCLCDG;
VALIDATE DATABASE VERBOSE ORCLCDG_STB;
VALIDATE NETWORK CONFIGURATION FOR ALL;

-- Project-specific primary Broker connection fix
EDIT DATABASE ORCLCDG
  SET PROPERTY DGConnectIdentifier = '10.20.2.4:1521/ORCLCDG_DGMGRL';
```

```bash
# Oracle software and patch inventory
"$ORACLE_HOME/OPatch/opatch" version
"$ORACLE_HOME/OPatch/opatch" lspatches

# RMAN
rman target /
# At RMAN prompt:
# SHOW ALL;
# LIST BACKUP SUMMARY;
# RESTORE DATABASE VALIDATE;
```

## Appendix B. Project status summary

**Implemented and validated:** AWS/Terraform infrastructure for the two-node Oracle 19c environment; primary and physical standby; RMAN active duplicate; redo transport/apply; Data Guard Broker; role-aware HA service/client testing; planned switchover; controlled failover; Flashback-based former-primary recovery/reinstate; Snapshot Standby; Active Data Guard; lag/gap checks; RMAN backup/recovery validation workflow.

**Requires current-state verification before claiming complete:** exact July 2026 RU installation and datapatch status, any post-patch validation, and Oracle 21c upgrade. The 21c upgrade was identified as future work. Do not mark these complete without recorded command output.

---
