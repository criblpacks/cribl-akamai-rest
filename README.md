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
* Connect your Akamai SIEM REST Collector to the Pack. On the global Routes page, add a new route, specify a filter expression (if using the default Collector name, than something like `__inputId.includes('in_akamai_siem')` will work) and choose the `cribl-akamai-rest` Pack in the Pipeline dropdown.

## Pack Configurable Items 
The following are the in-Pack configurable items - review/update them as needed. 

### Outputs

Akamai SIEM Security EVents can be configured to output data in either OCSF or normalized JSON (Splunk) format. Enable *only one* format in the `cribl_akamai_siem_security_events` pipeline.

### Variables

The Pack has the following variables:
* `akamai_siem_default_splunk_index`: Default index for the Splunk output - defaults to `akamai_siem`.

## Advanced Deployments


## Release Notes

### Version 0.1.0 - 2025-08-15

External Beta
* Contains the pipelines for processing Akamai SIEM Integration Security Events
* Supports either OCSF or Splunk output formats

## Contributing to the Pack
To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).