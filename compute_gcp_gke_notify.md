# Readme File for "compute_gcp_gke_notify.sentinel"

## Description

### This policy has two components :
- <b>Validating the master_authorized_networks_config block:</b>
   - The policy checks that the gke resources have "master_authorized_networks_config" block defined. If the terraform code doesn't have "master_authorized_networks_config" block defined, the policy will notify it.
   
   <br>

- <b>Validating the master_ipv4_cidr_block settings:</b>
   - The policy checks that the gke resources have "master_ipv4_cidr_block " block defined. If the terraform code doesn't have "master_ipv4_cidr_block" block defined, the policy will notify it.

-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan/v2" as tfplan
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all the GKE Instances
```
allGkeInstances = plan.find_resources("google_container_cluster")

```

## Working Code to Enforce policy(master_authorized_networks_config):

The policy code will iterate over all the resource type "google_container_cluster" and check whether the "master_authorized_networks_config" block is having empty list or if "cidr_block" is having null value.
If either of the case, the policy will return violations.


## The code :

```
violations_masterauth = {}
for allGkeInstances as address, rc {

	master_auth = plan.evaluate_attribute(rc.change.after, "master_authorized_networks_config")

	isnull_master_auth = rule { length(master_auth) == 0 or master_auth[0]["cidr_blocks"] == null }

	if isnull_master_auth {
		violations_masterauth[address] = rc
		print("master_authorized_networks_config value can't be Null")

	}

}

GCP_GKE_MASTERAUTH = rule { length(violations_masterauth) is 0 }
```

## Working Code to Enforce policy(master_ipv4_cidr_block settings):

The policy code will iterate over all the resource type "google_container_cluster" and check whether the "master_ipv4_cidr_block" is having empty "string" or if its having "null" or "undefined" value.
If either of the case, the policy will return violations.


## The code :

```
violations_master_cidr = {}
for allGkeInstances as address, rc {

    key = "private_cluster_config.0.master_ipv4_cidr_block"
	selected_node = plan.evaluate_attribute(rc.change.after, key)
	
    if types.type_of(selected_node) is "null" or types.type_of(selected_node) is "undefined" or selected_node is "" {
        
        violations_master_cidr[address] = rc
		print(key + " can not be null/blank.")
    }   
    
}

GCP_GKE_PRIVATECLUSTER = rule { length(violations_master_cidr) is 0 }

```



## The Main Function
This function returns "False" if length of violations is not 0.

```
main = rule { GCP_GKE_MASTERAUTH and GCP_GKE_PRIVATECLUSTER }

```