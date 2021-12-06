# storage_gcp_sql_versioning_enforce.sentinel

## Description


This policy is to validate whether the POSTGRESQL version mentioned in the Terraform Code is according to the allowed version or not. 


-------


## Import common-functions/tfplan-functions/tfplan-functions.sentinel with alias "plan"
```
import "tfplan/v2" as tfplan
import "tfplan-functions" as plan
import "strings"
import "types"
```

## # Get all SQL Database Instance
```
allSqlDatabaseInstance = plan.find_resources("google_sql_database_instance")
```

## Working of the Code to Enforce policy

The code which will iterate over all the resource type "google_sql_database_instance" and check whether the postgresql "database_version" is in the "allowed_database_versions" list or not. Incase, if the postgresql "database_version" is in the "allowed_database_versions" list, the policy will pass, otherwise it will return violations.

## The code:

```
violations = {}
for allSqlDatabaseInstance as address, rc {
    database_version = plan.evaluate_attribute(rc, "database_version")
    print(database_version)

    if not (database_version in allowed_database_versions) {
        violations[address] = rc
        print("The database version for: "+address+" should be from the following list: "+plan.to_string(allowed_database_versions))
    }
}


GCP_CLOUDSQL_VERSION = rule { length(violations) is 0 }

```
## The Main Function
This function returns "False" if length of violations is not 0.

```
GCP_CLOUDSQL_VERSION = rule { length(violations) is 0 }
## The Main Function
main = rule {GCP_CLOUDSQL_VERSION}

```


