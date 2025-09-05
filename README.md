# Cribl Akamai SIEM Integration REST Collector Pack
----
## About this Pack

This Pack is designed to collect, process, and output Akamai Security Event data via the [Akamai SIEM Integration REST API](https://techdocs.akamai.com/siem-integration/reference/get-configid). 

The Pack includes OCSF and Splunk output processing. 
* Security Event OCSF data is mapped to the [Detection Finding [2004] Class](https://schema.ocsf.io/1.4.0/classes/detection_finding).
* Security Event Splunk data is mapped to `sourcetype=akamaisiem` for compatibility with the [Akamai SIEM Integration App](https://splunkbase.splunk.com/app/4310)

## Deployment
After installing the Pack, you must perform the following:

### Add Akamai Credentials to Cribl Stream
*Note*: If you use an external secret store, the HMAC function will have to be modified to reference those secrets using the proper names.

* Open the worker group that will contain the Akamai collector.
* Navigate to **Group Settings > Security > Secrets**
* Create 2 Secrets. 
   * The first will have type **API key and secret key**. This secret should be named `akamai_client_credentials`. Populate the client ID as the API Key and the client secret as the Secret Key.
   * The second will have type **Text**. This secret should be named `akamai_access_token`. Populate the Value field with the Akamai Access Token.

### Install and Configured the Collector
* Add the Event Breaker Ruleset included in *Appendix A* below to your Stream instance under **Processing > Knowledge > Breaker Rulesets > Add Ruleset**. 
* Add the HMAC function included in *Appendix B* via **Processing** > **Knowledge** > **HMAC Functions**
* Add the pre-processing pipeline included in *Appendix C* via **Processing > Pipelines > Add Pipeline > Manage as JSON (button)**

* Add the Collector Source in *Appendix D* to your Stream instance via **Data > Sources > Collectors > REST**. 
  * Update the URL under **Collect URL** with the URL obtained from the Akamai Control Center.
  * Update the list in **Discover** with a list of *configIds* for this endpoint. Information on *configIds* can be found [here](https://techdocs.akamai.com/siem-integration/reference/get-configid).

* Perform a Run > Preview to verify that the Collector works correctly.
* Schedule the Collector and ensure State Tracking is enabled (the correct confihguration is already included).
* Connect your Akamai SIEM REST Collector to the Pack. On the global Routes page, add a new route, specify a filter expression (if using the default Collector name, than something like `__inputId.includes('in_akamai_siem')` will work) and choose the `cribl-akamai-rest` Pack in the Pipeline dropdown.

## Pack Configurable Items 
The following are the in-Pack configurable items - review/update them as needed. 

### Outputs

Akamai SIEM Security EVents can be configured to output data in either OCSF or normalized JSON (Splunk) format. Enable *only one* format in the `cribl_akamai_siem_security_events` pipeline.

### Variables

The Pack has the following variables:
* `akamai_siem_default_splunk_index`: Default index for the Splunk output - defaults to `akamai_siem`.

## Release Notes

### Version 0.1.0 - 2025-08-15

External Beta
* Contains the pipelines for processing Akamai SIEM Integration Security Events
* Supports either OCSF or Splunk output formats

## Contributing to the Pack
To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).


## Appendix A
Consolidated Akamai API Event Breaker

```
{
  "id": "Akamai SIEM Integration Ruleset",
  "minRawLength": 256,
  "rules": [
    {
      "condition": "_raw.includes(\"https://problems.cloudsecurity.akamaiapis.net\")",
      "type": "regex",
      "timestampAnchorRegex": "/^/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 1048576,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/^\\b$/",
      "name": "Akamai Error",
      "fields": [
        {
          "name": "__drop",
          "value": "true"
        }
      ]
    },
    {
      "condition": "true",
      "type": "json",
      "timestampAnchorRegex": "/start/",
      "timestamp": {
        "type": "auto",
        "length": 150
      },
      "timestampTimezone": "local",
      "timestampEarliest": "-420weeks",
      "timestampLatest": "+1week",
      "maxEventBytes": 134217728,
      "disabled": false,
      "parserEnabled": false,
      "shouldUseDataRaw": false,
      "eventBreakerRegex": "/[\\n\\r]+(?!\\s)/",
      "name": "Akamai Security Events",
      "fields": [
        {
          "name": "_raw",
          "value": "JSON.parse(_raw) || _raw"
        }
      ]
    }
  ],
  "description": "Event breaking rules to handle Akamai Multi-JSON responses"
}
```
## Appendix B
Akamai SIEM Integration HMAC Function
```
{
  "stringBuilders": [
    "[\n    method,\n    'https',\n    urlObj.hostname,\n    `${urlObj.pathname}${urlObj.search}`,\n    '', // headers in [k.lower():value] format, but nested scopes not supported\n    '', // content hash, but no body needed\n    `EG1-HMAC-SHA256 client_token=${C.Secret('akamai_client_credentials').apiKey};access_token=${C.Secret('akamai_access_token').value};timestamp=${C.Time.strftime(Date.now() / 1000, \"%Y%m%dT%H:%M:%S%Z\")};nonce=${C.Misc.uuidv4()};`\n].join('\\t')"
  ],
  "headerName": "Authorization",
  "headerExpression": "`${signatureString.split('\\t')[6]}signature=${C.Crypto.createHmac(signatureString, C.Crypto.createHmac([...signatureString.matchAll(/timestamp=([^\\;]+)/g)][0][1], C.Secret('akamai_client_credentials').secretKey, \"sha256\", \"base64\"), \"sha256\", \"base64\")}`",
  "stringDelim": "''",
  "id": "hmac_akamai_edgegrid"
}
```

## Appendix C
Akamai SIEM Integration Pre-processing Pipeline
```
{
  "id": "cribl_akamai_siem_parse_decode",
  "conf": {
    "output": "default",
    "streamtags": [],
    "groups": {
      "oQucZ4": {
        "name": "Parse Attack Data",
        "description": "Handles decoding and reformatting attack data",
        "index": 2,
        "disabled": false
      }
    },
    "asyncFuncTimeout": 1000,
    "functions": [
      {
        "conf": {},
        "filter": "offset || __drop === true",
        "id": "drop",
        "description": "Drop offset context events. This metadata is used for state tracking prior to this pipeline."
      },
      {
        "filter": "true",
        "conf": {
          "mode": "extract",
          "type": "json",
          "srcField": "_raw"
        },
        "id": "serde"
      },
      {
        "filter": "true",
        "conf": {
          "maxNumOfIterations": 5000,
          "activeLogSampleRate": 1,
          "useUniqueLogChannel": true,
          "code": "function decodeAkamaiConfigRule(attackData) {\n\n    let decodedAttackData = {};\n    \n    for (const [key, value] of Object.entries(attackData)) {\n        if (key.startsWith('rule')) {\n            decodedAttackData[key] = Array.from(C.Decode.uri(value).split(';'), (x) => C.Decode.base64(x))\n        }\n    }\n\n    return decodedAttackData\n}\n\nfunction mergeAkamaiConfigData(configData) {\n    let mergedData = [];\n\n    for (const [key, value] of Object.entries(configData)) {\n        for (let i = 0; i < value.length; i++) {\n\n            // let formattedKey = key.replace(/^rule(\\w{2,})$/, '$1').replace(/s$/, '');\n            let formattedKey = key.replace(/s$/, '');\n            formattedKey = (formattedKey === 'rule' ? 'Rule' : formattedKey);\n\n            try {\n                mergedData[i][formattedKey] = value[i]\n            } catch (e) {\n                mergedData[i] = {[formattedKey]: value[i]}\n            }\n        }\n    }\n\n    return mergedData\n}\n\n__e['attackData']['data'] = mergeAkamaiConfigData(decodeAkamaiConfigRule(__e['attackData']));"
        },
        "id": "code",
        "groupId": "oQucZ4",
        "disabled": false
      },
      {
        "filter": "true",
        "conf": {
          "remove": [
            "attackData.rule*"
          ],
          "add": [
            {
              "disabled": false,
              "name": "httpMessage.query",
              "value": "C.Decode.uri(httpMessage.query)"
            },
            {
              "disabled": false,
              "name": "httpMessage.requestHeaders",
              "value": "C.Decode.uri(httpMessage.requestHeaders).split('\\n').filter(String)"
            },
            {
              "disabled": false,
              "name": "httpMessage.responseHeaders",
              "value": "C.Decode.uri(httpMessage.responseHeaders).split('\\n').filter(String)"
            }
          ],
          "keep": []
        },
        "id": "eval",
        "groupId": "oQucZ4",
        "disabled": false
      },
      {
        "filter": "true",
        "conf": {
          "add": [
            {
              "disabled": false,
              "name": "attackData.rules",
              "value": "attackData.data"
            }
          ],
          "remove": [
            "attackData.data"
          ]
        },
        "id": "eval",
        "groupId": "oQucZ4",
        "disabled": false
      },
      {
        "filter": "true",
        "conf": {
          "depth": 5,
          "format": "none",
          "ignoreFields": [],
          "filterExpr": "",
          "digits": 0
        },
        "id": "numerify",
        "disabled": false,
        "groupId": "oQucZ4"
      },
      {
        "filter": "true",
        "conf": {
          "type": "json",
          "dstField": "_raw",
          "fields": [
            "!_*",
            "!__*",
            "!cribl*",
            "!host",
            "!source",
            "*"
          ]
        },
        "id": "serialize",
        "disabled": false
      },
      {
        "filter": "true",
        "conf": {
          "keep": [
            "_raw",
            "_time",
            "cribl*",
            "source*",
            "host"
          ],
          "remove": [
            "*"
          ]
        },
        "id": "eval"
      }
    ],
    "description": "Decode encoded data in Akamai security events"
  }
}
```

## Appendix D
Akamai SIEM Integration Collector Sources JSON

### Akamai SIEM Integration Collector - Generic
```
{
  "type": "collection",
  "ttl": "4h",
  "ignoreGroupJobsLimit": false,
  "removeFields": [],
  "resumeOnBoot": false,
  "schedule": {
    "cronSchedule": "* * * * *",
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
      "timestampTimezone": "UTC",
      "stateTracking": {
        "stateUpdateExpression": "offset ? {[__collectible.id]: {offset: offset}} : ''",
        "stateMergeExpression": "newState !== '' ? Object.assign(prevState, newState) : (prevState || {})",
        "enabled": true
      }
    },
    "resumeMissed": false,
    "enabled": true
  },
  "streamtags": [],
  "workerAffinity": false,
  "collector": {
    "conf": {
      "discovery": {
        "discoverType": "list",
        "itemList": [
          "12345"
        ]
      },
      "collectMethod": "get",
      "pagination": {
        "type": "none"
      },
      "authentication": "hmac",
      "timeout": 0,
      "useRoundRobinDns": false,
      "disableTimeFilter": true,
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
      "collectUrl": "`https://dummy.akamaiapis.net/siem/v1/configs/${id}`",
      "collectRequestParams": [
        {
          "value": "`${state[`${id}`].offset == null ? `${Math.round(Date.now() / 1000) - 5}` : undefined}`",
          "name": "from"
        },
        {
          "name": "limit",
          "value": "'150000'"
        },
        {
          "name": "offset",
          "value": "`${state[`${id}`].offset == null ? undefined : state[`${id}`].offset}`\n"
        }
      ],
      "hmacFunctionId": "hmac_akamai_edgegrid"
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
      "Akamai SIEM Integration Ruleset"
    ],
    "pipeline": "cribl_akamai_siem_parse_decode",
    "output": "devnull"
  },
  "savedState": {},
  "id": "in_akamai_siem_integration"
}
```
### Placeholder
