# Readme File for "compute_gcp_versioning_enforce.sentinel"

## Description

### This policy has two components :
- <b>Validating whether the "release_channel" has been set as "STABLE" or not:</b>
   - The policy checks that the gke resources have "release_channel" attribute defined or if the value for "release_channel" has been defined as "STABLE" in the terraform code to provision GKE resources.If otherwise, the policy will return violations.

   
   <br>

- <b>Validating whether "datapath_provider" has been set as "ADVANCED_DATAPATH" or not:</b>
   - The policy checks that the gke resources have "datapath_provider" attribute defined or if the value for "datapath_provider" has been defined as "ADVANCED_DATAPATH" in the terraform code to provision GKE resources.If otherwise, the policy will return violations.

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

## Working Code to Enforce policy(release_channel):

The policy code will iterate over all the resource type "google_container_cluster" and check whether the "release_channel" attribute is having "null" value or if it is having value other than "STABLE".
If either of the case, the policy will return violations.


## The code :

```
violations_release_channel = {}
for allGkeInstances as address, rc {

	release_channel = plan.evaluate_attribute(rc.change.after, "release_channel")
	print(release_channel)

	isnull_release_channel = rule { types.type_of(release_channel) == "null" }
	print(isnull_release_channel)
	if isnull_release_channel {
        print("The value for  " + address + " Can't be Null ")
		violations_release_channel[address] = rc

	} else {

		if not (release_channel[0]["channel"] == "STABLE") {
            print("The value for  " + address + " can only be STABLE")
			violations_release_channel[address] = rc

		}
	}
}

GCP_GKE_RELEASECHANNEL = rule { length(violations_release_channel) is 0 }
```

## Working Code to Enforce policy(datapath_provider):

The policy code will iterate over all the resource type "google_container_cluster" and check whether the "datapath_provider" attribute is having "null" value or if it is having value other than "ADVANCED_DATAPATH".
If either of the case, the policy will return violations.


## The code :

```
violations_dataplane = {}
for allGkeInstances as address, rc {

	dataplane = plan.evaluate_attribute(rc.change.after, "datapath_provider")
	isnull_dataplane = rule { types.type_of(dataplane) == "null" }
	print("Dataplane: " + plan.to_string(isnull_dataplane))

	if isnull_dataplane {
		violations_dataplane[address] = rc
        print("The value for  " + address + " Can't be Null ")

	} else {

		if not (dataplane == "ADVANCED_DATAPATH") {
			print("For Dataplane, only ADVANCED_DATAPATH value is supported")
			violations_dataplane[address] = rc
		}
	}

}

GCP_GKE_DATAPLANEV2 = rule { length(violations_dataplane) is 0 }

```



## The Main Function
This function returns "False" if length of violations is not 0.

```
main = rule { GCP_GKE_RELEASECHANNEL and GCP_GKE_DATAPLANEV2 }

```