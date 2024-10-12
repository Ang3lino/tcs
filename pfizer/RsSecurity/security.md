
# Redshift 
# Row Level Security

Redshift RLS allows for granular control over sensitive data by defining policies that restrict access to specific rows in tables. This works alongside column-level security for added control.

When creating RLS policies, we recommend that you create simple policies and avoid complex statements in policies. 

**Benefits:**
- Limit access to sensitive data based on user roles.
- No need to modify queries with additional conditions.

**Things to Consider:**
- Keep policies simple to avoid performance issues.
- Joins in policies can impact performance.
- Users can only update/delete rows they can see.

**How it Works:**
- Policies are defined using SQL expressions.
- Policies are attached to users, roles, or tables.
- RLS applies to SELECT, UPDATE, and DELETE statements.
- Multiple policies can be combined with AND or OR logic.

**Management:**
- Superusers, security admins, or users with `sys:secadmin` can manage policies.
- Use `ALTER TABLE` to turn RLS on/off for tables.
- Specific statements are used to create, alter, attach/detach, and drop policies.
- System views exist to monitor and troubleshoot RLS.

**Special Permissions:**
- `IGNORE RLS` allows users to query tables with policies without seeing them.
- `EXPLAIN RLS` allows users to view RLS policy filters in query explanations.


## Using RLS policies in SQL statements
[+] https://docs.aws.amazon.com/redshift/latest/dg/t_rls_statements.html
When using RLS policies in SQL statements, Redshift applies the following rules:

- Redshift applies RLS policies to the SELECT, UPDATE, and DELETE statements by default.
- For SELECT and UNLOAD, Amazon Redshift filters rows according to your defined policy.
- For UPDATE and DELETE, Redshift updates only the rows that are visible to you. For TRUNCATE, you can still truncate the table.
- For CREATE TABLE LIKE, tables created with the LIKE options won't inherit permission settings from the source table. 

## Combining multiple policies

RLS in Amazon Redshift supports attaching multiple policies per user and object. When there are multiple policies defined for a user, Amazon Redshift applies all the policies with either AND or OR syntax depending on the RLS CONJUNCTION TYPE setting for the table. For more information about conjunction type, see ALTER TABLE.

```sql
-- Create an analyst role and grant it to a user named Alice.
CREATE ROLE analyst;
CREATE USER alice WITH PASSWORD 'Name_is_alice_1';
GRANT ROLE analyst TO alice;

-- Create an RLS policy that only lets the user see concerts.
CREATE RLS POLICY policy_concerts
    WITH (catgroup VARCHAR(10))
    USING (catgroup = 'Concerts');

-- Create an RLS policy that only lets the user see sports.
CREATE RLS POLICY policy_sports
    WITH (catgroup VARCHAR(10))
    USING (catgroup = 'Sports');

-- Attach both to the analyst role.
ATTACH RLS POLICY policy_concerts ON category TO ROLE analyst;
ATTACH RLS POLICY policy_sports ON category TO ROLE analyst;

-- Activate RLS on the category table with OR CONJUNCTION TYPE. 
ALTER TABLE category ROW LEVEL SECURITY ON CONJUNCTION TYPE OR;

-- Change session to Alice.
SET SESSION AUTHORIZATION alice;

-- Select all from the category table.
SELECT catgroup, count(*)
FROM category
GROUP BY catgroup ORDER BY catgroup;

 catgroup | count 
---------+-------
 Concerts |  3
 Sports   |  5
(2 rows)
```

[+] ALTER TABLE https://docs.aws.amazon.com/redshift/latest/dg/r_ALTER_TABLE.html
```sql
ALTER TABLE table_name {
    ADD table_constraint
    | DROP CONSTRAINT constraint_name [ RESTRICT | CASCADE ]
    [...]
    | ROW LEVEL SECURITY { ON | OFF } [ CONJUNCTION TYPE { AND | OR } ] [ FOR DATASHARES ] }
```

## RLS policy ownership and management

A user that has the sys:secadmin role, can create, modify, or manage all RLS policies for tables. At the object level, you can turn row-level security on or off without modifying the schema definition for tables.

The statements used to manage RLS in Redshift are:

- **ALTER TABLE:** Turns RLS on or off for a table.
- **[CREATE| ALTER| ATTACH| DETACH| DROP] RLS POLICY:** Applies the proper action a new RLS policy for one or more tables and specifies users or roles.
- **GRANT and REVOKE:** Grants or revokes SELECT permissions for RLS policies referencing lookup tables.

**Redshift sys tables**
For the below, you need to have sys:secadmin permission.

 Use SVV_RLS_POLICY and SVV_RLS_ATTACHED_POLICY to monitor the policies created.
 Use SVV_RLS_RELATION to list RLS-protected relations.
 Use SVV_RLS_APPLIED_POLICY to trace the application of RLS policies on queries that reference RLS-protected relations, a superuser, sys:operator, or any user. Note that sys:secadmin is not granted these permissions by default.

Special Permissions:

- IGNORE RLS allows users to query tables with policies without seeing them.
- EXPLAIN RLS allows users to view RLS policy filters in query explanations.


# Dynamic Data Masking review

With DDM, you can obfuscate data in Redshift to certain columns under certain criteria to users or groups. This can be custom whether using SQL, Python or AWS Lambda.

## SQL cmds for handling DDM

You can perform CREATE, ALTER, ATTACH, DETACH and DROP actions for DDM. Generally we will be using CREATE and ATTACH. Below the syntax: 

```sql
CREATE MASKING POLICY 
   policy_name [IF NOT EXISTS]
   WITH (input_columns)
   USING (masking_expression);

ATTACH MASKING POLICY policy_name
   ON { relation_name }
   ( {output_columns_names | output_path} ) [ USING ( {input_column_names | input_path )} ]
   TO { user_name | ROLE role_name | PUBLIC }
   [ PRIORITY priority ];
```

Similar to define function, we define the policy name with their input columns (args) and the mask expression (body). When attaching we pass the relation and column names and define the role we will be using. 
We can define which rows can be masked using the masking expression.
[+] https://docs.aws.amazon.com/redshift/latest/dg/r_ddm-procedures.html

Redshift tables to audit DDM: 

- Use SVV_MASKING_POLICY to view all masking policies created on the cluster or workgroup.
- Use SVV_ATTACHED_MASKING_POLICY to view all the relations and users or roles with policies attached on the currently connected database.
- Use SYS_APPLIED_MASKING_POLICY_LOG to trace the application of masking policies on queries that reference DDM-protected relations.

One consideration that stands out is that DDM can't be used with data sharing. If we mask on the producer the consumer won't be able to read from these tables. 
To see all considerations
[+] https://docs.aws.amazon.com/redshift/latest/dg/t_ddm-considerations.html

End-to-end example: https://docs.aws.amazon.com/redshift/latest/dg/ddm-example.html



# Dynamic Data Masking (DDM) Overview

Dynamic Data Masking (DDM) in Amazon Redshift allows you to obfuscate sensitive data in specific columns based on user or group permissions. This enables fine-grained control over who can view sensitive information, providing an additional layer of security. DDM can be applied using SQL commands, Python, or AWS Lambda, offering flexibility in implementation.

## Key SQL Commands for Managing DDM

To manage DDM in Redshift, you can use the following actions: `CREATE`, `ALTER`, `ATTACH`, `DETACH`, and `DROP`. In most cases, you’ll primarily use `CREATE` and `ATTACH` to establish and apply masking policies.

### Example SQL Syntax:

```sql
-- Create a masking policy
CREATE MASKING POLICY 
   policy_name [IF NOT EXISTS]
   WITH (input_columns)
   USING (masking_expression);

-- Attach a masking policy to specific columns in a table
ATTACH MASKING POLICY policy_name
   ON { relation_name }
   ( {output_columns_names | output_path} ) 
   [ USING ( {input_column_names | input_path } ) ]
   TO { user_name | ROLE role_name | PUBLIC }
   [ PRIORITY priority ];
```

### Explanation:

- **CREATE MASKING POLICY**: Defines a policy with input columns (parameters) and a masking expression (logic).
- **ATTACH MASKING POLICY**: Links the policy to specific columns in a table, assigning roles or users who will have the masking applied.

Masking expressions allow you to customize which rows are masked, depending on the criteria you set.

For more details, refer to the official documentation:  
[Dynamic Data Masking Procedures in Redshift](https://docs.aws.amazon.com/redshift/latest/dg/r_ddm-procedures.html)

## Auditing DDM in Redshift

Redshift provides system views to audit and manage your DDM configurations:

- **SVV_MASKING_POLICY**: Lists all the masking policies created in the cluster or workgroup.
- **SVV_ATTACHED_MASKING_POLICY**: Displays the relations, users, or roles to which policies are attached in the connected database.
- **SYS_APPLIED_MASKING_POLICY_LOG**: Logs the application of masking policies for queries that involve DDM-protected data.

## Considerations for Using DDM

- **Data Sharing Limitation**: DDM is not compatible with Redshift’s data sharing feature. If masking is applied on a producer cluster, the consumer cluster will not be able to read the masked tables.
  
For a full list of considerations, visit the documentation:  
[DDM Considerations](https://docs.aws.amazon.com/redshift/latest/dg/t_ddm-considerations.html)

## End-to-End Example

For a complete implementation example, check out the official guide:  
[DDM Example](https://docs.aws.amazon.com/redshift/latest/dg/ddm-example.html)

---

# Amazon DataZone

Amazon DataZone is a data management service that makes it faster and easier for you to catalog, discover, share, and govern data stored across AWS, on-premises, and third-party sources. 

### Amazon DataZone: Key Capabilities

* **Data Governance:** Ensures right data access for right users based on organizational policies. Provides transparency on data usage and subscription approvals. Monitors data assets across projects.
* **Collaboration:** Enables seamless collaboration across teams with self-service access to data and analytics tools. Uses business terms for data search, sharing, and access. Provides business glossaries for data understanding.
* **Automated Data Discovery & Cataloging:** Reduces manual data attribute entry with machine learning. Improves data searchability through richer data in the catalog.

### Amazon DataZone Integrations

* **Producer Data Sources:** Publishes data assets from AWS Glue Data Catalog, Amazon Redshift tables/views, and Amazon S3 objects.
* **Consumer Tools:** Analyzes data assets using Amazon Athena or Amazon Redshift query editors.
* **Access Control & Fulfillment:** Grants access to AWS Lake Formation managed tables and views. Publishes events for custom integrations with other AWS services or third-party solutions.

### Accessing Amazon DataZone

* **Amazon DataZone Console:** Manages domains, blueprints, and users. URL: [https://console.aws.amazon.com/datazone](https://console.aws.amazon.com/datazone)
* **Amazon DataZone Data Portal:** Catalogs, discovers, governs, shares, and analyzes data. Authentication via IAM Identity Center or IAM credentials. URL obtainable from console.
* **Amazon DataZone HTTPS API:** Programmatic access via HTTPS requests. See Amazon DataZone API Reference for details.

## Terminology
### Amazon DataZone Summary

Amazon DataZone is a data management service offered by AWS that simplifies data governance, discovery, sharing, and analysis across your organization. It helps you catalog data from various sources (AWS, on-premises, and third-party) and provides tools for users to find, access, and collaborate on data projects.

Here are the key functionalities of Amazon DataZone:

* **Data Cataloging:** Organize and catalog your data assets with business context for easy discovery.
* **Data Governance:** Implement fine-grained access control to ensure users only access authorized data.
* **Data Sharing:** Securely share data between producers and consumers through self-service workflows.
* **Data Discovery:** Enable users to discover relevant data assets based on their needs.
* **Data Collaboration:** Facilitate collaboration on data projects through business glossaries and metadata forms.

Amazon DataZone consists of several components:

* **Business Data Catalog:** Catalog data with business context for easy searchability.
* **Publish and Subscribe Workflows:** Automate data sharing between producers and consumers.
* **Projects and Environments:** Create project workspaces for teams to collaborate on data use cases.
* **Data Portal:** A web application for users to access, manage, and analyze data assets.

Here are some additional details about Amazon DataZone:

* **Domains and User Access:** Organize your assets, users, and projects using domains. You can associate additional AWS accounts with your domain to bring together your data sources.
* **Data Inventory and Publishing Workflows:** Define data inventory for your projects and publish them to the domain catalog for broader access.
* **Subscription and Fulfillment Workflows:** Users can subscribe to assets and receive automated access through fulfillment workflows.
* **Security:**  Amazon DataZone offers fine-grained access control and security features like monitoring and auditing.

Overall, Amazon DataZone is a comprehensive solution for managing and governing your data assets within AWS and across hybrid environments.
