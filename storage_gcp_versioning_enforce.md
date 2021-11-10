# Storage_gcp_versioning_enforce.sentinel

## Description


The policy checks If versioning is enabled and  validate if a life cycle for versions is configured to avoid undesired amount of object's versions.


-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all Buckets
```
allBuckets = plan.find_resources("google_storage_bucket")
```

## Working of the Code to Enforce policy

The code which will iterate over all the resource type "google_storage_bucket" and check whether the versioning is enabled. 
If the versioning is enabled, the code will further check if "num_newer_versions" have been defined or not. Incase "num_newer_versions" have been enabled, the policy will pass, otherwise it will return violations.

## The code:

```
violations = {}
for allBuckets as address, rc {
	versioning = plan.evaluate_attribute(rc, "versioning.0.enabled")
	is_null = rule { types.type_of(versioning) is "null" }
	if not is_null {
		if versioning {
			num_versions = plan.evaluate_attribute(rc, "lifecycle_rule.0.condition.0.num_newer_versions")
			if num_versions is null {
				message = "num_newer_versions can't have null value if versioning is enabled "
				violations[address] = message
			}
		}
	}
}

print(violations)

GCP_GCS_VERSIONING = rule { length(violations) is 0 }
```
## The Main Function
This function returns "False" if length of violations is not 0.

```
GCP_GCS_VERSIONING = rule { length(violations) is 0 

```


