---
name: clusters
description: Manage Red Hat OpenShift clusters via API
api: openapi/red-hat-clusters-api-openapi.yml
operations:
  - listClusters
  - createCluster
  - getCluster
  - updateCluster
  - deleteCluster
---
# Cluster Management Skill
This skill provides steps to list, create, retrieve, update, and delete OpenShift clusters using the Red Hat Clusters API.

## Steps
1. **List Clusters**: Call the `listClusters` operation to retrieve all clusters.
2. **Create Cluster**: Use `createCluster` with required payload to provision a new cluster.
3. **Get Cluster**: Invoke `getCluster` with the cluster ID to fetch details.
4. **Update Cluster**: Call `updateCluster` with modifications.
5. **Delete Cluster**: Use `deleteCluster` to remove a cluster.
