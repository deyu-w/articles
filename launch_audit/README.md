# Technical Guide for Auditing a Launch Property 

The audit of a Launch property can be tedius if there are a lot of rules and data elements configured in it along the time. The following approach uses Launch API to dump setup from a property into a human readable format to facilitate the audits.

## 1 - Download Launch property to a local folder.

The Engineering team provides two tools for downloading and syncing a Launch property to/from a local copy:
* https://git.corp.adobe.com/marketingtech/reactor-downloader
* https://git.corp.adobe.com/marketingtech/reactor-sync

In this step, we'll use the `reactor-downloader` to dump a Launch property to local folder. 

1. Make sure you have the latest Node.js installed. On Mac you can use `homebrew` to install it
     ```shell
     brew install node
     ```
2. create a folder to download the property to.
     ```shell
     cd ~/Documents/dev
     mkdir node_launch_downloader
     cd node_launch_downloader
     ```
3. Install the `react-downloader` to the folder. This step is normally not necessary for running a package with `npx`. But we need to install it locally so that any further modificaiton to the package can be done to get the information we need.
    ```shell 
    npm install @adobe/reactor-downloader
    ```
4. Make sure you have Adobe IO integration created. Details can be found in the [Access Tokens](https://developer.adobelaunch.com/api/guides/access_tokens/) document.
5. Make the following change(between code starts and ends) at line 121 of the `node_modules/@adobe/reactor-downloader/bin/download/ruleComponents.js` to write rule-ruleComponents relationships to a local file, namely `rule_ruleComponents_mapping.csv`, while the proeprty is downloaded.
     ```javascript
      // create a name that links to the original file
      const sanitizedName = '_' + sanitize(ruleComponent.attributes.name, {
        replacement: '_'
      });
      if (!fs.existsSync(`${ruleRuleComponentsPath}/${sanitizedName}`)) {
        fs.symlinkSync(
          `../../../rule_components/${ruleComponent.id}`,
          `${ruleRuleComponentsPath}/${sanitizedName}`,
          'dir'
        );
      }
      
      //deyu code starts
      let eventT = "";
      if (ruleComponent.attributes.delegate_descriptor_id.indexOf('::actions::') !== -1) {
        eventT = "action";
      } else if (ruleComponent.attributes.delegate_descriptor_id.indexOf('::events::') !== -1) {
        eventT = "events"
      } else if (ruleComponent.attributes.delegate_descriptor_id.indexOf('::conditions::') !== -1) {
        eventT = "conditions"
      }

      let mapping = rule.id + "," + rule.attributes.name + "," + ruleComponent.attributes.rule_order + "," + ruleComponent.id + "," + ruleComponent.attributes.name + "," + ruleComponent.attributes.order + "," + eventT + "," + ruleComponent.attributes.delegate_descriptor_id + "\n";
      try {
        fs.appendFileSync( `${propertyDirectory}/rule_ruleComponents_mapping.csv`, mapping );
        //console.log('The "data to append" was appended to file!');
      } catch (err) {
        /* Handle the error */
        console.log(err);
        process.exit(0);
      }
      //deyu code ends
     ```
6. Download the property to local. The program will ask you which property to download. If you run into an error of `metascope`, check the correct metascope on the JWT tab of the Adobe IO integration you created on adobe.io, then update `index.js` under `.../node_launch_downloader/node_modules/@adobe/reactor-downloader/bin` with it.
     ```shell
     npx @adobe/reactor-downloader --env=production --private-key=/Users/deyuwang/.ssh/adobeio/flybuys/private.key --org-id=36F70D835D7628070A495C99@AdobeOrg --tech-account-id=28A5576C5E8BDBF60A495EA5@techacct.adobe.com --api-key=8b4261ca21f24c14b7ba487419axxxxx --client-secret=xxxxxx-52b9-4fde-99ea-7dd5a8e376bd
     ```

## 2 - Analyze the downloaded property

Once the property is downloaded, we can perform further analysis on data elements, rules and their relationships. For example, the following javascript code can be used to output relationships between data elements and rules/extensions.

```javascript
var findInFiles = require('find-in-files');
var finder = require('findit')(process.argv[2] || 'PR46ae4ef2c7264ce6bbf345d68f3680e3/rules');
const fs = require('fs');
var path = require('path');

function find_in_property(term) {
    var search_term = [
        "_satellite.getVar\\([\"']" + term + "[\"']\\)",
        "%" + term + "%" 
    ];
    for (i=0;i<2;i++) {
        findInFiles.find({'term': search_term[i], 'flags': 'ig'}, 'PR46ae4ef2c7264ce6bbf345d68f3680e3/rule_components', 'data.json$')
        .then(function(results) {
            for (var result in results) {
                var res = results[result];
                let match_result = result.match(/rule_components\/(.*)\//);
                let mapping = term + ',' + res.matches[0] + ',' + res.count + ',rule_component,' + match_result[1] + "\n";
                try {
                    fs.appendFileSync( `./ruleComponent_DE_mapping.csv`, mapping );
                  } catch (err) {
                    /* Handle the error */
                    console.log(err);
                    process.exit(0);
                  }
            }
        });

        findInFiles.find({'term': search_term[i], 'flags': 'ig'}, 'PR46ae4ef2c7264ce6bbf345d68f3680e3/extensions', 'data.json$')
            .then(function(results) {
                for (var result in results) {
                    var res = results[result];
                    let match_result = result.match(/extensions\/(.*)\//);
                    let mapping = term + ',' + res.matches[0] + ',' + res.count + ',extension,' + match_result[1] + "\n";
                    try {
                        fs.appendFileSync( `./ruleComponent_DE_mapping.csv`, mapping );
                      } catch (err) {
                        /* Handle the error */
                        console.log(err);
                        process.exit(0);
                      };
                }
            });
    }
}
var data_elements = ['aa_device',
    'AAM Integration cookie',
    'aa_pagename',
    'aa_page_url',
    'adobe_campaign',
    'angular_code_base',
    'build_date',
    'campaign_tag_parser',
    'channel',
    'clicked_link',
    'contact_id',
    'cookie_domain',
    'dl_maxt',
    'edm_cta_offer_id',
    'event_action',
    'event_category',
    'event_label',
    'faq_question',
    'fbmax_col_link_mode',
    'FbMax Offer Name',
    'fbmax_pageload_events',
    'flybuys_customer',
    'flybuys_velocity_point_transfer',
    'ga_campaign',
    'hierarchy',
    'is_prod',
    'member_login_status',
    'member_max_status',
    'member_proxy_id',
    'member_registration_step',
    'MIGRATION_aa_referrer',
    'MIGRATION_FMS to Ping_pageName - DE input',
    'MIGRATION_JoinFlow_pageName - dD input',
    'MIGRATION_JoinFlow_pageName - DE input',
    'MIGRATION_LoginFlow_pageName - DE input',
    'MIGRATION_pageName - JS',
    'MIGRATION_site section - dD input',
    'offer_id',
    'offer_id_campaign',
    'offer_type',
    'pageview_url',
    'search_query',
    'section',
    'sid_pid',
    'sort_by_type',
    'start_of_visit_cookie',
    'store_id',
    'sub_section',
    'TEST_aa_new_pagename',
    'TEST_aa_pagename',
    'TEST_aa_pagename_dataLayer',
    'TEST_DE_Landing Page',
    'TEST_lookup DigitalData PageName',
    'TEST_member_proxy_id',
    'TEST_page lookup',
    'toggle_title',
    'url_last_part',
    'window_location_hostname'];


for (var searchTerm in data_elements) {
    find_in_property(data_elements[searchTerm]);
}
```

2. Once all the raw materials are ready we can import them into an Excel spreadsheet for further analysis.

---

[Go Back](./README.md)