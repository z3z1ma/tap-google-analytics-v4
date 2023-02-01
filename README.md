# `tap-google-analytics-v4` Meltano Plugin [Hotglue Variant]

`tap-google-analytics-v4` tap is a Singer tap for extracting data from the [Google Analytics Data API (GA4)](https://developers.google.com/analytics/devguides/reporting/data/v1).
It produces JSON-formatted data following the Singer spec. This tap is produced by [Hotglue](https://gitlab.com/hotglue), with the addition of Google Sevice Account autorization provided by [@connorflyn](https://github.com/connorflyn).

Built with the [Meltano SDK](https://sdk.meltano.com) for Singer Taps and Targets.

## Capabilities

* `catalog`
* `state`
* `discover`
* `about`
* `stream-maps`

## Settings

| Setting          | Required | Default | Description |
|:-----------------|:--------:|:-------:|:------------|
| start_date       | True     | None    | The earliest record date to sync |
| property_id      | True     | None    | Google Analytics Property ID |
| key_file_location| False    | None    | File Path to Google Analytics Client Secrets (Service Account) |
| oauth_credentials| False    | None    | Google Analytics OAuth Credentials |
| reports          | False    | None    | Google Analytics Reports Definition |
| end_date         | False    | None    | The last record date to sync |

A full list of supported settings and capabilities is available by running: `tap-google-analytics-v4 --about`

## Installation


## Source Authentication and Authorization

### Authorization Methods

`tap-google-analytics` supports two different ways of authorization:
 - Service account based authorization, where an administrator manually creates a service account with the appropriate permissions to view the account, property, and view you wish to fetch data from
 - OAuth `access_token` based authorization, where this tap gets called with a valid `access_token` and `refresh_token` produced by an OAuth flow conducted in a different system.

If you're setting up `tap-google-analytics` for your own organization and only plan to extract from a handful of different views in the same limited set of properties, Service Account based authorization is the simplest. When you create a service account Google gives you a json file with that service account's credentials called the `client_secrets.json`, and that's all you need to pass to this tap, and you only have to do it once, so this is the recommended way of configuring `tap-google-analytics`.

If you're building something where a wide variety of users need to be able to give access to their Google Analytics, `tap-google-analytics` can use an `access_token` granted by those users to authorize it's requests to Google. This `access_token` is produced by a normal Google OAuth flow, but this flow is outside the scope of `tap-google-analytics`. This is useful if you're integrating `tap-google-analytics` with another system, like Stitch Data might do to allow users to configure their extracts themselves without manual config setup. This tap expects an `access_token`, `refresh_token`, `client_id` and `client_secret` to be passed to it in order to authenticate as the user who granted the token and then access their data.

## Required Analytics Reporting APIs & OAuth Scopes

In order for `tap-google-analytics` to access your Google Analytics Account, it needs the Analytics Reporting API *and* the Analytics API (which are two different things) enabled. If using a service account to authorize, these need to be enabled for a project inside the same organization as your Google Analytics account (see below), or if using an OAuth credential set, they need to be enabled for the project the OAuth client ID and secret come from.

If using the OAuth authorization method, the OAuth flow conducted elsewhere must request at minimum the `analytics.readonly` OAuth scope to get an `access_token` authorized to hit these APIs

### Creating service account credentials

If you have already have a valid `client_secrets.json` for a service account, or if you are using OAuth based authorization, you can skip the rest of this section.

As a first step, you need to create or use an existing project in the Google Developers Console:

1. Sign in to the Google Account you are using for managing Google Analytics (you must have Manage Users permission at the account, property, or view level).

2. Open the [Service accounts page](https://console.developers.google.com/iam-admin/serviceaccounts). If prompted, select a project or create a new one to use for accessing Google Analytics.

3. Click Create service account.

   In the Create service account window, type a name for the service account, and select Furnish a new private key. Then click Save and store it locally as `client_secrets.json`.

   If you already have a service account, you can generate a key by selecting 'Edit' for the account and then selecting the option to generate a key.

Your new public/private key pair is generated and downloaded to your machine; it serves as the only copy of this key. You are responsible for storing it securely.

### Add service account to the Google Analytics account

The newly created service account will have an email address that looks similar to:

```
quickstart@PROJECT-ID.iam.gserviceaccount.com
```

Use this email address to [add a user](https://support.google.com/analytics/answer/1009702) to the Google analytics view you want to access via the API. For using `tap-google-analytics` only [Read & Analyze permissions](https://support.google.com/analytics/answer/2884495) are needed.

### Enable the APIs

1. Visit the [Google Analytics Reporting API](https://console.developers.google.com/apis/api/analyticsreporting.googleapis.com/overview) dashboard and make sure that the project you used in the `Create credentials` step is selected.

From this dashboard, you can enable/disable the API for your account, set Quotas and check usage stats for the service account you are using with `tap-google-analytics`.

2. Visit the [Google Analytics API](https://console.developers.google.com/apis/api/analytics.googleapis.com/overview) dashboard, make sure that the project you used in the `Create credentials` step is selected and enable the API for your account.


## Configuration Settings

A sample config for `tap-google-analytics` might look like this:

One of the credentials keys must exist:

- key_file_location
- oauth_credentials


**sample_config.json**
```js
{
  "property_id": "123456789",
  "reports": "reports.json",
  "start_date": "2019-05-01T00:00:00Z",
  "end_date": "2019-06-01T00:00:00Z",
  "key_file_location": "client_secrets.json",
  or
  "client_secrets": {
    "foo": "bar"
  },
  or
  "oauth_credentials": {
    "foo": "bar"
  }
}
```

### Reports

You can easily run `tap-google-analytics` by itself or in a pipeline using [Meltano](https://meltano.com/).

As the Google Analytics Reports are defined dynamically and there are practically infinite combinations of dimensions and metrics a user can ask for, the entities and their schema (i.e. the Catalog for this tap) are not static. So, this tap behaves more or less similarly to a tap extracting data from a Data Source (e.g. a Postgres Database).

The difference of `tap-google-analytics` to a database tap is that the Catalog (available entities/streams and their schema) is dynamic but not available to be discovered at run time by connecting to the Data Source. It must be dynamically generated based on the reports the user wants to generate by connecting to the Google Analytics Reporting API.

To that end, this tap uses an additional JSON file for the definition of the reports that the user wants to be generated. You can check, as an example, the JSON file used as a default in [tap_google_analytics/defaults/default_report_definition.json](https://github.com/MeltanoLabs/tap-google-analytics/blob/main/tap_google_analytics/defaults/default_report_definition.json). Those report definitions could be part of the `config.json`, but we prefer to keep `config.json` small and clean and provide the definitions by using an additional file.

Based on the report(s) definition, it generates a valid Catalog that follows the [Singer spec](https://github.com/singer-io/getting-started/blob/master/SPEC.md).

It then behaves as any Singer compatible tap and uses that Catalog (or any Catalog generated by a `tap-google-analytics`) to generate the requested reports. The additional JSON file for defining the reports is only required for generating an initial Catalog.

When no report definitions are provided by the user, `tap-google-analytics` generates a default Catalog with some common reports provided:
- **website_overview**: Most common metrics (users, new users, sessions, avg session duration, page views, bounce rate, etc) per day
- **traffic_sources**: Most common metrics per day, source, medium and social network
- **pages**: Most common page metrics (page views, unique page views, avg time on page, etc) per day, host name and page path
- **locations**: Most common metrics per day and various location dimensions (continent, country, region, city, etc)
- **monthly_active_users**: Monthly active users (past 30 days) per day
- **weekly_active_users**: Weekly active users (past 7 days) per day
- **daily_active_users**: Daily active users (past 1 day) per day
- **devices**: Most common metrics per day, device category, operating system and browser

This tap only allows incremental syncs using STATE records if the `ga:date` dimension is included in the report. Otherwise it does not save state and does a full load each time. Without the `ga:date` dimension this can be mitigated by allowing for chunked runs using [start_date, end_date).

If not provided and the tap runs without a `--catalog` also provided, use [tap_google_analytics/defaults/default_report_definition.json](tap_google_analytics/defaults/default_report_definition.json) as the default definition.

The `reports.json` file structure expected by the `reports` config key is really simple:

**reports.json**
```
[
  { "name" : "name of stream to be used",
    "dimensions" :
    [
      "Google Analytics Dimension",
      "Another Google Analytics Dimension",
      ... up to 7 dimensions per stream ...
    ],
    "metrics" :
    [
      "Google Analytics Metric",
      "Another Google Analytics Metric",
      ... up to 10 metrics per stream ...
    ]
  },
  {

  	... another stream definition ...
  },
  ... as many streams / reports as the user wants ...
]
```

For example, if you want to extract user stats per day in a users_per_day stream and session stats per day and country in a sessions_per_country_day stream:

```
[
  { "name" : "users_per_day",
    "dimensions" :
    [
      "ga:date"
    ],
    "metrics" :
    [
      "ga:users",
      "ga:newUsers"
    ]
  },
  { "name" : "sessions_per_country_day",
    "dimensions" :
    [
      "ga:date",
      "ga:country"
    ],
    "metrics" :
    [
      "ga:sessions",
      "ga:sessionsPerUser",
      "ga:avgSessionDuration"
    ]
  }
]
```

You can check [tap-google-analytics/defaults/default_report_definition.json](tap-google-analytics/defaults/default_report_definition.json) for a more lengthy, detailed example.

##### Segments

If you want to use the `ga:segment` dimension, you must specify the segment IDs in your reports.json stream / report config:

```
[
  {
    "name": "acquisition",
    "dimensions": [
      "ga:date",
      "ga:segment",
      "ga:channelGrouping"
    ],
    "metrics": [
      "ga:users",
      "ga:newUsers",
      "ga:sessions"
    ],
    "segments": [
      "gaid::-1",
      "gaid::U7LSsrWRTq6JIIS8G8brrQ"
    ]
  }
]
```

Segment IDs can be found with the [GA Query explorer](https://ga-dev-tools.appspot.com/query-explorer). The account configured for authentication must either own the segment, or have "Collaborate" access to the GA view as well as the segment itself having its Segment Availability set to "Collaborators and I can apply/edit Segment in this View".


## Usage

You can easily run `tap-google-analytics` by itself or in a pipeline using [Meltano](https://meltano.com/).

### Executing the Tap Directly

```bash
tap-google-analytics-v4 --version
tap-google-analytics-v4 --help
tap-google-analytics-v4 --config CONFIG --discover > ./catalog.json
```

## Developer Resources

### Implementation details

This tap makes some explicit decisions:
- All Google Analytics Metrics and Dimensions used are preserved with their name changed to `ga_XXX` from `ga:XXX`

  e.g. `ga:date` --> `ga_date`

- All dimensions are part of the stream's key definition (`table-key-properties`).

- The {start_date, end_date} parameters for the report query are also added to the schema as {report_start_date, report_end_date}.

  This is important for defining the date range the records are for, especially when 'ga:date' is not part of the requested Dimensions.

- If 'ga:date' has not been added as one of the Dimensions, then the {report_start_date, report_end_date} attributes are also added as keys.

  For example, if a user requests to see user stats by device or by source, the {start_date, end_date} can be used as part of the key uniquelly identifying the generated stats.

  That way we can properly identify rows with no date dimension included and also update those rows over overlapping runs of the tap.

- All streams and attributes are set to `"inclusion": "automatic"`

- There is a custom metadata keyword defined for all schema attributes:

  `ga_type: dimension | metric`

  This keyword is required for processing the catalog afterwards and generating the query to be send to GA API, as dimensions and metrics are not treated equally (they are sent as separate lists of attributes)

- The tap dynamically fetches the available dimensions and metrics and their data type. We have to use the Google Analytics API V3 for fetching those lists, as they are not provided in the Analytics Reporting API V4.

  We use those lists to check for invalid dimension or metric names requested by the user and inform her about those errors before the tap runs and starts making requests to the Google Analytics API.

  We also use the dynamically fetched dimensions and metrics lists to set the data type for those attributes and cast the values accordingly (in case of integer or numeric values)



Tap shortcomings (contributions are more than welcome):

- This tap only allows incremental syncs using STATE records if the `ga:date` dimension is included in the report. Incremental syncs should be supported using other combinations of Time Dimensions if provided.

- We may be checking for valid Google Analytics Metrics and Dimensions and report back to the user with errors, but we are not checking for valid combinations. Not all dimensions and metrics can be queried together and we should add a black list of metrics and dimensions that can not be used with specific dimensions (e.g. ga:impressions with ga:browser) or with other metrics (e.g. ga:XXdayUsers with other ga:XXdayUsers or ga:users).

  The [Available Metrics/Dimensions and combos page](https://developers.google.com/analytics/devguides/reporting/core/dimsmets) provides a way to discover those restrictions.

### Initialize your Development Environment

```bash
pipx install poetry
poetry install
```

### Create and Run Tests

Create tests within the `tap_google_analytics/tests` subfolder and
  then run:

```bash
poetry run tox
poetry run tox -e pytest
poetry run tox -e format
poetry run tox -e lint
```

The tests require an environment variable called `CLIENT_SECRETS` which is either the escaped client secret json file content as a string (e.g. "{\"type\": \"service_account\",\"project_id\":...}") or the base64 encoded version of that same escaped string.
Base64 is an option primarily for CI to pass secrets in the recommended fashion.

You can also test the `tap-google-analytics` CLI interface directly using `poetry run`:

```bash
poetry run tap-google-analytics-v4 --help
```

### Testing with [Meltano](https://www.meltano.com)

_**Note:** This tap will work in any Singer environment and does not require Meltano.
Examples here are for convenience and to streamline end-to-end orchestration scenarios._

Your project comes with a custom `meltano.yml` project file already created. Open the `meltano.yml` and follow any _"TODO"_ items listed in
the file.

Next, install Meltano (if you haven't already) and any needed plugins:

```bash
# Install meltano
pipx install meltano
# Initialize meltano within this directory
cd tap-google-analytics-v4
meltano install
```

Now you can test and orchestrate using Meltano:

```bash
# Test invocation:
meltano invoke tap-google-analytics-v4 --version
# OR run a test `elt` pipeline:
meltano elt tap-google-analytics-v4 target-jsonl
```

### SDK Dev Guide

See the [dev guide](https://sdk.meltano.com/en/latest/dev_guide.html) for more instructions on how to use the SDK to 
develop your own taps and targets.
