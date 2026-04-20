# State of Michigan CP4D 5.1.2 to 5.3.1 Upgrade

**From:**

```
CPD: 5.1.2
OCP: 4.16
Storage: Portworx
Internet: airgap
Private container registry: yes
Components: ibm_licensing,cpd_platform,analyticsengine,ws_pipelines,mantaflow,wkc,datastage_ent_plus
```

**To:**

```
CPD: 5.3.1
OCP: 4.16
Storage: Portworx
Internet: airgap
Private container registry: yes
Components: ibm_licensing,cpd_platform,analyticsengine,ws_pipelines,mantaflow,wkc,datastage_ent_plus
```

---

## Prerequisites

#### 1. Backup of the cluster is done

Backup your IBM Software Hub cluster before the upgrade.
For details, see [Backing up and restoring IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=administering-backing-up-restoring-software-hub).

**Note:**
Some services don't support the offline OADP backup. Review the backup documentation and take the dedicated approach when necessary.

#### 2. The image mirroring completed successfully

If a private container registry is in-use to host the IBM Software Hub software images, you must mirror the updated images from the IBM® Entitled Registry to the private container registry.

Reference: [Mirroring images to private image registry](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=mipcr-mirroring-images-directly-private-container-registry-1)

#### 3. The permissions required for the upgrade is ready

- OpenShift cluster administrator permissions
- IBM Software Hub administrator permissions
- Permission to access the private image registry for pushing or pulling images
- Access to the bastion node for executing the upgrade commands

#### 4. A pre-upgrade health check is made to ensure the cluster's readiness for upgrade

- The OpenShift cluster, persistent storage and IBM Software Hub platform and services are in healthy status

---

## Table of Contents

- [Pre Upgrade Steps](#pre-upgrade-steps)
- [Pre Upgrade Backups](#pre-upgrade-backups)
- [OpenShift Container Platform Upgrade](#openshift-container-platform-upgrade)
- [Upgrade Execution](#upgrade-execution)
- [Service Instance Upgrades](#service-instance-upgrades)
- [Upgrade cpdbr Service](#upgrade-cpdbr-service)
- [RSI Patch Management](#rsi-patch-management)
- [Post Upgrade Validation](#post-upgrade-validation)

---

## Pre Upgrade Steps

### Required Tools

Ensure the following tools are installed and updated to the required versions:

- **IBM Software Hub CLI**: Version 14.3.1
- **OpenShift CLI (oc)**: Compatible version for your cluster
- **Helm CLI**: Version 3.16.3

**Installation and Update Instructions:**

For detailed instructions on installing or updating these tools, refer to:
- [Updating client workstations](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=53-updating-client-workstations)
  - [Updating IBM Software Hub CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-software-hub-cli-1)
  - [Updating OpenShift CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-updating-openshift-cli-1)
  - [Installing Helm CLI](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=ucw-installing-helm-cli-1)

### Access Requirements

**Required Access:**
- OpenShift cluster admin access
- IBM Entitlement Key with appropriate permissions
- Access to IBM Container Registry (cp.icr.io)
- Access to private registry: stateofmichigan.example.com:5000

### Environment Variables Setup (cpd_vars.sh)

Ensure your environment variables script is configured correctly. See [IBM Documentation](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cri-updating-your-environment-variables-script-1).

**Verify environment variables:**

```bash
# Source your environment variables script
source cpd_vars.sh

# Verify key variables are set
echo "CPD Version: ${VERSION}"
echo "OCP URL: ${OCP_URL}"
echo "Operators Namespace: ${PROJECT_CPD_INST_OPERATORS}"
echo "Operands Namespace: ${PROJECT_CPD_INST_OPERANDS}"
echo "Block Storage: ${STG_CLASS_BLOCK}"
echo "File Storage: ${STG_CLASS_FILE}"
echo "Components: ${COMPONENTS}"
echo "Private Registry: ${PRIVATE_REGISTRY_LOCATION}"
```

### Login to OpenShift Cluster

```bash
# Login using cpd-cli
${CPDM_OC_LOGIN}

# Or login using oc directly
${OC_LOGIN}

# Verify cluster access
oc whoami
oc get nodes
```

### Restart OLM Utils Container

```bash
# Restart the OLM utils container with updated environment
cpd-cli manage restart-container

# Verify container is running
podman ps | grep olm-utils
```

---

## Pre Upgrade Health Check

### Run Comprehensive Health Check

```bash
# Run CPD health check (includes cluster, nodes, operands, operators)
# This command also creates a backup of the current state
cpd-cli health runcommand \
--commands=cluster,nodes,operands,operators \
--control_plane_ns="${PROJECT_CPD_INST_OPERANDS}" \
--operator_ns="${PROJECT_CPD_INST_OPERATORS}" \
--log-level=debug \
--verbose \
--save

# The health check will create a timestamped directory with results
# Review the output for any critical issues before proceeding
```

### Check for Hot Fixes and Patches

```bash
# Check for image_digests in service CRs (indicates hot fixes/patches)
oc project ${PROJECT_CPD_INST_OPERANDS}

for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}' | grep -v zenextensions); do
  echo "************* $i *************"
  for x in $(oc get $i --no-headers | awk '{print $1}'); do
    echo "--------- $x ------------"
    oc get $i $x -o jsonpath={.spec} | jq
  done
done

# Review output for any image_digests fields
# Document any hot fixes/patches that need to be removed before upgrade
```

### Basic Cluster Validation

```bash
# Check node status
oc get nodes

# Check cluster operators
oc get co

# Check cluster version
oc get clusterversion
```

### Storage Validation

```bash
# Verify storage classes
oc get sc

# Check PVC status
oc get pvc -n ${PROJECT_CPD_INST_OPERANDS}

# Verify storage vendor health
# Portworx status
PX_POD=$(oc get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
oc exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
```

### CPD Platform Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for pods not running correctly (excludes completed jobs)
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'

# List service instances
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

## Upgrade Execution

### 4.1 Prepare Cluster for Upgrade

#### 4.1.1 Update Cluster-Scoped Resources for Shared Components

**Reference**: [Updating cluster-scoped resources for shared cluster components](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=pyc-updating-cluster-scoped-resources-shared-cluster-components-1)

```bash
# Generate cluster-scoped resource definitions for scheduling service
cpd-cli manage case-download \
  --components=scheduler \
  --release=${VERSION} \
  --scheduler_ns=${PROJECT_SCHEDULING_SERVICE} \
  --cluster_resources=true

# Change to work directory
cd cpd-cli-workspace/olm-utils-workspace/work

# Apply cluster-scoped resources
oc apply -f cluster_scoped_resources.yaml \
  --server-side \
  --force-conflicts

# Optional: Keep a record
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_SCHEDULING_SERVICE}-cluster_scoped_resources.yaml

# Return to base directory
cd -
```

#### 4.1.2 Update Cluster-Scoped Resources for CPD Instance

**Reference**: [Updating cluster-scoped resources for the instance](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=puish-updating-cluster-scoped-resources-instance-1)

```bash
# Generate cluster-scoped resource definitions for CPD instance
cpd-cli manage case-download \
  --components=${COMPONENTS} \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --cluster_resources=true

# Change to work directory
cd cpd-cli-workspace/olm-utils-workspace/work

# Apply cluster-scoped resources
oc apply -f cluster_scoped_resources.yaml \
  --server-side \
  --force-conflicts

# Optional: Keep a record
mv cluster_scoped_resources.yaml ${VERSION}-${PROJECT_CPD_INST_OPERATORS}-cluster_scoped_resources.yaml

# Return to base directory
cd -
```

#### 4.1.3 Creating image pull secrets for shared cluster components

Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task
```bash
${OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from
```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

#### 4.1.4 Creating image pull secrets for the instance  

Log in to Red Hat® OpenShift® Container Platform as a user with sufficient permissions to complete the task
```bash
${OC_LOGIN}
```

Create a file named dockerconfig.json based on where your cluster pulls images from
```bash
cat <<EOF > dockerconfig.json 
{
  "auths": {
    "${PRIVATE_REGISTRY_LOCATION}": {
      "auth": "${IMAGE_PULL_CREDENTIALS}"
    }
  }
}
EOF
```

Create the image pull secret in the operators project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

Create the image pull secret in the operands project for the instance
```bash
oc create secret docker-registry ${IMAGE_PULL_SECRET} \
--from-file ".dockerconfigjson=dockerconfig.json" \
--namespace=${PROJECT_CPD_INST_OPERATORS}
```

#### 4.1.5 Apply Entitlements

**Reference**: [Applying your entitlements](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=aye-applying-your-entitlements-without-node-pinning-2)

```bash
# Apply IBM Cloud Pak for Data Enterprise Edition entitlement
cpd-cli manage apply-entitlement \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --entitlement=cpd-enterprise \
  --production=true
```

---

### 4.2 Upgrade Shared Cluster Components

#### 4.2.1 Upgrade IBM Licensing

**Reference**: [Upgrading shared cluster components](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=upgrading-shared-cluster-components)

```bash
# Upgrade IBM Licensing service
cpd-cli manage apply-cluster-components \
  --release=${VERSION} \
  --license_acceptance=true \
  --licensing_ns=${PROJECT_LICENSE_SERVICE}

# Verify licensing pods are running
oc get pods -n ${PROJECT_LICENSE_SERVICE}
```

#### 4.2.2 Upgrade Scheduler (if installed)

**Reference**: [Upgrading the scheduling service](https://www.ibm.com/docs/en/cloud-paks/cp-data/5.3.x?topic=components-upgrading-scheduling-service)

```bash
# Check if scheduler is installed
oc get scheduling -A

# If scheduler exists, upgrade it
cpd-cli manage apply-scheduler \
  --release=${VERSION} \
  --license_acceptance=true \
  --scheduler_ns=${PROJECT_SCHEDULING_SERVICE}

# Verify scheduler pods are running
oc get pods -n ${PROJECT_SCHEDULING_SERVICE}
```

---

### 4.3 Upgrade IBM Software Hub Platform

**Reference**: [Upgrading IBM Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

```bash
# Upgrade CPD platform using install-components (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=cpd_platform \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor platform upgrade progress (this takes 60-80 minutes)
watch -n 30 'oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath="{.status.zenStatus}"'

# Check platform pods
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep -E "zen|usermgmt|ibm-nginx"

# Verify platform version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'
```

---

### 4.4 Upgrade Services

**Individual Service Upgrade (For More Control)**

Upgrade services one at a time for better monitoring and troubleshooting:

#### 4.4.1 Upgrade Watson Knowledge Catalog

```bash
# Upgrade wkc (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=wkc \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor wkc upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=wkc
```

#### 4.4.2 Upgrade Manta Flow

```bash
# Upgrade mantaflow (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=mantaflow \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor mantaflow upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=mantaflow
```

#### 4.4.3 Upgrade Watson Pipelines

```bash
# Upgrade ws_pipelines (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=ws_pipelines \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor ws_pipelines upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=ws_pipelines
```

#### 4.4.4 Upgrade DataStage Enterprise Plus

```bash
# Upgrade datastage_ent_plus (5.3.x method)
cpd-cli manage install-components \
  --license_acceptance=true \
  --components=datastage_ent_plus \
  --release=${VERSION} \
  --operator_ns=${PROJECT_CPD_INST_OPERATORS} \
  --instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --image_pull_prefix=${IMAGE_PULL_PREFIX} \
  --image_pull_secret=${IMAGE_PULL_SECRET} \
  --run_storage_tests=false \
  --upgrade=true

# Monitor datastage_ent_plus upgrade
cpd-cli manage get-cr-status \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --components=datastage_ent_plus
```

---

## Service Instance Upgrades

After upgrading service CRs, some services require additional instance upgrades. The following services in your configuration need manual instance upgrades:

### Prerequisites

1. Complete all CR upgrades successfully
2. Create a CPD profile with these permissions:
   - `can_provision` (Create service instances)
   - `manage_service_instances` (Manage service instances)
3. Set the `CPD_PROFILE_NAME` environment variable

**Documentation**: [Creating a CPD profile](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=cli-creating-cpd-profile)

---

### Services Requiring Manual Instance Upgrades

#### 1. Analytics Engine

**Service Type**: `spark`

**Upgrade Command** (all instances):
```bash
cpd-cli service-instance upgrade \
--service-type=spark \
--profile=${CPD_PROFILE_NAME} \
--all
```

**Documentation**: [Upgrading Analytics Engine](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=u-upgrading-from-version-53-21#cli-upgrade__svc-inst__title__1)

---

## Upgrade cpdbr Service

You must upgrade the cpdbr service after you upgrade IBM Software Hub.

**Reference**: [Updating the cpdbr service](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=uish-updating-cpdbr-service-1)

### For Environments Without Scheduling Service

```bash
cpd-cli oadp install \
--component=cpdbr-tenant \
--cpdbr-hooks-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpd \
--cpfs-image-prefix=${PRIVATE_REGISTRY_LOCATION}/cpopen/cpfs \
--namespace=${OADP_OPERATOR_NS} \
--tenant-operator-namespace=${PROJECT_CPD_INST_OPERATORS} \
--skip-recipes=true \
--upgrade=true \
--log-level=debug \
--verbose
```

**Note**: This upgrade should be performed after all services have been upgraded.

---

## RSI Patch Management

If you have any custom RSI patches that patch zen pods or IBM Cloud Pak foundational services pods, reapply the patches after the upgrade.

### Step 1: List Existing RSI Patches

Run the following command to get a list of the RSI patches in the operands project:

```bash
cpd-cli manage get-rsi-patch-info \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --all
```

**Expected Output**: List of all RSI patches with their status and configuration.

### Step 2: Reapply Custom Patches

If there are patches that apply to zen or IBM Cloud Pak foundational services pods, run the following command to apply your custom patches:

```bash
cpd-cli manage apply-rsi-patches \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

**Reference**: [IBM Documentation - Upgrading Software Hub](https://www.ibm.com/docs/en/software-hub/5.3.x?topic=upgrading)

---

## Post Upgrade Validation

### Post Upgrade Validation

```bash
# Check CR status
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}

# Check for pods not running correctly (excludes completed jobs)
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed'

# Verify CPD version
oc get ZenService lite-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.zenStatus.versions[0].version}'

# List service instances
cpd-cli service-instance list --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

---

### Post Upgrade Migrations

#### Catalog API Migration (Required when upgrading FROM 5.1.x)

**Reference**: [Completing the Catalog API migration](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=services-completing-catalog-api-migration)

**Overview**: When upgrading from Cloud Pak for Data 4.8, 5.0, or Software Hub 5.1 to 5.2.x, the back-end database for the catalog-api service is migrated from CouchDB to PostgreSQL. This migration is NOT required for 5.2.x → 5.3.x upgrades.

**Prerequisites**:
- Common core services upgrade completed
- All services upgraded successfully
- Platform validation completed

##### Step 1: Check the Migration Method Used

The migration can be either automatic or semi-automatic. Determine which method was used:

```bash
# Check migration method
oc describe ccs ccs-cr \
  --namespace ${PROJECT_CPD_INST_OPERANDS} \
  | grep use_semi_auto_catalog_api_migration
```

**Action based on response**:
- **Empty response** = Automatic migration → Proceed to Step 4 (Collecting statistics)
- **Returns "true"** = Semi-automatic migration → Proceed to Step 2

##### Step 2: Check Status of Migration Jobs (Semi-automatic only)

Monitor the migration jobs. These may take significant time depending on the number of assets:

```bash
# Check job status
oc get job cams-postgres-migration-job jobs-postgres-upgrade-migration \
  --namespace ${PROJECT_CPD_INST_OPERANDS} \
  -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[0].type,COMPLETIONS:.status.succeeded
```

**Expected output**:
```
NAME                             STATUS    COMPLETIONS
cams-postgres-migration-job      Complete   1/1
jobs-postgres-upgrade-migration  Complete   1/1
```

**Action based on status**:
- **Failed** → Contact IBM Support
- **InProgress** → Wait and check again
- **Both Complete** → Proceed to Step 3

##### Step 3: Complete the Migration (Semi-automatic only)

**Important**: Minimize database updates during this step. Large write operations will increase migration time.

```bash
# Continue semi-automatic migration
oc patch ccs ccs-cr \
  --namespace ${PROJECT_CPD_INST_OPERANDS} \
  --type merge \
  --patch '{"spec": {"continue_semi_auto_catalog_api_migration": true}}'

# Wait for CCS custom resource to complete (minimum 10 minutes, possibly longer)
oc get ccs ccs-cr --namespace ${PROJECT_CPD_INST_OPERANDS}
```

**Expected output when complete**:
```
NAME     VERSION   RECONCILED   STATUS      PERCENT   AGE
ccs-cr   11.0.0    11.0.0       Completed   100%      1d
```

**Action based on status**:
- **Failed** → Contact IBM Support
- **InProgress** → Wait and check again
- **Completed** → Proceed to Step 4

##### Step 4: Collect Migration Statistics

Create and run the migration status script:

```bash
# Create migration_status.sh script
cat > migration_status.sh << 'EOF'
#!/bin/bash

# Set postgres connection parameters
postgres_password=$(oc get secret -n ${PROJECT_CPD_INST_OPERANDS} ccs-cams-postgres-app -o json 2>/dev/null | jq -r '.data."password"' | base64 -d)
postgres_username=cams_user
postgres_db=camsdb
postgres_migrationdb=camsdb_migration

echo -e "======MIGRATION STATUS==========="

# Total migrated database(s)
databases=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_migrationdb -c "select count(*) from migration.status where state='complete'" 2>/dev/null)
if [ -n "$databases" ];then
  databases_no_space=$(echo "$databases" | tr -d ' ')
  echo "Total catalog-api databases migrated: $databases_no_space"
else
  echo "Unable to fetch migration information for databases"
fi

# Total migrated assets
assets=$(oc -n ${PROJECT_CPD_INST_OPERANDS} -c postgres exec ccs-cams-postgres-1 -- psql -t postgresql://$postgres_username:$postgres_password@localhost:5432/$postgres_db -c "select count(*) from cams.asset" 2>/dev/null)
if [ -n "$assets" ];then
  assets_no_space=$(echo "$assets" | tr -d ' ')
  echo -e "Total catalog-api assets migrated: $assets_no_space\n"
else
  echo "Unable to fetch migration information for assets"
fi
EOF

chmod +x migration_status.sh

# Run the script
./migration_status.sh
```

##### Step 5: Back Up the PostgreSQL Database

Create and run the backup script:

```bash
# Create backup_postgres.sh script
cat > backup_postgres.sh << 'EOF'
#!/bin/bash

# Make sure PROJECT_CPD_INST_OPERANDS is set
if [ -z "$PROJECT_CPD_INST_OPERANDS" ]; then
  echo "Environment variable PROJECT_CPD_INST_OPERANDS is not defined."
  exit 1
fi

echo "PROJECT_CPD_INST_OPERANDS namespace is: $PROJECT_CPD_INST_OPERANDS"

# Find the replica pod
REPLICA_POD=$(oc get pods -n $PROJECT_CPD_INST_OPERANDS -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')

if [ -z "$REPLICA_POD" ]; then
  echo "No replica pod found."
  exit 1
fi

echo "Replica pod: $REPLICA_POD"

# Extract JDBC URI from secret
JDBC_URI=$(oc get secret ccs-cams-postgres-app -n $PROJECT_CPD_INST_OPERANDS -o jsonpath="{.data.uri}" | base64 -d)

if [ -z "$JDBC_URI" ]; then
  echo "JDBC URI not found in secret."
  exit 1
fi

# Set path on the pod to save the dump file
TARGET_PATH="/var/lib/postgresql/data/forpgdump"

# Run pg_dump with nohup inside the pod
oc exec "$REPLICA_POD" -n $PROJECT_CPD_INST_OPERANDS -- bash -c "
  TARGET_PATH=\"$TARGET_PATH\"
  JDBC_URI=\"$JDBC_URI\"
  echo \"TARGET_PATH is $TARGET_PATH\"
  mkdir -p $TARGET_PATH &&
  chmod 777 $TARGET_PATH &&
  nohup bash -c '
    pg_dump $JDBC_URI -Fc -f $TARGET_PATH/cams_backup.dump > $TARGET_PATH/pgdump.log 2>&1 &&
    echo \"Backup succeeded. Please copy $TARGET_PATH/cams_backup.dump file from this pod to a safe place and delete it on this pod to save space.\" >> $TARGET_PATH/pgdump.log
  ' &
  echo \"pg_dump started in background. Logs: $TARGET_PATH/pgdump.log\"
"
EOF

chmod +x backup_postgres.sh

# Run the backup script
./backup_postgres.sh

# Set REPLICA_POD variable
REPLICA_POD=$(oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l app=ccs-cams-postgres -o jsonpath='{range .items[?(@.metadata.labels.role=="replica")]}{.metadata.name}{"\n"}{end}')

# Open remote shell to monitor backup
oc rsh ${REPLICA_POD}

# Inside the pod, monitor the backup progress
cd /var/lib/postgresql/data/forpgdump/
ls -lat

# Monitor pgdump.log file - backup is complete when you see:
# "Backup succeeded. Please copy /var/lib/postgresql/data/forpgdump/cams_backup.dump
#  file from this pod to a safe place and delete it on this pod to save space."

# Exit the pod shell
exit
```

**After backup completes**, copy the backup to a safe location:

```bash
# Set backup storage location
export POSTGRES_BACKUP_STORAGE_LOCATION=<directory>

# Copy backup from pod
oc cp ${REPLICA_POD}:/var/lib/postgresql/data/forpgdump/cams_backup.dump \
  $POSTGRES_BACKUP_STORAGE_LOCATION/cams_backup.dump

# Delete backup from replica pod
oc rsh $REPLICA_POD rm -f /var/lib/postgresql/data/forpgdump/cams_backup.dump
```

##### Step 6: Consolidate the PostgreSQL Database

Consolidate identical data across governed catalogs into single records:

```bash
# Get CPD instance URL
export INSTANCE_URL=$(cpd-cli manage get-cpd-instance-details \
  --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
  --get_admin_initial_credentials=true | grep "Cloud Pak for Data URL" | awk '{print $NF}')

# Get catalog-api-jobs pod
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep catalog-api-jobs

# Set pod name
export CAT_API_JOBS_POD=<pod-name-from-above>

# Open bash prompt in pod
oc exec ${CAT_API_JOBS_POD} -n ${PROJECT_CPD_INST_OPERANDS} -it -- bash

# Inside the pod, set AUTH_TOKEN
AUTH_TOKEN=$(cat /etc/.secrets/wkc/service_id_credential)

# Start consolidation
curl -k -X PUT "${INSTANCE_URL}/v2/shared_assets/initialize_content?bss_account_id=999" \
     -H "Authorization: Basic $AUTH_TOKEN"

# Save the transaction ID returned for debugging if needed
# Exit the pod
exit

# Get catalog-api pod
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep catalog-api | grep -v catalog-api-jobs

# Set pod name
export CAT_API_POD=<pod-name-from-above>

# Check for success message
oc logs ${CAT_API_POD} -n ${PROJECT_CPD_INST_OPERANDS} \
  | grep "Initial consolidation for bss account 999 complete"

# If empty response, check for failure messages
oc logs ${CAT_API_POD} -n ${PROJECT_CPD_INST_OPERANDS} \
  | grep "Error running initial consolidation with resource key"

oc logs ${CAT_API_POD} -n ${PROJECT_CPD_INST_OPERANDS} \
  | grep "Error executing initial consolidation for bss 999"

# If all checks return empty, wait 10 minutes and check again
```

##### Step 7: Clean Up Migration Resources (After Several Weeks)

**Important**: Wait several weeks to confirm projects, catalogs, and spaces are working correctly before cleanup.

```bash
# Delete migration pods
oc delete pod -n ${PROJECT_CPD_INST_OPERANDS} -l app=cams-postgres-migration-app

# Delete migration jobs
oc delete job -n ${PROJECT_CPD_INST_OPERANDS} -l app=cams-postgres-migration-app

# Delete migration config maps
oc delete cm -n ${PROJECT_CPD_INST_OPERANDS} -l app=cams-postgres-migration-app

# Delete migration secrets
oc delete secret -n ${PROJECT_CPD_INST_OPERANDS} -l app=cams-postgres-migration-app

# Delete migration PVC
oc delete pvc cams-postgres-migration-pvc -n ${PROJECT_CPD_INST_OPERANDS}
```

**Estimated Duration**:
- Migration jobs: 30-120 minutes (depending on asset count)
- PostgreSQL backup: 1-4 hours (depending on database size)
- Consolidation: 30-60 minutes
- **Total: 2-6 hours**

---

#### IKC Database Migration (CouchDB to PostgreSQL)

**Reference**: [Post upgrade setup for Knowledge Catalog](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrading-post-upgrade-setup-knowledge-catalog)

**Overview**: Watson Knowledge Catalog requires migration from CouchDB to PostgreSQL for improved performance and scalability.

**Prerequisites**:
- WKC service must be upgraded
- Catalog API migration completed
- Backup all catalogs and governance artifacts

**Migration Steps**:

```bash
# 1. Verify WKC upgrade completed
oc get wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.wkcStatus}'

# 2. Get WKC pod for migration
WKC_POD=$(oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l component=wkc-search -o jsonpath='{.items[0].metadata.name}')

# 3. Check if IKC database migration is required
oc exec -n ${PROJECT_CPD_INST_OPERANDS} ${WKC_POD} -- /opt/ibm/wkc/bin/check-migration-status.sh

# 4. Backup IKC database before migration
oc exec -n ${PROJECT_CPD_INST_OPERANDS} ${WKC_POD} -- /opt/ibm/wkc/bin/backup-ikc-db.sh

# 5. Run IKC database migration from CouchDB to PostgreSQL
oc exec -n ${PROJECT_CPD_INST_OPERANDS} ${WKC_POD} -- /opt/ibm/wkc/bin/migrate-couchdb-to-postgres.sh

# 6. Monitor migration progress (may take 30-120 minutes depending on data size)
oc logs -f ${WKC_POD} -n ${PROJECT_CPD_INST_OPERANDS} -c wkc-search

# 7. Verify migration completion
oc exec -n ${PROJECT_CPD_INST_OPERANDS} ${WKC_POD} -- /opt/ibm/wkc/bin/check-migration-status.sh
```

**Validation**:
```bash
# Verify WKC pods are running
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} | grep wkc

# Check WKC status
oc get wkc wkc-cr -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.status.wkcStatus}'

# Verify catalog accessibility from CPD console
oc get route cpd -n ${PROJECT_CPD_INST_OPERANDS} -o jsonpath='{.spec.host}'
# Access WKC from CPD console and verify catalogs are accessible

# Check for migration errors
oc logs ${WKC_POD} -n ${PROJECT_CPD_INST_OPERANDS} | grep -i error
```

**Estimated Duration**: 30-120 minutes (depending on catalog size and data volume)

**Important Notes**:
- Do not interrupt the migration process once started
- Ensure sufficient disk space for PostgreSQL database
- Monitor pod resources during migration
- Keep backup until migration is verified successful

---

**End of Runbook**
