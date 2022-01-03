# Readme File for "assetmgmt_gcp_resiliency_validation.sentinel"


## Description

This sentinel policy enforces usage of health check in load balancer.
The policy will check if the resource: "google_compute_region_backend_service" has "health_checks" attribute defined in the terraform code. If there is no "health_checks" attribute defined, or if the "health_checks" attribute is having empty list, the policy will return violations.

## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all the Backend Service Resources
```
backendServiceResources = plan.find_resources("google_compute_region_backend_service")

```

## Working Code to Enforce policy

The policy code will iterate over all the resource type "google_compute_region_backend_service" and check whether the "health_checks" attribute is undefined or is having empty list. If either of the case, the policy will return violations.


## The code :
```
messages = {}
for backendServiceResources as address, rc {
	health_checks = rc["change"]["after"]["health_checks"]
	if types.type_of(health_checks) is not "undefined" {
		if health_checks is null {
			messages[address] = "health_checks Needs to be enabled for google_compute_region_backend_service"
		}
	}
}

```


## The Main Function
This function returns "False" if length of messages is not 0.

```
GCP_LB_HC = rule { length(messages) is 0 }

main = rule { GCP_LB_HC }

```