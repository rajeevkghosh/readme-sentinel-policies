# Readme File for "assetmgmt_gcp_naming_validation.sentinel"

## Description

The policy enforces the use of following Resource Labels:

|Key | Valid Values | Description |
|--------|------|:-----------:|
|gcp_region|US|Geolocation scope for the project (Do not change unless you are not in US)|
|owner|Tech owner (e.g: hybridenv)|Technology owner group of the resources, for example: hybridenv (Hybrid Environment) is used for Landing Zone shared resources.|
|application_division|pci, paa, hdpa, hra, others(TBD)|Use to identify family/cluster of applications that have the same risk rating|
|application_name|(Apps to define)|Use to identify disparate resources – use up to 15 alpha-numeric characters and use Camel case for a very long name|
|application_role|app, web, auth, data|Use to describe the function of a resource|
|environment|prod, int, uat, stage, dev, test|Environment refers to:based on hierarchical stages in an application development cycle|
|au|TBD. Pull from Comments section|Hybrid code = 0223092 Other Environments - get code from your technical lead|
|created|(YYYYMMDD)|This is a dynamic field and represents the date when the resource was created|

-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan-functions" as plan
import "strings"
import "types"
```

# List of params
```
param gcp_region
param owner
param application_division
param application_name
param application_role
param environment
param au
```

# List of mandatory labels
```
mandatory_labels = {
	"gcp_region":           gcp_region,
	"owner":                owner,
	"application_division": application_division,
	"application_name":     application_name,
	"application_role":     application_role,
	"environment":          environment,
	"au":                   au,
	"created":              null,
}
```

## Get all the Resources as per resourceTypes map
```
# Fetch all resources based on resourceTypes
allResources = {}
for resourceTypes as rt, _ {
	resources = plan.find_resources(rt)
	for resources as address, rc {
		allResources[address] = rc
	}
}

```

## Working Code to Enforce policy

The code which will iterate over all the resource types mentioned in "resourceTypes" map :eg. "google_pubsub_topic/google_dataproc_cluster/google_secret_manager_secret/google_bigquery_dataset/google_storage_bucket/google_spanner_instance/google_sql_database_instance/google_kms_crypto_key" and check whether the resource Labels are defined in the following format:

|Key | Valid Values | Description |
|--------|------|:-----------:|
|gcp_region|US|Geolocation scope for the project (Do not change unless you are not in US)|
|owner|Tech owner (e.g: hybridenv)|Technology owner group of the resources, for example: hybridenv (Hybrid Environment) is used for Landing Zone shared resources.|
|application_division|pci, paa, hdpa, hra, others(TBD)|Use to identify family/cluster of applications that have the same risk rating|
|application_name|(Apps to define)|Use to identify disparate resources – use up to 15 alpha-numeric characters and use Camel case for a very long name|
|application_role|app, web, auth, data|Use to describe the function of a resource|
|environment|prod, int, uat, stage, dev, test|Environment refers to:based on hierarchical stages in an application development cycle|
|au|TBD. Pull from Comments section|Hybrid code = 0223092 Other Environments - get code from your technical lead|
|created|(YYYYMMDD)|This is a dynamic field and represents the date when the resource was created|


If the Label defined is according to the above format and constraints, the policy will pass, otherwise it will return violations.





## The code :

```
# Rule to evaluate mandatory_labels
AllLabelsExist = rule { AllLabels }

# Declare  violators  & messages Map
violators = {}
messages = {}
if AllLabelsExist {
	for allResources as address, rc {
		messages[address] = []
		violation = false
		for mandatory_labels as lk, lv {
			if types.type_of(lv) is not "null" {
				if not check_label_value(rc, lk, lv) {
					violation = true
					append(messages[address], plan.to_string(lk) + " Label with value " + plan.to_string(plan.evaluate_attribute(rc, "labels." + lk)) + " is not allowed")
				}
			}
		}

		if violation {
			violators[address] = rc
		}
	}
}

```

## The resourceTypes Map
This map contains all the resources that we need to enforce policy on. example - "google_pubsub_topic", "google_bigquery_dataset" etc.

Now since for each resources the name of the "Label" attribute can be diffrent, hence we have used key-value pair where key is the resource type and value is the name of the "label" attribute. 
This is to be noted that for some resources, the name for "label" attribute can be other than "labels".One example would be "google_sql_database_instance" for which the label attribute name is - "user_labels" and hence it has been defined accordingly in the map.

Also notice that we have mentioned "settings.0.user_labels" instead of "user_labels". This is due to, since "user_labels" are defined under settings{} block in terraform code.


```
# Resource Types to check labels
resourceTypes = {
	"google_pubsub_topic":            "labels",
	"google_secret_manager_secret":   "labels",
	"google_dataproc_cluster":        "labels",
	"google_kms_crypto_key":          "labels",
	"google_bigquery_dataset":        "labels",
	"google_compute_forwarding_rule": "labels",
	"google_storage_bucket":          "labels",
	"google_spanner_instance":        "labels",
	"google_sql_database_instance":   "settings.0.user_labels",
	"google_container_cluster":       "node_config.0.labels",
}

```
## How to Add another "Resource Type" in Above Map?
The code is made flexible enough to accomodate another resource types for which we need to enforce the naming policy.

As per our requirements, we can add more resources in the  "resourceTypes" map as shown in below example.
- In this example , we have shown how a new example resource called "google_example_resource" can be added in the "resourceTypes" map:
```
# Resource Types to check labels
resourceTypes = {
	"google_pubsub_topic":            "labels",
	"google_secret_manager_secret":   "labels",
	"google_dataproc_cluster":        "labels",
	"google_kms_crypto_key":          "labels",
	"google_bigquery_dataset":        "labels",
	"google_compute_forwarding_rule": "labels",
	"google_storage_bucket":          "labels",
	"google_spanner_instance":        "labels",
	"google_sql_database_instance":   "settings.0.user_labels",
	"google_container_cluster":       "node_config.0.labels",
    "google_example_resource":        "labels",
}
```

# Fetch all resources which dont have mandatory_labels
This function will check if the label defined for the resource are in the "mandatory_labels" map or not. 

```
for allResourcesArray as lkey, resources {
	rcs = plan.filter_attribute_not_contains_list(resources,
		lkey, keys(mandatory_labels), true)
	if length(rcs) > 0 {
		append(violatingResources, rcs)
	}
}
```

# Function to Check if Labels Values are allowed

This function will check if the label values defined in the terraform code are in the allowed value list or not.

```
check_label_value = func(rc, label_key, label_value) {
	label_key_prefix = resourceTypes[rc["type"]]
	rc_label_value = plan.evaluate_attribute(rc, label_key_prefix + "." + label_key)
	if label_value not contains rc_label_value {
		return false
	} else {
		return true
	}
}
```


## The Main Function
This function returns "False" if length of violations is not 0.

```

# Rule to evaluate mandatory_labels & label Values
GCP_RES_LABELS = rule { length(violators) is 0 and AllLabelsExist }

# Main rule
main = rule {
	GCP_RES_LABELS
}

```