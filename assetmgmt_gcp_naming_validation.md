# Readme File for "assetmgmt_gcp_naming_validation.sentinel"

## Description

The policy enforces the use of following Resource Naming convention only: [org]-[country]-[env]-*-[appid]-[uid]

One exception is "google_bigquery_dataset" for which Naming should be like this : [org]\_[country]\_[env]\_*\_[appid]\_[uid]. This is due to BigQuery resources does not allow ‘-’ in the name and ‘_’ should be used instead. 

<b>The policy also validates the constraints as mentioned below:</b>

|Section | Size | Allowed Values | Description |
|--------|------|----------------|:-----------:|
|org|2 chars|wf|Organization that belong the resource|
|country|2 chars|us|Country of the resource(obs: some resources are global)|
|env|4-7 chars|prod, nonprod, sandbox, core|Hosting environment|
|service|2-3 chars|ex: sm,gcs,vpc,bq etc|Service type|
|appid|2-6 chars|Example: msac, gcp, app01|Application ID|
|uid|2-10 chars|Example: res123|Unique ID within the service type for the specific instance|

-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all the Resources as per resourceTypesNameMap map
```
# Fetch all resources based on resourceTypes
allResources = {}
for resourceTypesNameMap as rt, _ {
	resources = plan.find_resources(rt)
	for resources as address, rc {
		allResources[address] = rc
	}
}
```

## Working Code to Enforce policy

The code which will iterate over all the resource types mentioned in "resourceTypesNameMap" :eg. "google_pubsub_topic/google_dataproc_cluster/google_secret_manager_secret/google_dialogflow_cx_agent/google_compute_interconnect_attachment/google_spanner_instance/google_sql_database_instance/google_kms_key_ring" and check whether the resource naming is in the following format: [org]-[country]-[env]-*-[appid]-[uid] or not and also if the naming is according to following constraints:

|Section | Size | Allowed Values | Description |
|--------|------|----------------|:-----------:|
|org|2 chars|wf|Organization that belong the resource|
|country|2 chars|us|Country of the resource(obs: some resources are global)|
|env|4-7 chars|prod, nonprod, sandbox, core|Hosting environment|
|service|2-3 chars|ex: sm,gcs,vpc,bq etc|Service type|
|appid|2-6 chars|Example: msac, gcp, app01|Application ID|
|uid|2-10 chars|Example: res123|Unique ID within the service type for the specific instance|

If the resource naming is according to the above format and constraints, the policy will pass, otherwise it will return violations.





## The code :

```
violators = {}
messages = {}
org_param = org
country_param = country
environment_param = environment

for allResources as address, rc {
	rt = rc["type"]
	resource_name = rc["change"]["after"][resourceTypesNameMap[rt]["name_param"]]
	resource_name_substring = resourceTypesNameMap[rt]["name_substring"]
	if rt in _naming {
		resource_prefix = check_resource_prefix(resource_name, resource_name_substring, org_param, country_param, environment_param, "_")
	} else {
		resource_prefix = check_resource_prefix(resource_name, resource_name_substring, org_param, country_param, environment_param, "-")
	}

	if types.type_of(resource_prefix) is "undefined" {
		message = plan.to_string(address) + " has " + plan.to_string(resourceTypesNameMap[rt]) + " with value " +
			plan.to_string(resource_name) +
			" that is not allowed."
		violators[address] = rc
		messages[address] = message
	} else {
		if not resource_prefix["status"] {
			violators[address] = rc
			messages[address] = strings.join(resource_prefix["message"], ".")
		}
	}
}

```

## The resourceTypesNameMap
This map contains all the resources that we need to enforce policy on. example - "google_pubsub_topic", "google_bigquery_dataset" etc.

Now since for each resources the "name" field of the naming attribute can be diffrent, hence we have used "name_param" parameter here, and, the values should be according to naming attribute mentioned in terraform registry document for that resource.

We have used another parameter to uniquely identify the resource called, "name_substring" and its values would be between 2-3 characters.


```
resourceTypesNameMap = {
	"google_pubsub_topic": {
		"name_param":     "name",
		"name_substring": "ps",
	},
	"google_secret_manager_secret": {
		"name_param":     "secret_id",
		"name_substring": "sm",
	},
	"google_dataproc_cluster": {
		"name_param":     "name",
		"name_substring": "dp",
	},
	"google_storage_bucket": {
		"name_param":     "name",
		"name_substring": "gcs",
	},
	"google_kms_key_ring": {
		"name_param":     "name",
		"name_substring": "kms",
	},
	"google_kms_crypto_key": {
		"name_param":     "name",
		"name_substring": "kms",
	},
	"google_bigquery_dataset": {
		"name_param":     "dataset_id",
		"name_substring": "bq",
	},
	"google_compute_interconnect_attachment": {
		"name_param":     "name",
		"name_substring": "ic",
	},
	"google_compute_network": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_subnetwork": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_network_peering": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_route": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_router": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_data_loss_prevention_job_trigger": {
		"name_param":     "display_name",
		"name_substring": "dlp",
	},
	"google_dialogflow_cx_agent": {
		"name_param":     "display_name",
		"name_substring": "df",
	},
	"google_compute_firewall": {
		"name_param":     "name",
		"name_substring": "fw",
	},
	"google_spanner_instance": {
		"name_param":     "name",
		"name_substring": "spn",
	},
	"google_spanner_database": {
		"name_param":     "name",
		"name_substring": "spn",
	},
	"google_compute_forwarding_rule": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_ssl_certificate": {
		"name_param":     "name",
		"name_substring": "ssl",
	},
	"google_compute_region_target_https_proxy": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_container_cluster": {
		"name_param":     "name",
		"name_substring": "gke",
	},
	"google_container_node_pool": {
		"name_param":     "name",
		"name_substring": "gke",
	},
	"google_cloudfunctions_function": {
		"name_param":     "name",
		"name_substring": "cf",
	},
	"google_logging_project_sink": {
		"name_param":     "name",
		"name_substring": "log",
	},
	"google_pubsub_subscription": {
		"name_param":     "name",
		"name_substring": "ps",
	},
	"google_scc_notification_config": {
		"name_param":     "config_id",
		"name_substring": "scc",
	},
	"google_dns_managed_zone": {
		"name_param":     "name",
		"name_substring": "dns",
	},
	"google_dns_policy": {
		"name_param":     "name",
		"name_substring": "dnsp",
	},
	"google_compute_firewall_policy": {
		"name_param":     "short_name",
		"name_substring": "fw",
	},
	"google_compute_firewall_policy_association": {
		"name_param":     "name",
		"name_substring": "fw",
	},
	"google_kms_key_ring_import_job": {
		"name_param":     "import_job_id",
		"name_substring": "kms",
	},
	"google_filestore_instance": {
		"name_param":     "name",
		"name_substring": "fs",
	},
	"google_compute_region_backend_service": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_url_map": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_health_check": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_sql_database_instance": {
		"name_param":     "name",
		"name_substring": "sql",
	},
	"google_sql_database": {
		"name_param":     "name",
		"name_substring": "sql",
	},
}

```
## How to Add another "Resource Type" in Above Map?
The code is made flexible enough to accomodate another resource types for which we need to enforce the naming policy.

As per our requirements, we can add more resources in the  "resourceTypesNameMap" as shown in below example.
- In this example , we have shown how a new example resource called "google_example_resource" can be added in the "resourceTypesNameMap" map:
```
resourceTypesNameMap = {
	"google_pubsub_topic": {
		"name_param":     "name",
		"name_substring": "ps",
	},
	"google_secret_manager_secret": {
		"name_param":     "secret_id",
		"name_substring": "sm",
	},
	"google_dataproc_cluster": {
		"name_param":     "name",
		"name_substring": "dp",
	},
	"google_storage_bucket": {
		"name_param":     "name",
		"name_substring": "gcs",
	},
	"google_kms_key_ring": {
		"name_param":     "name",
		"name_substring": "kms",
	},
	"google_kms_crypto_key": {
		"name_param":     "name",
		"name_substring": "kms",
	},
	"google_bigquery_dataset": {
		"name_param":     "dataset_id",
		"name_substring": "bq",
	},
	"google_compute_interconnect_attachment": {
		"name_param":     "name",
		"name_substring": "ic",
	},
	"google_compute_network": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_subnetwork": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_network_peering": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_route": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_compute_router": {
		"name_param":     "name",
		"name_substring": "vpc",
	},
	"google_data_loss_prevention_job_trigger": {
		"name_param":     "display_name",
		"name_substring": "dlp",
	},
	"google_dialogflow_cx_agent": {
		"name_param":     "display_name",
		"name_substring": "df",
	},
	"google_compute_firewall": {
		"name_param":     "name",
		"name_substring": "fw",
	},
	"google_spanner_instance": {
		"name_param":     "name",
		"name_substring": "spn",
	},
	"google_spanner_database": {
		"name_param":     "name",
		"name_substring": "spn",
	},
	"google_compute_forwarding_rule": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_ssl_certificate": {
		"name_param":     "name",
		"name_substring": "ssl",
	},
	"google_compute_region_target_https_proxy": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_container_cluster": {
		"name_param":     "name",
		"name_substring": "gke",
	},
	"google_container_node_pool": {
		"name_param":     "name",
		"name_substring": "gke",
	},
	"google_cloudfunctions_function": {
		"name_param":     "name",
		"name_substring": "cf",
	},
	"google_logging_project_sink": {
		"name_param":     "name",
		"name_substring": "log",
	},
	"google_pubsub_subscription": {
		"name_param":     "name",
		"name_substring": "ps",
	},
	"google_scc_notification_config": {
		"name_param":     "config_id",
		"name_substring": "scc",
	},
	"google_dns_managed_zone": {
		"name_param":     "name",
		"name_substring": "dns",
	},
	"google_dns_policy": {
		"name_param":     "name",
		"name_substring": "dnsp",
	},
	"google_compute_firewall_policy": {
		"name_param":     "short_name",
		"name_substring": "fw",
	},
	"google_compute_firewall_policy_association": {
		"name_param":     "name",
		"name_substring": "fw",
	},
	"google_kms_key_ring_import_job": {
		"name_param":     "import_job_id",
		"name_substring": "kms",
	},
	"google_filestore_instance": {
		"name_param":     "name",
		"name_substring": "fs",
	},
	"google_compute_region_backend_service": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_url_map": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_compute_region_health_check": {
		"name_param":     "name",
		"name_substring": "lb",
	},
	"google_sql_database_instance": {
		"name_param":     "name",
		"name_substring": "sql",
	},
	"google_sql_database": {
		"name_param":     "name",
		"name_substring": "sql",
	},
    "google_example_resource": {
		"name_param":     "name",
		"name_substring": "ex",
	},
}

```

## The check_org Function
This function will be called from within "check_resource_prefix" function.
This function checks if the "org" defined in the name of the resource is among the valid names mentioned in the "org" param field.
```
check_org = func(org, name) {
	if org contains name {
		return true
	} else {
		return false
	}
}
```
## The check_country Function
This function will be called from within "check_resource_prefix" function.
This function checks if the "country" defined in the name of the resource is among the valid names mentioned in the "country" param field.
```
check_country = func(country, name) {
	if country contains name {
		return true
	} else {
		return false
	}
}
```
## The check_environment Function
This function will be called from within "check_resource_prefix" function.
This function checks if the "environment" defined in the name of the resource is among the valid names mentioned in the "environment" param field.
```
check_environment = func(environment, name) {
	if environment contains name {
		return true
	} else {
		return false
	}
}
```
## The check_application_id Function
This function will be called from within "check_resource_prefix" function.
This function checks if the application_id defined in the name is between 4 and 6 characters .
```
check_application_id = func(name) {
	if length(name) >= 4 and length(name) <= 6 {
		return true
	} else {
		return false
	}
}
```
## The check_uid Function
This function will be called from within "check_resource_prefix" function.
This function checks if the "uid" defined in the name is between 2 and 10 characters .
```
check_uid = func(name) {
	if length(name) >= 2 and length(name) <= 10 {
		return true
	} else {
		return false
	}
}
```

## The check_resource_prefix Function
This code will first split the name of the resource based on the split character defined in the calling function. The split character will be either "-" or "\_".

Then the values retrieved have been passed on to the variables as below:
- org = resource_name_arr[0]
- country = resource_name_arr[1]
- environment = resource_name_arr[2]
- name_substring = resource_name_arr[3]
- application_id = resource_name_arr[4]
- uid = resource_name_arr[5]

Then further it calls below functions to check constraints. These functions are:
- check_org
- check_country
- check_environment
- check_application_id
- check_uid

If every critera is met, the policy will "pass" otherwise it will return "violations".

```
check_resource_prefix = func(name, resource_name_substring, org_param, country_param, environment_param, split_char) {

	if types.type_of(name) is not "undefined" {
		resource_name_arr = strings.split(name, split_char)
		result_map = {}
		result_map["status"] = true
		result_map["message"] = []
		if length(resource_name_arr) > 5 {
			org = resource_name_arr[0]
			country = resource_name_arr[1]
			environment = resource_name_arr[2]
			name_substring = resource_name_arr[3]
			application_id = resource_name_arr[4]
			uid = resource_name_arr[5]
			# if not check_scope(scope) {
			#  result_map["status"] = false
			#  append(result_map["message"], "Value of scope  is " + scope + " which is not allowed")
			# }
			if not check_org(org_param, org) {
				result_map["status"] = false
				append(result_map["message"], "Value of org  is " + org + " which is not allowed")
			}
			if not check_country(country_param, country) {
				result_map["status"] = false
				append(result_map["message"], "Value of country  is " + country + " which is not allowed")
			}
			if not check_environment(environment_param, environment) {
				result_map["status"] = false
				append(result_map["message"], "Value of environment  is " + environment + " which is not allowed")
			}
			if name_substring != resource_name_substring {
				result_map["status"] = false
				append(result_map["message"], "Value of resource_name_substring  is " + name_substring + " which is not allowed")
			}
			if not check_application_id(application_id) {
				result_map["status"] = false
				append(result_map["message"], "Value of application_id  is " + application_id + " which is not allowed")
			}

			if not check_uid(uid) {
				result_map["status"] = false
				append(result_map["message"], "Value of uid  is " + uid + " which is not allowed")
			}
			# if not check_system(system) {
			#  result_map["status"] = false
			#  append(result_map["message"], "Value of system  is " + system + " which is not allowed")
			# }

		} else {
			return undefined
		}
	} else {
		return undefined
	}
	return result_map
}


```


## The Main Function
This function returns "False" if length of messages is not 0.

```

GCP_RES_ID = rule { length(messages) is 0 }

# Main rule
main = rule { GCP_RES_ID }

```