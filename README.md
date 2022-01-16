## Introduction

The Atadataco Warehouse dbt framework is a set of data models, data transformations and data warehousing design patterns for use with dbt ("Data Build Tool"), an open-source data transformation and orchestration toolkit we use as the core set of models and transformations on all of our client projects.

* Contains pre-built, standardised data source models for popular sources (Facebook Ads, Google Ads, Stripe, Segment, etc)
* Currently configured for Segment data pipeline services
* Optimized for Snowflake data warehouse targets
* Combines and integrates data from multiple sources, deduplicates and creates single contact and company records
* Creates subject-area dimensional warehouses e.g. Finance, Marketing, Product, CRM
* Provides utilities for data profiling, ETL run logging and analysis
* Is configured through a set of variables in the dbt_project.yml file
* [Getting Started with dbt](https://rittmananalytics.com/getting-started-with-dbt) consulting packages
* [dbt Viewpoint](https://docs.getdbt.com/docs/about/viewpoint/)
* [dbtCloud](https://docs.getdbt.com/docs/dbt-cloud/cloud-overview) for scheduling and orchestrating dbt and the RA Data Warehouse


## What Data Warehouse, Data Pipeline and Data Collection Technologies are Supported?

* Snowflake Data Warehouse
* Segment

## What SaaS Sources are Supported?

* Stripe Payments (Segment)
* Stripe Subscriptions (Segment)
* Segment Events and Pageviews (Segment)
* Google Ads (Segment)
* Facebook Ads (Segment)
* Custom data sources

## Design Approach

### Separation of Data Sources, Integration and Warehouse Module Layers

There are three distinct layers in the data warehouse:

1. A layer of source and ETL pipeline-specific data sources, containing SQL code used to transform and rename incoming tables from each source into common formats

2. An Integration layer, containing SQL transformations used to integrate, merge, deduplicate and transform data ready for loading into the main warehouse fact and dimension tables.

3. A warehouse layer made-up of subject area data marts, each of which contains multiple fact and conformed dimension tables

### dbt Package Structure

dbt models inside this project are grouped together by these layers, with each data source "adapter" having all of its source SQL transformations contained with it.


### Dimension Union and Merge Deduplication Design Pattern

Customers, contacts, projects and other shared dimensions are automatically created from all data sources, deduplicating by name and merge lookup files using a process that preserves source system keys whilst assigning a unique ID for each customer, contact etc.

1. Each set of dbt source module provides a unique ID, prefixed with the source name, and another field value (for example, user name) that can be used for deduplicating dimension members downstream.

```
WITH source AS (
    {{ filter_stitch_relation(relation=var('stg_hubspot_crm_stitch_companies_table'),unique_column='companyid') }}
  ),
  renamed AS (
    SELECT
      CONCAT('{{ var('stg_hubspot_crm_id-prefix') }}',companyid) AS company_id,
      REPLACE(REPLACE(REPLACE(properties.name.value, 'Limited', ''), 'ltd', ''),', Inc.','') AS company_name,
      properties.address.value AS                   company_address,
      properties.address2.value AS                  company_address2,
      properties.city.value AS                      company_city,
      properties.state.value AS                     company_state,
      properties.country.value AS                   company_country,
      properties.zip.value AS                       company_zip,
      properties.phone.value AS                     company_phone,
      properties.website.value AS                   company_website,
      properties.industry.value AS                  company_industry,
      properties.linkedin_company_page.value AS     company_linkedin_company_page,
      properties.linkedinbio.value AS               company_linkedin_bio,
      properties.twitterhandle.value AS             company_twitterhandle,
      properties.description.value AS               company_description,
      cast (null as {{ dbt_utils.type_string() }}) AS                      company_finance_status,
      cast (null as {{ dbt_utils.type_string() }})      as                 company_currency_code,
      properties.createdate.value AS                company_created_date,
      properties.hs_lastmodifieddate.value          company_last_modified_date
    FROM
      source
  )
SELECT
  *
FROM
  renamed
```

2. These tables are then initially unioned (UNION ALL) together in the i_* integration view, with the set of sources to be merged determined by the relevant variable in the dbt_project.yml config file:

```

crm_warehouse_company_sources: ['hubspot_crm','harvest_projects','xero_accounting','stripe_payments','asana_projects','jira_projects'

```

Unioning takes place using a Jinja "for" loop

```
with t_companies_pre_merged as (

    {% for source in var('crm_warehouse_company_sources') %}
      {% set relation_source = 'stg_' + source + '_companies' %}

      select
        '{{source}}' as source,
        *
        from {{ ref(relation_source) }}

        {% if not loop.last %}union all{% endif %}
      {% endfor %}

    )
```

3. An CTE containing an array of source dimension IDs is then created within the int_ integration view, grouped by the deduplication column (in this example, user name)

Any other multivalue columns are similarly-grouped by the deduplication column in further CTEs within the i_ integration view, for example list of email addresses for a user. If the warehouse is Snowflake, this is detected through the target.type Jinja function and the functionally-equivalent Snowflake SQL version is used instead.

```
{% elif target.type == 'snowflake' %}

      all_company_ids as (
          SELECT company_name,
                 array_agg(
                    distinct company_id
                  ) as all_company_ids
            FROM t_companies_pre_merged
          group by 1),
      all_company_addresses as (
          SELECT company_name,
                 array_agg(
                      parse_json (
                        concat('{"company_address":"',company_address,
                               '", "company_address2":"',company_address2,
                               '", "company_city":"',company_city,
                               '", "company_state":"',company_state,
                               '", "company_country":"',company_country,
                               '", "company_zip":"',company_zip,'"} ')
                      )
                 ) as all_company_addresses

```

For dimensions where merging of members by name is not sufficient (for example, company names that cannot be relied on to always be spelt the same across all sources) we can add seed files to map one member to another and then extend the logic of the merge to make use of this merge file, if the target is Snowflake the following SQL is executed instead:

```
{% elif target.type == 'snowflake' %}

             left outer join (
                      select company_name, array_agg(all_company_ids) as all_company_ids
                           from (
                             select
                               c2.company_name as company_name,
                               c2.all_company_ids as all_company_ids
                             from   {{ ref('companies_merge_list') }} m
                             join (
                               SELECT c1.company_name, c1f.value::string as all_company_ids from {{ ref('int_companies_pre_merged') }} c1,table(flatten(c1.all_company_ids)) c1f) c1
                             on m.old_company_id = c1.all_company_ids
                             join (
                               SELECT c2.company_name, c2f.value::string as all_company_ids from {{ ref('int_companies_pre_merged') }} c2,table(flatten(c2.all_company_ids)) c2f) c2
                             on m.company_id = c2.all_company_ids
                             union all
                             select
                               c2.company_name as company_name,
                               c1.all_company_ids as all_company_ids
                             from   {{ ref('companies_merge_list') }} m
                             join (
                               SELECT c1.company_name, c1f.value::string as all_company_ids from {{ ref('int_companies_pre_merged') }} c1,table(flatten(c1.all_company_ids)) c1f) c1
                               on m.old_company_id = c1.all_company_ids
                               join (
                                 SELECT c2.company_name, c2f.value::string as all_company_ids from {{ ref('int_companies_pre_merged') }} c2,table(flatten(c2.all_company_ids)) c2f) c2
                               on m.company_id = c2.all_company_ids
                             )
                       group by 1
                  ) m
             on c.company_name = m.company_name
             where c.company_name not in (
                 select
                 c2.company_name
                 from   {{ ref('companies_merge_list') }} m
                 join (SELECT c2.company_name, c2f.value::string as all_company_ids
                       from {{ ref('int_companies_pre_merged') }} c2,table(flatten(c2.all_company_ids)) c2f) c2
                       on m.old_company_id = c2.all_company_ids)
```

4. Within the i_ integration view, all remaining columns are then deduplicated by the deduplication column.

```
SELECT user_name,
		MAX(contact_is_contractor) as contact_is_contractor,
		MAX(contact_is_staff) as contact_is_staff,
		MAX(contact_weekly_capacity) as contact_weekly_capacity ,
		MAX(user_phone) as user_phone,
		MAX(contact_default_hourly_rate) as contact_default_hourly_rate,
		MAX(contact_cost_rate) as contact_cost_rate,
		MAX(contact_is_active) as contact_is_active,
		MAX(user_created_ts) as user_created_ts,
		MAX(user_last_modified_ts) as user_last_modified_ts,
	FROM t_users_merge_list
	GROUP BY 1
```

5. Then this deduplicated CTE is joined-back to the CTE, along with any other multivalue column CTEs

```
SELECT i.all_user_ids,
        u.*,
        e.all_user_emails
 FROM (
	SELECT user_name,
		MAX(contact_is_contractor) as contact_is_contractor,
		MAX(contact_is_staff) as contact_is_staff,
		MAX(contact_weekly_capacity) as contact_weekly_capacity ,
		MAX(user_phone) as user_phone,
		MAX(contact_default_hourly_rate) as contact_default_hourly_rate,
		MAX(contact_cost_rate) as contact_cost_rate,
		MAX(contact_is_active) as contact_is_active,
		MAX(user_created_ts) as user_created_ts,
		MAX(user_last_modified_ts) as user_last_modified_ts,
	FROM t_users_merge_list
	GROUP BY 1) u
JOIN user_emails e
ON u.user_name = COALESCE(e.user_name,'Unknown')
JOIN user_ids i
ON u.user_name = i.user_name
```

5. The wh_ warehouse dimension table then adds a surrogate key for each dimension member.

```
WITH companies_dim as (
  SELECT
    {{ dbt_utils.surrogate_key(['company_name']) }} as company_pk,
    *
  FROM
    {{ ref('int_companies') }} c
)
select * from companies_dim
```

6. The i_ integration view for the associated fact table contains rows referencing these deduplicated dimension members using the source system IDs e.g. 'harvest-2122', 'asana-22122'

7. When loading the associated wh_ fact table, the lookup to the wh_ dimension table uses UNNEST() to query the array of source system IDs when the target is BigQuery, returning the company_pk as the dimension surrogate key

```
WITH delivery_projects AS
  (
  SELECT *
  FROM   {{ ref('int_delivery_projects') }}
),
{% if target.type == 'bigquery' %}
  companies_dim as (
    SELECT {{ dbt_utils.star(from=ref('wh_companies_dim')) }}
    from {{ ref('wh_companies_dim') }}
  )
SELECT
	...
FROM
   delivery_projects p
   {% if target.type == 'bigquery' %}
     JOIN companies_dim c
       ON p.company_id IN UNNEST(c.all_company_ids)
```

Wheras when Snowflake is the target warehouse, the following SQL is used instead:

```
{% elif target.type == 'snowflake' %}
companies_dim as (
    SELECT c.company_pk, cf.value::string as company_id
    from {{ ref('wh_companies_dim') }} c,table(flatten(c.all_company_ids)) cf
)
SELECT
   ...
FROM
   delivery_projects p
{% elif target.type == 'snowflake' %}
     JOIN companies_dim c
       ON p.company_id = c.company_id
```

8. The wh_ dimension table contains the source system IDs and other multivalue dimension columns as repeating columns for BigQuery warehouse targets and Variant datatypes containing JSON values for Snowflake.

Note that the threshold at which the profiler recommends that a columm should be considered for a unique key or NOT NULL test is configurable at the start of the macro code.

Then, within each data source adapter you will find a model definition such as this one for the Asana Projects source :

```
{% if not var("enable_asana_projects_source") %}
{{
    config(
        enabled=false
    )
}}
{% endif %}
{% if var("etl") == 'fivetran' %}
  {{  profile_schema(var('fivetran_schema')) }}
{% elif var("etl") == 'stitch' %}
  {{  profile_schema(var('stitch_schema')) }}
{% endif %}
```

These models when run will automatically create views within the the "profile" dataset (e.g. `analytics_profile`) that you can use to audit and profile the data from newly-enabled data source adapters (note that you will need to create corresponding model files yourself for any new, custom data source adapters).

There is also a "profile_wh_tables.sql" model within the /models/utils folder that runs the following jinja code:

```
{{ profile_schema(target.schema) }}
```

to automatically profile all of the fact and dimension tables in the warehouse at the end of dbt processing.



## Setup Steps.

Note that these are fairly basic instructions and more documentation will be added in due course, consider this a starting point and be prepared to dig around in the code to work out how it all works.

1. Fork or clone the repo to create a fresh copy for your project.

2. Install dbt and create your profile.yml file with either Snowflake as your target data warehouse. The Warehouse framework will automatically run either Snowflake-dialect SQL code depending on which warehouse target is being used.

3. Edit the dbt_project.yml configuration file to specify which data sources provide data for the various integration modules. 

Start by locating the vars: section in the config file:

```vars:
  crm_warehouse_company_sources: []
  crm_warehouse_contact_sources: []
  crm_warehouse_conversations_sources: []
  marketing_warehouse_ad_campaign_sources: []
```

and specify the data sources for each integration table like this:

```
vars:
  crm_warehouse_company_sources: ['hubspot_crm','harvest_projects','xero_accounting','stripe_payments','asana_projects','jira_projects','looker_usage']
  crm_warehouse_contact_sources: ['hubspot_crm','harvest_projects','xero_accounting','mailchimp_email','asana_projects','jira_projects','looker_usage']
  crm_warehouse_conversations_sources: ['hubspot_crm','intercom_messaging']
  marketing_warehouse_ad_campaign_sources: ['google_ads','facebook_ads','mailchimp_email','hubspot_email']
```

4. Now edit the variable settings for the source modules you have chosen to use, for example for Facebook Ads you can choose from Stitch or Segment as the data pipeline (ETL) technology, specify the database name and schema name.

```
stg_facebook_ads_id-prefix: fbads-
  stg_facebook_ads_etl: segment
  stg_facebook_ads_stitch_schema: stitch_facebook_ads
  stg_facebook_ads_stitch_ad_performance_table: "{{ source('stitch_facebook_ads', 'insights') }}"
```

5. Note also the settings as the end of the dbt_project.yml file:

```
web_sessionization_trailing_window: 3
  web_inactivity_cutoff: 30 * 60
  attribution_create_account_event_type: account_opened
  attribution_conversion_event_type: subscribed
  attribution_topup_event_type: account_credited
  attribution_converter_ltv: 200
  enable_companies_merge_file: true
  enable_ip_geo_enrichment: false
```

TODO: Further documentation on the setup process.


## Contributing

Contributions are welcome. To contribute:

1. fork this repo,
2. make and test changes, and
3. submit a PR. All contributions must be widely relevant to users of each SaaS data source and not contain logic specific to a given business.



## Credit for original open-source repo to:
https://github.com/rittmananalytics/
