# Readme File for "iam_gcp_identity_enforce.sentinel"

## Description

The policy checks that the gke resources have "workload_identity_config" attribute defined and if the value for "workload_identity_config" has been defined as "${data.google_project.project.project_id}.svc.id.goog" in the terraform code to provision GKE resources or not.

If otherwise, the policy will return violations.

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

## Working Code to fetch the "${data.google_project.project.project_id}.svc.id.goog" from project datasource:
The code will check wither the datasource has been defined for project. If the datasource for project is defined, it will further check the "project_id".
This will help us to get the "workload_pool_var" as mentioned below:

```
workload_pool_var = plan.to_string(project_id) + ".svc.id.goog"
```

## The code :

```
is_null_datasource = rule { types.type_of(datasource) == "undefined" }

workload_pool_var = ""
if not is_null_datasource {
	projects = filter tfplan.raw.prior_state.values.root_module.resources as _, rc {
		rc.type is "google_project"
	}

	project_id = ""
	for projects as address, rc {
		project_id = plan.evaluate_attribute(rc, "values.project_id")
	}

	workload_pool_var = plan.to_string(project_id) + ".svc.id.goog"
}
```

## Working Code to Enforce policy(workload_identity_config):

The policy code will iterate over all the resource type "google_container_cluster" and check whether the "workload_identity_config" attribute is having "null" value or if it is having value other than "${data.google_project.project.project_id}.svc.id.goog".
If either of the case, the policy will return violations.


## The code :

```
violations_workload_pool = {}
for allGkeInstances as address, rc {

	workload_pool = plan.evaluate_attribute(rc.change.after, "workload_identity_config")
	print(workload_pool)

	isnull_workload_pool = rule { types.type_of(workload_pool) == "null" }
	#print(isnull_workload_pool)
	if isnull_workload_pool {
		print("The value for  " + address + " Can't be Null ")
		violations_workload_pool[address] = rc

	} else {
		if workload_pool_var == "" {
			violations_workload_pool[address] = rc
			print("For resource: " + address + " Please define data source for resource type: google_project")
		} else {
			if not (workload_pool[0]["workload_pool"] == workload_pool_var) {
				print("For the resource: " + plan.to_string(address) + " The value for workload_identity_config.workload_pool can only be ${data.google_project.project.project_id}.svc.id.goog")
				violations_workload_pool[address] = rc

			}
		}
	}
}

GCP_GKE_WORLOADIDENT = rule { length(violations_workload_pool) is 0 }

```



## The Main Function
This function returns "False" if length of violations is not 0.

```
main = rule { GCP_GKE_WORLOADIDENT }

```