# Cribl Akamai SIEM Integration REST Collector Pack
----
## About this Pack

This Pack is designed to collect, process, and output Akamai Security Event data via the [Akamai SIEM Integration REST API](https://techdocs.akamai.com/siem-integration/reference/get-configid). 

The Pack includes OCSF and Splunk output processing:
* Security Event data is mapped to the OCSF [Detection Finding [2004] Class](https://schema.ocsf.io/1.4.0/classes/detection_finding).
* Security Event data is mapped to the Splunk `sourcetype=akamaisiem` for compatibility with the [Akamai SIEM Integration App](https://splunkbase.splunk.com/app/4310)

## Deployment
After installing the Pack, you must perform the following:

### Add Akamai Credentials to Cribl Stream
*Note*: If you use an external secret store, the HMAC function *must* modified to reference those secrets using the proper names.

* Obtain an API Key, Secret, and Access Token from your Akamai Control Center
* Open the worker group that will contain the Akamai collector.
* Navigate to **Group Settings > Security > Secrets**
* Create 2 Secrets. 
   * The first will have type **API key and secret key**. This secret should be named `akamai_client_credentials`. Populate the client ID as the API Key and the client secret as the Secret Key.
   * The second will have type **Text**. This secret should be named `akamai_access_token`. Populate the Value field with the Akamai Access Token.

### Configure the Collector
* Update the `in_akamai_siem_integration` Collector with the following:
   * In **Discover**, add the Akamai `configId'`s you want to Collect from.
   * In **Collect**, replace the placeholder `YOUR_AKAMAI_URL` with your actual Akamai URL.
* Perform a **Run > Preview** of the  `in_akamai_siem_integration` Collector to verify that it works correctly.
* Schedule the Collector and ensure State Tracking is enabled (the correct configuration is already included).

## Pack Configurable Items 
The following are the in-Pack configurable items - review/update them as needed. 

### Outputs

Akamai SIEM Security EVents can be configured to output data in either OCSF or normalized JSON (Splunk) format. Enable *only one* format in the `cribl_akamai_siem_security_events` pipeline.

### Variables

The Pack has the following variables:
* `akamai_siem_default_splunk_index`: Default index for the Splunk output - defaults to `akamai_siem`.

## Advanced Deployments

## Issue: Default Collector is Lagging
If your data collection is lagging, it could be that your collection is too voluminous or a DDOS attack is flooding the logs. This can lead to data loss, inaccurate analysis, and potential security risks. The Akamai API being limited in throughput (approximately 2500-3000 EPS), you can't keep up with the volume during an attack. To solve this you must skip ahead whenever the data collection is lagging.

* Configure (like in the section above) and use the `in_akamai_siem_integration_offset_advanced` REST Collector. This collection is mono-threaded, just like the standard one, and still uses offset (which is the best way to ensure every event is collected exactly once). However, state tracking will detect when the collection is behind and will skip ahead when it lags. Jumping ahead will allow you to skip a burst of logs and avoid lagging. The drawback is that your data collection will be incomplete. Completeness VS Recency : the Akamai API only allows you to pick one. Note: this data collection is *monothreaded* as the offset is sent last by Akamai, so it's limited to 2000-4000eps maximum.

* Monitor your lag to verify that most of the time your lag is OK (below 90s typically). There are multiple ways to achieve this; the easiest is just to check your state in your `input->manage state` : it will show you the latest event time and if it's lagging. You can also gather collection stats in Cribl Lake and monitor them in Cribl search with a custom pipeline (see *Monitor the collection lag and duplicates* section for details).

At this point, if your collection is still lagging, your baseline data volume is too high for a single-threaded collection so you'll need to use the  multithreaded Collector from the next section.

## Issue: Require High Volume Collection
If your Akamai Id sends more than 3000 eps, you might want to use the multithreaded option. This method uses a to/from Collector and splits every minute into 4 jobs of 15 seconds. This collection can ingest a lot more events (8-15000 EPS with 4 threads, in theory). Since it doesn't use offset, it cannot guarantee that you receive every log (for example, if the job fails).

Note that this configuration can only handle *one* Id, which you set directly in the url.

* Configure (like in the section above) and use the `in_akamai_siem_integration_multithreaded` REST Collector 
* Replace the YOURE_CONFIG_ID_HERE at the end of the URL with your Akamai Id

Note that this collection **will** generate duplicate data. Akamai will randomly send you data outside the date boundary, data that you already collected a minute ago. To solve this issue, perform the following:

* Enable the `cribl_akamai_siem_events_monitor_lag` Route and enable the `Drop` function in the `cribl_akamai_siem_monitor_rest_collection_lag` pipeline. This will drop events where the `eventtime` is earlier than the time boundary set.
* Optional but recommened : follow the steps in the *Monitor the collection lag and duplicates* section.

If data collection is still lagging constantly, add more threads. To do so, check the discovery itemList to split every minute : 0,15,30,45 means 4 thread of 15 seconds. Having 6 threads can be achieved by having the discovery itemlist set to : 0,10,20,30,40,50 and changing the to and from request parameters to adjust for the 10-second difference instead of 15. But refrain from changing the current config of 4 threads in a minutes, this is a sweet spot.

## Monitor the collection lag and duplicates
The Pack includes functionality to monitor the data ingestion lag via the `cribl_akamai_siem_monitor_rest_collection_lag` pipeline. It uses `now` as ingestion time, and event time to calculate the lag, effectively changing the _time of the event to visually see the event by the time it came in. It also creates a SHA field from the time and the raw log to detect duplicates without sending sensitive information. 

* You will need a destination configured that allows searching the data e.g. Cribl Lake/Search.
* Enable the `cribl_akamai_siem_events_monitor_lag` Route and add a Destination.  
* Monitor the lag (in Cribl Search or similar) with this search : `dataset="mydataset" input="collection:xxx" |timestats span=5s count(), max(lag), min(lag), percentile(lag,90)`
* Monitor duplicates with this search `dataset="xxx"  | summarize count() by sha | where count_ > 1`

## Release Notes

### Version 0.1.0 - 2025-08-15

External Beta
* Contains the pipelines for processing Akamai SIEM Integration Security Events
* Supports either OCSF or Splunk output formats

## Contributing to the Pack
To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## Acknowledgments
Thanks to Sidd Shah - sshah@cribl.io and Simon Duchene - sduchene@cribl.io for creating the Collector configurations and initial pipelines!

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).