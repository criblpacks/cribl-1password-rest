# 1Password Rest Collector
----

## About this Pack

This pack is designed to handle JSON data collected from the 1Password Audit Event API endpoint. The JSON is parsed and the timestamp normalized from the proper field within each event. The pack offers two different optional methods of reduction:

1. Drop, Sample, or Suppress based on action/object type. You can target individual action/object types by modifying the provided 1password_audit_events.csv lookup under Knowledge > Lookups.
2. Remove duplicate fields.

The pack also currently includes three forms of outputs:

1. Normalized JSON
2. OCSF - Primarily meant for Amazon Security Lake, the pack can normalize the data into the proper OCSF categories based on action/object type. The target OCSF category can also be managed on an individual action/object type basis through the lookup file **1password_audit_events.csv** .
3. Splunk - default index and sourcetype supplied from Knowledge > Variables, but can be overwritten in pipeline

## Deployment

The 1Password Rest pack allows for events to be sent from the Audit Events API endpoint and normalized into the proper format for the required destinations. To use this pack, follow these steps:

## 1. Configure the Pack

This pack includes several functions that can help reduce events. Please make sure you evaluate the functions before enabling to ensure vital data is not missed.

Additionally, several output formats are available to be selected. Please only enable one output, as enabling multiple may break the output formatting.

## 2. Configure the Event Breaker Rule

The Event Breaker strips headers from events, leaving only the records of interest and capturing timestamps properly. You will have a single event for each unique record. The event breaker is configured to set the timestamp equal to the 'timestamp' field within each event.

- Navigate to Manage > Processing > Knowledge > Event Breaker Rules.
- Click Add Ruleset.
- Click Manage as JSON at lower left.
- Paste the Event Breaker Ruleset JSON from Appendix A below into the window.
- Click OK.

## 3. Configure the Rest Collector Source

From the top nav of a Cribl Stream instance or Group, select Data > Sources, then select Collectors > REST from the Data Sources page's tiles or the Sources left nav. Click Add Collector to open the REST > New Collector modal.

- Click Configure as JSON to open the configuration editor.
- Paste the Rest Collector JSON from Appendix B below into the window.
- Click Configure Collector in the upper left.
- Enter the value for Collect URL based on info in the above API reference. Example: 'https://events.1password.com/api/v2/auditevents'.
- Enter the value for the Collect header field named Authorization. Example: 'Bearer eYk4hkj3hk3h423424...........'
- Click Save.

## 4. Connect the pack

Connect the pack to the 1Password Rest Collector on the Global Routes page where you can add a new route. 

- specify a filter expression for the new 1Password rest collector source. Example: `__inputId.includes('1Password')`
- choose the cribl-1password-rest pack in the Pipeline dropdown.

## Release Notes

### Version 0.1.0
Initial release

## Contributing to the Pack

To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).

## Appendix A
### Event Breaker JSON
```
{
  "id": "1Password",
  "minRawLength": 256,
  "rules": [
    {
      "condition": "true",
      "type": "json_array",
      "timestampAnchorRegex": "/timestamp:/",
      "timestamp": {
        "type": "auto",
        "length": 100
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 10000000,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "jsonExtractAll": false,
      "eventBreakerRegex": "/[\\n\\r]+(?!\\s)/",
      "name": "Items",
      "jsonArrayField": "items"
    }
  ],
  "description": "The1Password Events API returns multiple records per query under the \"item\" JSON objects."
}
```

## Appendix B
### Collector Source JSON
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "cronSchedule": "*/5 * * * *",
    "maxConcurrentRuns": 1,
    "skippable": true,
    "run": {
      "rescheduleDroppedTasks": true,
      "maxTaskReschedule": 1,
      "logLevel": "info",
      "jobTimeout": "60m",
      "mode": "run",
      "timeRangeType": "relative",
      "timeWarning": {},
      "expression": "true",
      "minTaskSize": "1MB",
      "maxTaskSize": "10MB",
      "stateTracking": {
        "stateUpdateExpression": "__timestampExtracted !== false && {latestTime: (state.latestTime || 0) > _time ? state.latestTime : _time}",
        "stateMergeExpression": "(prevState.latestTime || 0) > newState.latestTime ? prevState : newState",
        "enabled": true
      },
      "earliest": "-2d",
      "latest": "now"
    },
    "enabled": false
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "none"
      },
      "collectMethod": "post_with_body",
      "pagination": {
        "type": "response_body",
        "maxPages": 50,
        "attribute": [
          "cursor",
          "has_more"
        ],
        "lastPageExpr": "has_more === false"
      },
      "authentication": "none",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": false,
      "decodeUrl": false,
      "rejectUnauthorized": true,
      "captureHeaders": false,
      "safeHeaders": [],
      "retryRules": {
        "type": "backoff",
        "interval": 1000,
        "limit": 5,
        "multiplier": 2,
        "maxIntervalMs": 20000,
        "codes": [
          429,
          503
        ],
        "enableHeader": true,
        "retryConnectTimeout": false,
        "retryConnectReset": false,
        "retryHeaderName": "retry-after"
      },
      "__scheduling": {
        "stateTracking": {}
      },
      "collectUrl": "'https://<INSERT YOUR BASE URL HERE>/api/v2/auditevents'",
      "collectBody": "{\n    \"limit\": (cursor === NULL) ? 50 : NULL,\n    \"start_time\": (cursor === NULL) ? `${C.Time.strftime(state.latestTime || earliest, '%Y-%m-%dT%H:%M:%S-00:00')}` : NULL,\n    \"end_time\": (cursor === NULL) ? `${C.Time.strftime(latest || Date.now()/1000, '%Y-%m-%dT%H:%M:%S-00:00')}` : NULL,\n    \"cursor\": cursor || NULL\n}",
      "collectRequestHeaders": [
        {
          "name": "Content-Type",
          "value": "'application/json'"
        },
        {
          "name": "Authorization",
          "value": "'Bearer <INSERT YOUR BEARER TOKEN HERE>'"
        }
      ]
    },
    "destructive": false,
    "encoding": "utf8",
    "type": "rest"
  },
  "input": {
    "type": "collection",
    "staleChannelFlushMs": 10000,
    "sendToRoutes": true,
    "preprocess": {
      "disabled": true
    },
    "throttleRatePerSec": "0",
    "breakerRulesets": [
      "1Password"
    ]
  },
  "description": "This uses the 1Password Events API to retrieve information about activity in your 1Password account – like audit events, item usage, and sign-in attempts – and send it to your security information and event management (SIEM) system.",
  "savedState": {},
  "id": "1Password"
}
```
