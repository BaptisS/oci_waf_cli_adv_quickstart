# oci_waf_cli_adv_quickstart

### Initial Configuration for OCI WAF
Just after creating your OCI WAF Policy through OCI console you may want to enable some features currently available only through CLI (Device Fingerprint Challenge, Human Interaction Challenge , Address rate limit , Threat feeds, etc. ) ,  or you may just want to apply a pre-defined configuration to your OCI WAF Policy. (Bulk enablement of protection rules and other settings)   

#### Prerequisites:
- At least one WAF Policy already created (and in the ‘Available' state)  (https://docs.cloud.oracle.com/iaas/Content/WAF/Tasks/managingwaf.htm)
- The OCI CLI already configured to access your OCI Tenant. (https://docs.cloud.oracle.com/iaas/Content/API/SDKDocs/cliinstall.htm) 
- The WAF policy OCID for the desired WAF Policy. 

### 1-    Define the ‘POLOCID’ variable ($POLOCID).

[opc@instance~]$ POLOCID=ocid1.waaspolicy.oc1..aaaaaaaxxxx

### 2-    Review your current OCI WAF Policy configuration

[opc@instance~]$ oci waas waf-config get --waas-policy-id $POLOCID

### 3-   [Optional] Export your 'Threat-Feeds' list (Required only if you intend to enable Threat-Feeds intelligence)
 
[opc@instance~]$ oci waas threat-feed list --waas-policy-id $POLOCID | jq -r '.[]'>/tmp/threat-feeds.json

To enable some or all Threat Feeds, edit the threat-feeds.json file and replace the "action" parameter value with the desired one (BLOCK/DETECT/OFF) for each of the listed feeds.
Remove the last line of the file containing the “Etag” value (ie. W/"2019-07-26T16:40:50.519Z”) before saving it.     

### 4-   Update OCI WAF Policy configuration

[opc@instance~]$ oci waas waf-config update --waas-policy-id $POLOCID --access-rules file:///tmp/accessrules.json --address-rate-limiting file:///tmp/addressrate.json --device-fingerprint-challenge file:///tmp/devfing.json --human-interaction-challenge file:///tmp/humaninter.json --js-challenge file:///tmp/jsch.json --protection-rules file:///tmp/defprot.json --protection-settings file:///tmp/protset.json --threat-feeds file:///tmp/threat-feeds.json


Sample Json files are provided in the appendix of this document. Proposed configuration may need to be adjusted to reflect your exact needs. 

- ***AccessRules.json :*** Basic Access Rule Example. (Need to be modified/removed to fit with your needs)
- ***AddressRate.json :*** Enabled (Allowed rate per single incoming source address = 100 per seconds)  
- ***DevFing.json (Bot Management):*** Enabled in BLOCK mode with CAPTCHA Challenge. 
  - ****Failure Threshold:**** Five / Threshold expiration: 60 sec. 
  - ****Max Address Count:**** 10 / Max Address Count expiration: 60sec
- ***HumanInter.json (Bot Management):***  Enabled in BLOCK mode with CAPTCHA Challenge. 
  - ****Failure Threshold:**** 5 / Threshold expiration: 60 sec 
  - ****Interaction Threshold:**** 3 / recording period: 15 sec
- ***Jsch.json (Bot Management):*** Enabled in BLOCK mode with CAPTCHA Challenge. 
  - ****Failure Threshold:**** 5
- ***DefProt :*** Set of common protection rules enabled in BLOCK Mode.  
- ***ProtSet :*** Default protection settings (HTTP Error 403)
- ***Threat-feeds.json :*** The Threat-Feeds json file must contain the proper "Threat Feeds Keys" belonging to your OCI Tenant / WAF Policy.   You must first export the Feeds list as described in Step 3 above if you want to enable Threat-Feeds Intelligence. 

Once successfully executed the command above will return a "Work Request OCID».
The configuration changes are done asynchronously and can take 10min+ to be replicated safely across all OCI WAF endpoints worldwide. During this time you cannot modify any additional settings.  

### 5-    Check the status for an OCI WAF work request

[opc@instance~]$ oci waas work-request get --work-request-id ocid1.waasworkrequest.oc1..axxxx


### 6-    Create a custom protection rule (Optional) 

***Prerequisite:*** 
- The Compartment OCID (ocid1.compartment.oc1..aaaaaxxxx) where your WAF policy has been created.    

[opc@instance~]$ COMPOCID=ocid1.compartment.oc1..aaaaaxxxx

[opc@instance~]$ oci waas custom-protection-rule create --compartment-id $COMPOCID --display-name Custom_Protection_Rule_01 –template '   SecRule REQUEST_COOKIES "regex matching SQL injection - part 1/2" \           "phase:2,
                                                 \           msg:'Detects chained SQL injection attempts 1/2.',
												 \           id: {{id_1}},
												 \           ctl:ruleEngine={{mode}},
												 \           deny"   
	SecRule REQUEST_COOKIES "regex matching SQL injection - part 2/2" \           "phase:2,
	\           msg:'Detects chained SQL injection attempts 2/2.',
	\           id: {{id_2}},
	\           ctl:ruleEngine={{mode}},
	\           deny"   `


For more information about ModSecurity's open source WAF rules, see Mod Security's documentation.  (https://www.modsecurity.org/CRS/Documentation/making.html ) 

### Links and References : 

- Terraform : 
  - https://www.terraform.io/docs/providers/oci/r/waas_waas_policy.html
- Access Control : 
  - https://docs.cloud.oracle.com/iaas/Content/WAF/Tasks/wafaccesscontrol.htm
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/access-rule.html
- Address Rate Limit:
  The number of allowed requests per second from one IP address. If unspecified, defaults to "1" .
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/address-rate-limiting.html
- Device Fingerprint Challenge (BotManagement) : 
The DFC generates a hashed signature of both virtual and real browsers based on 50+ attributes. These proprietary signatures are then leveraged for real-time correlation to identify and block malicious bots.
The signature is based on a library of attributes detected via JavaScript listeners; the attributes include OS, screen resolution, fonts, UserAgent, IP address, etc. We are constantly making improvements and considering new libraries to include in our DFC build. We can also exclude attributes from the signature as needed.
DFC collects attributes to generate a hashed signature about a client – if a fingerprint is not possible, then it will result in a block or alert action. Actions can be enforced across multiple devices if they share they have the same fingerprint.
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/device-fingerprint-challenge.html
- Human Interaction Challenge (BotManagement): 
  HIC is a countermeasure that allows the proxy to check the user's browser for various behaviors that distinguish a human presence from a bot.
  At high level HIC first allows the client request then start recording any potential user activity such as ‘mouse move, scroll, button click, etc.’ 
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/human-interaction-challenge/update.html
- JavaScript Challenge  (BotManagement) : 
  Javascript Challenge is the function to filter abnormal or malicious bots and allow access to real clients. JavaScript Challenge validates that the client can accept JavaScript with a binary decision
  - https://docs.cloud.oracle.com/iaas/Content/WAF/Tasks/botmanagement.htm
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/js-challenge/update.html
- Protection Rules: 
  Protection rules match web traffic to rule conditions and determine the action to be taken when the conditions are met.
    - https://docs.cloud.oracle.com/iaas/Content/WAF/Tasks/wafprotectionrules.htm
    - https://docs.cloud.oracle.com/iaas/Content/WAF/Reference/protectionruleids.htm
    - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/protection-rule.html
- Threat Feeds Intelligence: 
  You can block requests from IP addresses based on their reputations with various commercial and open source threat feeds.
  - https://docs.cloud.oracle.com/iaas/Content/WAF/Tasks/threatintel.htm
  - https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/waas/threat-feed.html



