# Readme File for "network_gcp_region_restricition.sentinel"

## Description


The policy checks that resources(eg. PubSub/BigQuery/Dataproc/SecretManager/DialogFlow) need to be created in US region only.


-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all the Resources as per resourceTypesRegionMap map
```
for resourceTypesRegionMap as rt, _ {
	resources = plan.find_resources(rt)
	for resources as address, rc {
		allResources[address] = rc
	}
}
```

## Working Code to Enforce policy

The code which will iterate over all the resource type "google_pubsub_topic/google_bigquery_dataset/google_dataproc_cluster/google_secret_manager_secret/google_dialogflow_cx_agent" and check whether the resource is associated with US region only or not. 
If the resource is belonging to US then the policy will be passed, otherwise it will return violations.

## The code :

```
msgs = {}
for allResources as address, rc {
	msg = check_for_location(
		address,
		rc,
		resourceTypesRegionMap[rc["type"]]["key"],
		resourceTypesRegionMap[rc["type"]]["location_key"],
	)
	if length(msg) > 0 {
		msgs[address] = msg
	}
}
```

## The resourceTypesRegionMap
This map contains all the topics that we need to enforce policy on. example - "google_pubsub_topic", "google_bigquery_dataset" etc.

Now since for each topic the location key can be diffrent, hence we have used "key" parameter here and the values can be either "location" or the specific values where the location is specified in the mock( for example for "google_topic_pubsub", the value would be ""message_storage_policy.0.allowed_persistence_regions").

For "google_secret_manager_secret", since there can be multiple occurances of secret manager keys and locations, we have introduced "location_key" parameter to retrieve location for each occurance of keys.


```
resourceTypesRegionMap = {
	"google_pubsub_topic": {
		"key": "message_storage_policy.0.allowed_persistence_regions",
	},
	"google_bigquery_dataset": {
		"key": "location",
	},
	"google_dataproc_cluster": {
		"key": "region",
	},
	"google_secret_manager_secret": {
		"key":          "replication.0.user_managed.0.replicas",
		"location_key": "location",
	},
	"google_dialogflow_cx_agent": {
		"key": "location",
	},
	 "google_compute_interconnect_attachment": {
		"key": "region",
	},
}
```
## How to Add another "Resource Type" in Above Map?
The code is made flexible enough to accomodate another resource types for which we need to restrict the locations.
As per our requirements, we can add more resources in the  "resourceTypesRegionMap" as shown in below example.
- In this example , we have shown how a new example resource called "google_example_resource" can be added in the "resourceTypesRegionMap" map:
```
resourceTypesRegionMap = {
	"google_pubsub_topic": {
		"key": "message_storage_policy.0.allowed_persistence_regions",
	},
	"google_bigquery_dataset": {
		"key": "location",
	},
	"google_dataproc_cluster": {
		"key": "region",
	},
	"google_secret_manager_secret": {
		"key":          "replication.0.user_managed.0.replicas",
		"location_key": "location",
	},
    "google_dialogflow_cx_agent": {
		"key": "location",
	},
	 "google_compute_interconnect_attachment": {
		"key": "region",
	},
    "google_example_resource": {
		"key": "location",
	},
}
```

## The check_for_location Function
This code will first check if the location is string or not. If this is string, it will convert it to array. otherwise code will call another function called "get_list_element" to fetch the location from the list or map.
incase the returned value from "get_list_element" function is a map, the  check_for_location function will further iterate it over and get the locations from the map.

Once we have all the location elements fetched from above two functions, it will call another function called "check_for_matches" which will perform the validation as whether the location retrieved are defined in "prefix" or "multi_region_var variables. if there's some violations, the main function will return false.

```
check_for_location = func(address, rc, location_key_param, location_key) {
	violations = {}
	locations = []
	location = plan.evaluate_attribute(rc.change.after, location_key_param)

	is_undefined = rule { types.type_of(location) is "undefined" }
	if is_undefined {
		violations[address] = rc
	} else {
		is_type = types.type_of(location)
		if is_type is "string" {
			append(locations, location)
		} else if is_type is "list" {
			if get_list_element_type(location) is "string" {
				locations = location
			} else if get_list_element_type(location) is "map" {
				for location as each_map {
					each_location = plan.evaluate_attribute(each_map, location_key)
					append(locations, each_location)
				}
			}
		}
		match = check_for_matches(locations, address, rc)
		violations = match
	}
	return violations
}


```
## The Main Function
This function returns "False" if length of violations is not 0.

```
GCP_RES_LOCATION = rule { length(msgs) is 0 }

```