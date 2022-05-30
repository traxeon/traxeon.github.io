---
title: Job to Trigger Deduplication Ticket Creation
date: 2022-05-23 12:00:00 -500
published: true
categories: [servicenow,data,duplication]
tags: [data,quality]
---

# Job to Trigger Deduplication Ticket Creation

Sometimes it is necessary to align records into a table an trigger the deduplication engine so that a person may merge
the records in the platform for better reconciliation without losing data fidelity.

The following code allows for the triggering of the deduplication function on ServiceNow through targeted specification.
- cmdbTable: set this for the target table/class for the cleanup
- matchField: set this for the field or fields that should be used for duplication reduction; for hardware consider using serial number which is more specific

Because this job targets a table, any records that exist in other tables through misclassification should be moved to the proper class.
It is not recommended to use the sameClass=False switch when using Name as a search field.  The more inaccurate the data is, the 
more important it is to get the combination of values exactly right without trigger a high volume of false positive tickets.

```javascript

/**
 * CMDBDeduplicateTaskUtil().createDeduplicateTask() example
 * 
 * OVERVIEW
 * 
 * This script will allow you to search the CMDB
 * for records that may be duplicates based on the field
 * and filter definitions of your chosing
 * 
 */

try {

    //SET INITIAL VARIABLES

    //Set the table that you would like to check for duplicates - note this MUST be a CMDB table
    var cmdbTable = 'cmdb_ci_hardware';

    //Add a filter as an encoded query if desired - only objects within this filter will be matched
    // this query filters items are Apple computers located in New York (NY)
    var filterQuery = 'sys_class_name=cmdb_ci_computer^sys_class_name=cmdb_ci_computer^manufacturer=67f8e2cd1b3e8810416e8550cd4bcb73';

    //Which field shall we use to check for duplicates?
    var matchField = 'name';

    //Do matches need to have the same cmdb_ci class? - This prevents child device types from matching if enabled i.e. Computer, Server
    var sameClass = true;

    //Should empty values be excluded from match?
    var excludeEmpty = true;

    //Minimum number of duplicates required to create a de-duplication task - must be at least 2
    var numberOfMatches = 2;


    //USE GLIDE AGGREGATE TO IDENTIFY POSSIBLE DUPLICATES

    var dupeSetAgg = new GlideAggregate(cmdbTable);
    if (filterQuery) dupeSetAgg.addEncodedQuery(filterQuery);
    dupeSetAgg.addAggregate('COUNT', matchField);
    dupeSetAgg.orderByAggregate('COUNT', matchField);
    dupeSetAgg.groupBy(matchField);
    if (sameClass) dupeSetAgg.groupBy('sys_class_name');
    if (excludeEmpty) dupeSetAgg.addNotNullQuery(matchField);

    //Group by domain if domain separation is enabled (prevent matches across domains)
    if (gs.getProperty('glide.sys.domain.partitioning') == 'true') dupeSetAgg.groupBy('sys_domain');

    //Must have at least 2 CI records that match
    if (numberOfMatches < 2) numberOfMatches = 2;
    dupeSetAgg.addHaving('COUNT', matchField, '>=', parseInt(numberOfMatches));

    dupeSetAgg.query();


    //LOOP THROUGH EACH SET OF MATCHES

    while (dupeSetAgg.next()) {
        var dupeRecordCount = dupeSetAgg.getAggregate('COUNT', matchField);
        var dupeRecordsList = [];
        var dupeAggQuery = dupeSetAgg.getAggregateEncodedQuery();
        var dedupeShortDesc = 'Found ' + dupeRecordCount + ' records where ' + matchField + ' = ' + dupeSetAgg.getValue(matchField) + '\n';
        var dedupeDescription = '';

        gs.log(dedupeShortDesc, 'Dedupe Task Script');

        //Prepare notes - Header row
        dedupeDescription += 'NAME | CREATED ON | UPDATED ON'


        //GATHER ADDITIONAL DEVICE DETAILS (NEED SYS_IDs)

        var dupeDevices = new GlideRecord(cmdbTable);
        dupeDevices.addEncodedQuery(dupeAggQuery); //Use the exact query that the aggregate used for this group
        dupeDevices.query();

        while (dupeDevices.next()) {
            var devName = dupeDevices.getValue('name');
            var devCreated = dupeDevices.getDisplayValue('sys_created_on');
            var devUpdated = dupeDevices.getDisplayValue('sys_updated_on');
            var devId = dupeDevices.getUniqueValue();

            //Add CI to notes
            dedupeDescription += '\n' + devName + ' | ' + devCreated + ' | ' + devUpdated;

            //Add sys_id to list of matched records for this group
            dupeRecordsList.push(devId);
        }
        gs.log(dedupeDescription, 'Dedupe Task Script');


        //CREATE DEDUPLICATION TASK

        //Convert array to string - .createDuplicateTask() expects a comma separated string
        var deviceIdString = dupeRecordsList.join();

        var dedupeUtil = new CMDBDuplicateTaskUtils();
        var dedupeTaskId = dedupeUtil.createDuplicateTask(deviceIdString);


        //UPDATE DEDUPLICATION TASK - ADD DETAILS

        if (dedupeTaskId) {

            var dedupeTask = new GlideRecord('reconcile_duplicate_task');
            dedupeTask.get(dedupeTaskId);
            dedupeTask.setValue('short_description', dedupeShortDesc);
            dedupeTask.work_notes = dedupeDescription;
            dedupeTask.update();

            gs.log('>>>Created de-duplication task ' + dedupeTask.getValue('number') + ' for ' + devName, 'Dedupe Task Script');
	    gs.info('>>>Created de-duplication task ' + dedupeTask.getValue('number') + ' for ' + devName, 'Dedupe Task Script');

        }

    }

} catch (er) {
    gs.log(er, 'Dedupe Task Script');

}
```
