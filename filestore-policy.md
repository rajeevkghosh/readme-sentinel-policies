# filestore-policy.sentinel

## Description


This policy is to ensure that  the <b>"connect_mode"</b> attribute of the <b>"google_filestore_instance"</b> resource  mentioned in the Terraform Code is <b>"PRIVATE_SERVICE_ACCESS"</b>.

<b>Note:</b> If <b>"connect_mode"</b> not provided in the terraform code, the connect mode defaults to <b> DIRECT_PEERING </b>. 


-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan/v2" as tfplan
import "tfplan-functions" as plan
import "strings"
import "types"
```

## Get all Filestore Instances
```
allFilestoreInstances = plan.find_resources("google_filestore_instance")

```

## Working of the Code to Enforce policy

The code which will iterate over all the resource type <b>"google_filestore_instance"</b> and check whether the <b>"connect_mode"</b> attribute is defined as <b>"PRIVATE_SERVICE_ACCESS"</b>. Incase, if the <b>"connect mode"</b> attribute is defined as <b>"DIRECT_PEERING"</b>, the policy will return violations.

<b>Note:</b> If "connect_mode" not provided in the terraform code, the connect mode defaults to <b> DIRECT_PEERING </b>.

## The code:

```
violations = {}
for allFilestoreInstances as address, rc {

	mode = plan.evaluate_attribute(rc.change.after, "networks.0.connect_mode")

	is_mode_undefined = rule { types.type_of(mode) == "null" or types.type_of(mode) == "undefined" }

	if is_mode_undefined {

		print("The value for connect_mode for  " + address + " can't be null")
		violations[address] = rc

	} else {

		if not (mode == "PRIVATE_SERVICE_ACCESS") {
			print("The value for connect_mode for  " + address + " can only be PRIVATE_SERVICE_ACCESS")
			violations[address] = rc
		}
	}

}

GCP_FILESTORE_ACCESS = rule { length(violations) is 0 }

```
## The Main Function
This function returns <b>"False"</b> if length of violations is not 0.

```
## The Main Function
main = rule { GCP_FILESTORE_ACCESS }

```


