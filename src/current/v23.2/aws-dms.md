---
title: Migrate with AWS Database Migration Service (DMS)
summary: Learn how to use AWS Database Migration Service (DMS) to migrate data to a CockroachDB target cluster.
toc: true
docs_area: migrate
---

This page has instructions for setting up [AWS Database Migration Service (DMS)](https://aws.amazon.com/dms/) to migrate data to CockroachDB from an existing, publicly hosted database containing application data, such as PostgreSQL, MySQL, Oracle, or Microsoft SQL Server.

For a detailed tutorial about using AWS DMS and information about specific migration tasks, see the [AWS DMS documentation site](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html).

For any issues related to AWS DMS, aside from its interaction with CockroachDB as a migration target, contact [AWS Support](https://aws.amazon.com/contact-us/).

{{site.data.alerts.callout_info}}
Using CockroachDB as a source database within AWS DMS is unsupported.
{{site.data.alerts.end}}

## Before you begin

Complete the following items before starting this tutorial:

- Configure a [replication instance](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_ReplicationInstance.Creating.html) in AWS.
- Configure a [source endpoint](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Source.html) in AWS pointing to your source database.
- Ensure you have a secure, publicly available CockroachDB cluster running the latest **{{ page.version.version }}** [production release](https://www.cockroachlabs.com/docs/releases/), and have created a [SQL user]({% link {{ page.version.version }}/security-reference/authorization.md %}#sql-users) that you can use for your AWS DMS [target endpoint](#step-1-create-a-target-endpoint-pointing-to-cockroachdb).
    - Set the following [session variables]({% link {{ page.version.version }}/set-vars.md %}#supported-variables) using [`ALTER ROLE ... SET {session variable}`]({% link {{ page.version.version }}/alter-role.md %}#set-default-session-variable-values-for-a-role):

        {% include_cached copy-clipboard.html %}
        ~~~ sql
        ALTER ROLE {username} SET copy_from_retries_enabled = true;
        ~~~

        {% include_cached copy-clipboard.html %}
        ~~~ sql
        ALTER ROLE {username} SET copy_from_atomic_enabled = false;
        ~~~

        This prevents a potential issue when migrating especially large tables with millions of rows.

- If you are migrating to a CockroachDB {{ site.data.products.cloud }} cluster and plan to [use replication as part of your migration strategy](#step-2-1-task-configuration), you must first **disable** [revision history for cluster backups]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %}) for the migration to succeed.
    {{site.data.alerts.callout_danger}}
    You will not be able to run a [point-in-time restore]({% link {{ page.version.version }}/take-backups-with-revision-history-and-restore-from-a-point-in-time.md %}#point-in-time-restore) as long as revision history for cluster backups is disabled. Once you [verify that the migration succeeded](#step-3-verify-the-migration), you should re-enable revision history.
    {{site.data.alerts.end}}

    - If the output of [`SHOW SCHEDULES`]({% link {{ page.version.version }}/show-schedules.md %}) shows any backup schedules, run [`ALTER BACKUP SCHEDULE {schedule_id} SET WITH revision_history = 'false'`]({% link {{ page.version.version }}/alter-backup-schedule.md %}) for each backup schedule.
    - If the output of `SHOW SCHEDULES` does not show backup schedules, [contact Support](https://support.cockroachlabs.com) to disable revision history for cluster backups.
- Manually create all schema objects in the target CockroachDB cluster. AWS DMS can create a basic schema, but does not create indexes or constraints such as foreign keys and defaults.
    - If you are migrating from PostgreSQL, MySQL, Oracle, or Microsoft SQL Server, [use the **Schema Conversion Tool**](https://www.cockroachlabs.com/docs/cockroachcloud/migrations-page) to convert and export your schema. Ensure that any schema changes are also reflected on your tables, or add [transformation rules](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TableMapping.SelectionTransformation.Transformations.html). If you make substantial schema changes, the AWS DMS migration may fail.

    {{site.data.alerts.callout_info}}
    All tables must have an explicitly defined primary key. For more guidance, see the [Migration Overview]({% link {{ page.version.version }}/migration-overview.md %}#schema-design-best-practices).
    {{site.data.alerts.end}}

As of publishing, AWS DMS supports migrations from these relational databases (for a more accurate view of what is currently supported, see [Sources for AWS DMS](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Introduction.Sources.html)):

- Amazon Aurora
- Amazon DocumentDB (with MongoDB compatibility)
- Amazon S3
- IBM Db2 (LUW edition only)
- MariaDB
- Microsoft Azure SQL
- Microsoft SQL Server
- MongoDB
- MySQL
- Oracle
- PostgreSQL
- SAP ASE

## Step 1. Create a target endpoint pointing to CockroachDB

1. In the AWS Console, open **AWS DMS**.
1. Open **Endpoints** in the sidebar. A list of endpoints will display, if any exist.
1. In the top-right portion of the window, select **Create endpoint**.
    <img src="{{ 'images/v23.2/aws-dms-create-endpoint.png' | relative_url }}" alt="AWS-DMS-Create-Endpoint" style="max-width:100%" />

    A configuration page will open.
1. In the **Endpoint type** section, select **Target endpoint**.
1. Supply an **Endpoint identifier** to identify the new target endpoint.
1. In the **Target engine** dropdown, select **PostgreSQL**.
1. Under **Access to endpoint database**, select **Provide access information manually**. 

    For information about where to find CockroachDB connection parameters, see [Connect to a CockroachDB Cluster]({% link {{ page.version.version }}/connect-to-the-database.md %}).
1. Enter the **Server name** and **Port** of your CockroachDB cluster.
1. Supply a **User name**, **Password**, and **Database name** from your CockroachDB cluster.
    {{site.data.alerts.callout_info}}
    To connect to a CockroachDB {{ site.data.products.serverless }} cluster, set the **Database name** to `{serverless-hostname}.{database-name}`. For details on how to find these parameters, see [Connect to a CockroachDB Serverless cluster](https://www.cockroachlabs.com/docs/cockroachcloud/connect-to-a-serverless-cluster.html?filters=connection-parameters#connect-to-your-cluster). Also set **Secure Socket Layer (SSL) mode** to **require**.
    {{site.data.alerts.end}}
    <img src="{{ 'images/v23.2/aws-dms-endpoint-configuration.png' | relative_url }}" alt="AWS-DMS-Endpoint-Configuration" style="max-width:100%" />
1. If needed, you can test the connection under **Test endpoint connection (optional)**.
1. To create the endpoint, select **Create endpoint**.
    <img src="{{ 'images/v23.2/aws-dms-test-endpoint.png' | relative_url }}" alt="AWS-DMS-Test-Endpoint" style="max-width:100%" />

## Step 2. Create a database migration task

A database migration task, also known as a replication task, controls what data are moved from the source database to the target database.

### Step 2.1. Task configuration

1. While in **AWS DMS**, select **Database migration tasks** in the sidebar. A list of database migration tasks will display, if any exist.
1. In the top-right portion of the window, select **Create task**.
    <img src="{{ 'images/v23.2/aws-dms-create-db-migration-task.png' | relative_url }}" alt="AWS-DMS-Create-DB-Migration-Task" style="max-width:100%" />

    A configuration page will open.
1. Supply a **Task identifier** to identify the replication task.
1. Select the **Replication instance** and **Source database endpoint** you created prior to starting this tutorial.
1. For the **Target database endpoint** dropdown, select the CockroachDB endpoint created in the previous section.
1. Select the appropriate **Migration type** based on your needs.

    {{site.data.alerts.callout_danger}}
    If you choose **Migrate existing data and replicate ongoing changes** or **Replicate data changes only**, you must first [disable revision history for backups](#before-you-begin).
    {{site.data.alerts.end}}
    <img src="{{ 'images/v23.2/aws-dms-task-configuration.png' | relative_url }}" alt="AWS-DMS-Task-Configuration" style="max-width:100%" />

### Step 2.2. Task settings

1. For the **Editing mode** radio button, keep **Wizard** selected.
1. To preserve the schema you manually created, select **Truncate** or **Do nothing** for the **Target table preparation mode**.
    <img src="{{ 'images/v23.2/aws-dms-task-settings.png' | relative_url }}" alt="AWS-DMS-Task-Settings" style="max-width:100%" />
1. Optionally check **Enable validation** to compare the data in the source and target rows, and verify that the migration succeeded. You can view the results in the [**Table statistics**](#step-3-verify-the-migration) for your migration task. For more information about data validation, see the [AWS documentation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Validating.html).
1. Check the **Enable CloudWatch logs** option. We highly recommend this for troubleshooting potential migration issues. 
1. For the **Target Load**, select **Detailed debug**.
    <img src="{{ 'images/v23.2/aws-dms-cloudwatch-logs.png' | relative_url }}" alt="AWS-DMS-CloudWatch-Logs" style="max-width:100%" />

### Step 2.3. Table mappings

{{site.data.alerts.callout_info}}
When specifying a range of tables to migrate, the following aspects of the source and target database schema **must** match unless you use [transformation rules](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.CustomizingTasks.TableMapping.SelectionTransformation.Transformations.html):

- Column names must be identical.
- Column types must be compatible.
- Column nullability must be identical.
{{site.data.alerts.end}}

1. For the **Editing mode** radio button, keep **Wizard** selected.
1. Select **Add new selection rule**.
1. In the **Schema** dropdown, select **Enter a schema**.
1. Supply the appropriate **Source name** (schema name), **Table name**, and **Action**.
    <img src="{{ 'images/v23.2/aws-dms-table-mappings.png' | relative_url }}" alt="AWS-DMS-Table-Mappings" style="max-width:100%" />

{{site.data.alerts.callout_info}}
Use `%` as an example of a wildcard for all schemas in a PostgreSQL database. However, in MySQL, using `%` as a schema name imports all the databases, including the metadata/system ones, as MySQL treats schemas and databases as the same.
{{site.data.alerts.end}}

## Step 3. Verify the migration

Data should now be moving from source to target. You can analyze the **Table Statistics** page for information about replication.

1. In **AWS DMS**, open **Database migration tasks** in the sidebar.
1. Select the task you created in Step 2.
1. Select **Table statistics** below the **Summary** section.

If your migration succeeded, you should now [re-enable revision history](#before-you-begin) for cluster backups.

If your migration failed for some reason, you can check the checkbox next to the table(s) you wish to re-migrate and select **Reload table data**.

<img src="{{ 'images/v23.2/aws-dms-reload-table-data.png' | relative_url }}" alt="AWS-DMS-Reload-Table-Data" style="max-width:100%" />

## Optional configurations

### AWS PrivateLink

If using CockroachDB {{ site.data.products.dedicated }}, you can enable [AWS PrivateLink](https://aws.amazon.com/privatelink/) to securely connect your AWS application with your CockroachDB {{ site.data.products.dedicated }} cluster using a private endpoint. To configure AWS PrivateLink with CockroachDB {{ site.data.products.dedicated }}, see [Network Authorization](https://www.cockroachlabs.com/docs/cockroachcloud/network-authorization#aws-privatelink).

### `BatchApplyEnabled`

The `BatchApplyEnabled` setting can improve replication performance and is recommended for larger workloads.

1. Open the existing database migration task.
1. Choose your task, and then choose **Modify**.
1. From the **Task settings** section, switch the **Editing mode** from **Wizard** to **JSON editor**. Locate the `BatchApplyEnabled` setting and change its value to `true`. Information about the `BatchApplyEnabled` setting can be found [here](https://aws.amazon.com/premiumsupport/knowledge-center/dms-batch-apply-cdc-replication/).

<img src="{{ 'images/v23.2/aws-dms-batchapplyenabled.png' | relative_url }}" alt="AWS-DMS-BatchApplyEnabled" style="max-width:100%" />

{{site.data.alerts.callout_info}}
`BatchApplyEnabled` does not work when using **Drop tables on target** as a target table preparation mode. Thus, all schema-related changes must be manually copied over if using `BatchApplyEnabled`.
{{site.data.alerts.end}}

## Known limitations

- When using **Truncate** or **Do nothing** as a target table preparation mode, you cannot include tables with any hidden columns. You can verify which tables contain hidden columns by executing the following SQL query:

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > SELECT table_catalog, table_schema, table_name, column_name FROM information_schema.columns WHERE is_hidden = 'YES';
    ~~~

- If you are migrating from PostgreSQL, are using a [`STRING`]({% link {{ page.version.version }}/string.md %}) as a [`PRIMARY KEY`]({% link {{ page.version.version }}/primary-key.md %}), and have selected **Enable validation** in your [task settings](#step-2-2-task-settings), validation can fail due to a difference in how CockroachDB handles case sensitivity in strings. 

    To prevent this error, use `COLLATE "C"` on the relevant columns in PostgreSQL or a [collation]({% link {{ page.version.version }}/collate.md %}) such as `COLLATE "en_US"` in CockroachDB.

- A migration to a [multi-region cluster](multiregion-overview.html) using AWS DMS will fail if the target database has [regional by row tables](table-localities.html#regional-by-row-tables). This is because the `COPY` statement used by DMS is unable to process the `crdb_region` column in regional by row tables.

    To prevent this error, [set the regional by row table localities to `REGIONAL BY TABLE`](alter-table.html#set-the-table-locality-to-regional-by-row) and perform the migration. After the DMS operation is complete, [set the table localities to `REGIONAL BY ROW`](alter-table.html#set-the-table-locality-to-regional-by-row).

## Troubleshooting common issues

- For visibility into migration problems:

    - Check the `SQL_EXEC` [logging channel]({% link {{ page.version.version }}/logging-overview.md %}#logging-channels) for log messages related to `COPY` statements and the tables you are migrating.
    - Check the [Amazon CloudWatch logs that you configured](#step-2-2-task-settings) for messages containing `SQL_ERROR`.

- If you encounter errors like the following:

    ~~~
    2022-10-21T13:24:07 [SOURCE_UNLOAD   ]W:  Value of column 'metadata' in table 'integrations.integration' was truncated to 32768 bytes, actual length: 116664 bytes  (postgres_endpoint_unload.c:1072)
    ~~~

    Try selecting **Full LOB mode** in your [task settings](#step-2-2-task-settings). If this does not resolve the error, select **Limited LOB mode** and gradually increase the **Maximum LOB size** until the error goes away. For more information about LOB (large binary object) modes, see the [AWS documentation](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_Tasks.LOBSupport.html).

- Run the following query from within the target CockroachDB cluster to identify common problems with any tables that were migrated. If problems are found, explanatory messages will be returned in the `cockroach sql` shell.

    {% include_cached copy-clipboard.html %}
    ~~~ sql
    > WITH
        invalid_columns
            AS (
                SELECT
                    'Table '
                    || table_schema
                    || '.'
                    || table_name
                    || ' has column '
                    || column_name
                    || ' which is hidden. Either drop the column or mark it as not hidden for DMS to work.'
                        AS fix_me
                FROM
                    information_schema.columns
                WHERE
                    is_hidden = 'YES'
                    AND table_name NOT LIKE 'awsdms_%'
            ),
        invalid_version
            AS (
                SELECT
                    'This cluster is on a version of CockroachDB which does not support AWS DMS. CockroachDB v21.2.13+ or v22.1+ is required.'
                        AS fix_me
                WHERE
                    split_part(
                        substr(
                            substring(
                                version(),
                                e'v\\d+\\.\\d+.\\d+'
                            ),
                            2
                        ),
                        '.',
                        1
                    )::INT8
                    < 22
                    AND NOT
                            (
                                split_part(
                                    substr(
                                        substring(
                                            version(),
                                            e'v\\d+\\.\\d+.\\d+'
                                        ),
                                        2
                                    ),
                                    '.',
                                    1
                                )::INT8
                                = 21
                                AND split_part(
                                        substr(
                                            substring(
                                                version(),
                                                e'v\\d+\\.\\d+.\\d+'
                                            ),
                                            2
                                        ),
                                        '.',
                                        2
                                    )::INT8
                                    = 2
                                AND split_part(
                                        substr(
                                            substring(
                                                version(),
                                                e'v\\d+\\.\\d+.\\d+'
                                            ),
                                            2
                                        ),
                                        '.',
                                        3
                                    )::INT8
                                    >= 13
                            )
            ),
        has_no_pk
            AS (
                SELECT
                    'Table '
                    || a.table_schema
                    || '.'
                    || a.table_name
                    || ' has column '
                    || a.column_name
                    || ' has no explicit PRIMARY KEY. Ensure you are not using target mode "Drop tables on target" and that this table has a PRIMARY KEY.'
                        AS fix_me
                FROM
                    information_schema.key_column_usage AS a
                    JOIN information_schema.columns AS b ON
                            a.table_schema = b.table_schema
                            AND a.table_name = b.table_name
                            AND a.column_name = b.column_name
                WHERE
                    b.is_hidden = 'YES'
                    AND a.column_name = 'rowid'
                    AND a.table_name NOT LIKE 'awsdms_%'
            )
    SELECT fix_me FROM has_no_pk
    UNION ALL SELECT fix_me FROM invalid_columns
    UNION ALL SELECT fix_me FROM invalid_version;
    ~~~

- Refer to Debugging Your AWS DMS Migrations ([Part 1](https://aws.amazon.com/blogs/database/debugging-your-aws-dms-migrations-what-to-do-when-things-go-wrong-part-1/), [Part 2](https://aws.amazon.com/blogs/database/debugging-your-aws-dms-migrations-what-to-do-when-things-go-wrong-part-2/), and [Part 3](https://aws.amazon.com/blogs/database/debugging-your-aws-dms-migrations-what-to-do-when-things-go-wrong-part-3/)) on the AWS Database Blog.

- If the migration is still failing, [contact Support](https://support.cockroachlabs.com) and include the following information when filing an issue:
    - Source database name.
    - CockroachDB version.
    - Source database schema.
    - CockroachDB database schema.
    - Any relevant logs (e.g., the last 100 lines preceding the AWS DMS failure).
    - Ideally, a sample dataset formatted as a database dump file or CSV.

## See Also

- [Migration Overview]({% link {{ page.version.version }}/migration-overview.md %})
- [Schema Conversion Tool](https://www.cockroachlabs.com/docs/cockroachcloud/migrations-page)
- [`cockroach demo`]({% link {{ page.version.version }}/cockroach-demo.md %})
- [AWS DMS documentation](https://docs.aws.amazon.com/dms/latest/userguide/Welcome.html)
- [Client connection parameters]({% link {{ page.version.version }}/connection-parameters.md %})
- [Third-Party Database Tools]({% link {{ page.version.version }}/third-party-database-tools.md %})
- [Learn CockroachDB SQL]({% link {{ page.version.version }}/learn-cockroachdb-sql.md %})