# Campaign Report
This module generates campaign reports for old versions of Xploree (prior to V2.0) for use of the adorbit team.

### Report Format
The reports are generated in CSV format with the following columns:
|Field Name           |
|---------------------|
|Date                 |
|Version              |
|Campaign ID          |
|Category Name        |
|Subcategory Name     |
|Advertiser Name      |
|Advertiser Brand     |
|Discovery Impressions|
|Discovery Clicks     |
|Search Impressions   |
|Search Clicks        |
|Bar Impressions      |
|Key Clicks           |

### GA Data Sources
The aggregates are collected from Google Analytics using the [GA Reporting API v4][gapi].
The following views are used:
 * `107846091` - *1.2 - 1.2.1*
 * `110269395` - *1.2.2*
 * `111131925` - *1.3*
 * `112078635` - *1.3.1*
 * `112721733` - *1.4*
 * `113911700` - *1.4.1 - 1.7.3*

#### Aggregate Fields
The aggregates are calculated using the following conditions:

**1. Discovery Impressions** *is calculated as count of all events where:*
|Event Category is       |
|------------------------|
|`PrestoMore_Impressions`|
***OR***
|Event Action is one of |
|-----------------------|
|`Swiped`               |
|`swiped`               |
|`Source_Swipe`         |
|`PrestoMore_Impression`|
***OR*** *combination of following categories and actions*
|Event Category         |Event Action    |
|-----------------------|----------------|
|`PrestoTile_Impression`|`PrestoKeyClick`|
|`PrestoTile_Impression`|`prestoKeyClick`|

**2. Discovery Clicks** *is calculated as count of all events where:*
|Event Action is one of  |
|------------------------|
|`CTA_Type_MoreInfo`     |
|`CTA_Type_Webclick`     |
|`WebsiteClick`          |
|`CTA_Type_Appclick`     |
|`CTA_Type_Callclick`    |
|`CTA_Type_Couponclick`  |
|`GetCodeClick`          |
|`ImageClick`            |
|`PrestoTile_Image_click`|
***OR***
|Event Label is one of   |
|------------------------|
|`Source_PrestoTile`     |
|`Source_Favourite`      |
|`Source_Share`          |
|`Source_PrestoBar`      |
|`CallClick`             |
|`Source_Notification`   |

**3. Search Impressions**  *is calculated as count of all events where:*
|Event Category is   |
|--------------------|
|`Search_impressions`|

**4. Search Clicks**  *is calculated as count of all events where:*
|Event Category is|
|-----------------|
|`Search_Clicks`  |

**5. Bar Impressions**  *is calculated as count of all events where:*
|Event Category is       |
|------------------------|
|`PrestoMore_Impressions`|
***OR***
|Event Action is        |
|-----------------------|
|`PrestoMore_Impression`|

**6. Key Clicks** *is calculated as count of all events with the combination of:*
|Event Category         |Event Action    |
|-----------------------|----------------|
|`PrestoTile_Impression`|`PrestoKeyClick`|
|`PrestoTile_Impression`|`prestoKeyClick`|

### Building the Repository:
*Step 1. Clone the services_misc repo from git*
```sh
$ git clone git:/kpt/services_misc.git
```
*Step 2. Go to `reports/campaign_reports` folder. Create a virtual python3 environment and activate it*
```sh
$ virtualenv -p python3 local_env
$ source local_env/bin/activate
```

*Step 3. Install the required modules*
```sh
$ pip install -r requirements.txt
```

The campaign reports repo should be ready to use now. For correct execution, it will additionally require a couple of config/cred files. Read the next sections for details.

### Daily Report
The python script `gnerate_report.py` is responsible for generating the daily report. This script fetches the aggregates for the given date *(default report date is day-before-yesterday, i.e., 2 days prior to run-date)* from GA and populates the remaining fields by looking up the *adorbit_campaign* db to generate a csv report. This report then gets uploaded to [AWS S3][daily].

**Usage**

```
$ ./generate_report.py --help
usage: generate_report.py [-h] [--ga-cred-file] [--db-cred-file]
                          [--report-date]

optional arguments:
  -h, --help       show this help message and exit
  --ga-cred-file   default: ./XplGA-1827a5600a39.json
  --db-cred-file   default: ./db_config.py
  --report-date    Report date as YYYY-MM-DD
```

Read the section on *Config files*  for information on the cred files.

### Weekly Report
The python script `weekly_report.py` takes care of generating a consolidated report by combining multiple daily reports. This script downloads the daily reports from [AWS S3][daily] for all days between the given start and end dates. *By default, it has the start and end dates of last week (ending 2 days prior to run-date)*. Once downloaded, the daily reports are combined together into one weekly csv report, which in turn gets uploaded to the weekly bucket of [AWS S3][weekly]. Finally, a copy of this conbined report is e-mailed to the adorbit team members.

**Usage**
```
$ ./weekly_report.py --help
usage: weekly_report.py [-h] [-s] [-e]

optional arguments:
  -h, --help          show this help message and exit
  -s , --start-date   Report start date as YYYY-MM-DD
  -e , --end-date     Report end date as YYYY-MM-DD
```

### DB Dump
The category and advertiser related fields in the report are populated from the Aurora MySQL database `adorbit_campaign`. This database is in turn populated from the MySQL adorbit production database by taking nightly dumps of the relevant tables `adorbit_campaign_master` and `adorbit_dfp_category_subcategory`. This db dump operation is done nightly to keep it up-to-date.
The bash script `db_dumps/campaign_tables/db_dump` in the `services_misc` repo takes care of this task. It expects a config file for the source and destination db credentials explained in the next section.

**Usage**
```
$ ./db_dump --help
usage: DB_CONFIG=<db_config_path> SOURCE_DB=<db name>  ./db_dump [--help]

The following variables need to be supplied in the same statement or
exported from a parent bash command:
  DB_CONFIG     path to the db config file;
  SOURCE_DB     db name of the source database

optional arguments:
  --help        show this help message and exit
```

### Config files
The `generate_report` script requires two credentials files:
1. `--ga-cred-file` - The GA credentials file is required to make OAuth 2.0 authorized calls to Google APIs for querying the GA data. By default, the script looks for `XplGA-1827a5600a39.json` file in the current directory.
2. `--db-cred-file` - The DB credentials file is required to connect to the AWS Aurora MySQL `adorbit_campaign` database. The config file contents should look like:
```sh
db = {
    'user'     : '<db user name>',
    'password' : '********',
    'host'     : '<db host>',
    'database' : '<db name>'
}
```
The `db_dump` bash script requires a db config file comprising db credentials for the source and destination databases. The contents of this db config file should look like:
```sh
[mysqldump]
# config for creating db dump from adorbit production database
host=<hostname of source database>
user=<db username>
password=********
verbose # command line option for printing the output of mysqldump

[mysql]
# config for importing campaign tables from adorbit db dump
host=<hostname of destination database>
user=<db username>
password=********
```
***NOTE*** - *The source database name for `mysqldump` is provided as command line variable and is NOT taken from this config.*jjjjjj

### Cron
Three cron jobs are required for the campaign reports:
1. Daily report - scheduled to run *everyday at 9 AM*,
2. Weekly report - scheduled to execute *every Monday at 10 AM*, and
3. DB Dump - scheduled to run *every night at 1 AM*.

The bash script `reports/campaign_reports/bin/add_to_cron` takes care of scheduling the first two jobs. The script requires the paths to Google and DB credential files in variables `GA_CRED` and `DB_CRED` respectively.

The bash script `db_dumps/campaign_tables/add_to_cron` takes care of scheduling the third cron job (db dump). The script expects the path to db config file and source db name as explained above.

### Additional Remarks
There are two additional files `bq_service.py` and `bq_query.py` in the `reports/campaign_reports` repo, which are not being used currently. These files contain functions to pull the aggregates data from *Google Big Query* as an alternative to *Google Analytics APIs*. These functions can be used if there is a need to switch to BQ in future or to compare the data collected from GA.

### TODO
* ***Alert Logs in Email*** - At present, a basic alert email is sent from the weekly report if any of the required daily reports are missing, asking to log on to the server and check the daily report logs for debugging. It would be good to send alert emails (with logs attached) whenever a daily report job has an error.


[//]: <Hyperlinks>
[gapi]: <https://developers.google.com/analytics/devguides/reporting/core/v4/>
[daily]:<https://s3.console.aws.amazon.com/s3/buckets/xploree.reports/campaign_reports_v1/daily/?region=ap-southeast-1>
[weekly]:<https://s3.console.aws.amazon.com/s3/buckets/xploree.reports/campaign_reports_v1/weekly/?region=ap-southeast-1>
