# Readme File for "iam_gcp_policy_restricitions.sentinel"

## Description


The policy will only allow Groups and Service Accounts within IAM policies. Also the policy will not allow custom IAM roles


-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan". Also importing necessary in-built sentinel functions
```
import "tfplan-functions" as plan
import "strings"
import "types"
import "json"
```

## Get all the Resources as per resourceTypesRegionMap map
```
allResources = {}
for resourceTypesIamMap as rt, _ {
	resources = plan.find_resources(rt)
	for resources as address, rc {
		allResources[address] = rc
	}
}
```

## Working Code to Enforce policy
## - The code as Two parts :
* GCP_IAM_ENTITY
     - This validates whether the iam policies only contains "Groups" or "Service Accounts". 
* GCP_IAM_CUSTOM
     - This part validates whether any custom iam role is defined. If any custom iam role is found, it will return violations
------

 # Working of  GCP_IAM_ENTITY Component

The code which will iterate over all the resource type "google_project_iam_policy/google_project_iam_binding/google_project_iam_member/google_organization_iam_policy/google_organization_iam_binding/google_organization_iam_member" and check whether the iam member is part of "Group" or "Service Accounts". If any "User" account is found, it will return violations.

## The code :

```
msgs = {}
for allResources as address, rc {
	type = rc["type"]
	for resourceTypesIamMap[type]["keys"] as mkey {
		msg = check_for_member(
			address,
			rc,
			mkey,
		)
		#print(violations)
		if length(msg) > 0 {
			msgs[address] = msg
		}
	}

}
```

## The check_for_member Function
This function will check if the memmer/members are of string type or if its list.
The function fetches all the members defined in the Terraform code for Iam policy and further validates if they are part of "groups" or "service Accounts" as mentioned in "allowed_prefix" variable.
incase if any of the member/members are found to be  "user", it returns violations.

```
violations = {}
check_for_member = func(address, rc, mkey) {
	members_list = []

	member = plan.evaluate_attribute(rc.change.after, mkey)
	is_list = rule { types.type_of(member) is "list" }
	is_string = rule { types.type_of(member) is "string" }
	if is_string and strings.has_prefix(member, "{") {

		pol_data = json.unmarshal(member)
		for pol_data as ad, lp {
			for lp as rt {
				mem = plan.evaluate_attribute(rt, "members")
				members_list = mem
			}
		}
	} else {

		if is_string {
			append(members_list, member)

		}
		if is_list {
			members_list = member

		}

	}
	for members_list as lst {
		print(lst)
		if not (strings.has_prefix(lst, allowed_prefix[0]) or strings.has_prefix(lst, allowed_prefix[1])) {
			violations[address] = rc
		}
	}

	return violations
}

```
---------
---------

# Working of GCP_IAM_CUSTOM Component
The code will iterate over all the resources and check if the mock file contains any resource type mentioned in { deny_services = ["google_project_iam_custom_role", "google_organization_iam_custom_role"] } variable.
### If any resource type mentioned in "deny_services" list is found, the code will return violations.

## The Code:

```
# Fetch all resources based on deny_services
allRes = {}
for deny_services as rt {
	resources = plan.find_resources(rt)
	for resources as address, rc {
		allRes[address] = rc
	}
}


# Fetch all resources which are not allowed as per deny services list
violatingResources = plan.filter_attribute_not_contains_list(allRes,
	"type", deny_services, true)
print(violatingResources)

GCP_IAM_CUSTOM = rule { length(violatingResources["messages"]) is 0 }
```



## The Main Rule
The main function returns "false" if the value of any of the Rule - GCP_IAM_ENTITY or GCP_IAM_CUSTOM is false.

```
# Main rule
main = rule { GCP_IAM_ENTITY and GCP_IAM_CUSTOM }

```