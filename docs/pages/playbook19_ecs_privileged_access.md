---
title: ECS Privileged Access issues
keywords: playbook19
sidebar: mydoc_sidebar
permalink: playbook19_ecs_privileged_access.html
---

# Playbook 19: ECS Privileged Access issues

## Introduction

This playbook describes how to configure Dow Jones Hammer to detect ECS privileged access issues.

## 1. Issue Identification

Dow Jones Hammer identifies those ECS privileged access is enabled or not.

When Dow Jones Hammer detects an issue, it writes the issue to the designated DynamoDB table.

According to the [Dow Jones Hammer architecture](/index.html), the issue identification functionality uses two Lambda functions.
The table lists the Python modules that implement this functionality:

|Designation   |Path                  |
|--------------|:--------------------:|
|Initialization|`hammer/identification/lambdas/ecs-privileged-access-issues-identification/initiate_to_desc_ecs_privileged_access_issues.py`|
|Identification|`hammer/identification/lambdas/ecs-privileged-access-issues-identification/describe_ecs_privileged_issues.py`|

## 2. Issue Reporting

You can configure automatic reporting of cases when Dow Jones Hammer identifies an issue of this type. Dow Jones Hammer supports integration with [JIRA](https://www.atlassian.com/software/jira) and [Slack](https://slack.com/).
These types of reporting are independent from one another and you can turn them on/off in the Dow Jones Hammer configuration.

Thus, in case you have turned on the reporting functionality for this issue and configured corresponding integrations, Dow Jones Hammer, as [defined in the configuration](#43-the-ticket_ownersjson-file), can:
* raise a JIRA ticket and assign it to a specific person in your organization;
* send the issue notification to the Slack channel or directly to a Slack user.

Additionally Dow Jones Hammer tries to detect person to report issue to by examining ECS privileged access issues. In case the privileged access is enable **valid JIRA/Slack user**:
* for JIRA: `jira_owner` parameter from [ticket_owners.json](#43-the-ticket_ownersjson-file) **is ignored** and discovered `owner` **is used instead** as a JIRA assignee;
* for Slack: discovered `owner` **is used in addition to** `slack_owner` value from [ticket_owners.json](#43-the-ticket_ownersjson-file).

This Python module implements the issue reporting functionality:
```
hammer/reporting-remediation/reporting/create_ecs_privileged_access_issue_tickets.py
```


## 3. Setup Instructions For This Issue

To configure the detection, reporting, you should edit the following sections of the Dow Jones Hammer configuration files:

### 3.1. The config.json File

The **config.json** file is the main configuration file for Dow Jones Hammer that is available at `deployment/terraform/accounts/sample/config/config.json`.
To identify and report issues of this type, you should add the following parameters in the **ecs_privileged_access** section of the **config.json** file:

|Parameter Name                |Description                            | Default Value|
|------------------------------|---------------------------------------|:------------:|
|`enabled`                     |Toggles issue detection for this issue |`true`|
|`ddb.table_name`              |Name of the DynamoDB table where Dow Jones Hammer will store the identified issues of this type| `hammer-ecs-privileged-access` |
|`reporting`                   |Toggle Dow Jones Hammer reporting functionality for this issue type    |`true`|

Sample **config.json** section:
```
"""
"ecs_privileged_access": {
        "enabled": true,
        "ddb.table_name": "hammer-ecs-privileged-access",
        "reporting": true,
        "remediation": false,
        "remediation_retention_period": 21
    }
```

### 3.2. The whitelist.json File

You can define exceptions to the general automatic remediation settings for specific ECS task definitions. To configure such exceptions, you should edit the **ecs_privileged_access** section of the **whitelist.json** configuration file as follows:

|Parameter Key | Parameter Value(s)|
|:------------:|:-----------------:|
|AWS Account ID|ECS task definition ids(s)|

Sample **whitelist.json** section:
```
"ecs_privileged_access": {
    "123456789012": ["task_definition_arn1", "task_definition_arn2"]
}
```

### 3.3. The ticket_owners.json File

You should use the **ticket_owners.json** file to configure the integration of Dow Jones Hammer with JIRA and/or Slack for the issue reporting purposes.

You can configure these parameters for specific AWS accounts and globally. Account-specific settings precede the global settings in the **ticket_owners.json** configuration file.

Check the following table for parameters:

|Parameter Name       |Description                                                         |Sample Value     |
|---------------------|--------------------------------------------------------------------|:---------------:|
|`jira_project`       |The name of the JIRA project where Dow Jones Hammer will create the issue     | `AWSSEC`        |
|`jira_owner`         |The name of the JIRA user to whom Dow Jones Hammer will assign the issue      | `Support-Cloud` |
|`jira_parent_ticket` |The JIRA ticket to which Dow Jones Hammer will link the new ticket it creates | `AWSSEC-1234`   |
|`slack_owner`        |Name(s) of the Slack channels (prefixed by `#`) and/or Slack users that will receive issue reports from Dow Jones Hammer | `["#devops-channel", "bob"]` |

Sample **ticket_owners.json** section:

Account-specific settings:
```
{
    "account": {
        "123456789012": {
            "jira_project": "",
            "jira_owner": "Support-Cloud",
            "jira_parent_ticket": "",
            "slack_owner": ""
        }
    },
    "jira_project": "AWSSEC",
    "jira_owner": "Support-General",
    "jira_parent_ticket": "AWSSEC-1234",
    "slack_owner": ["#devops-channel", "bob"]
}
```

## 4. Logging

Dow Jones Hammer uses **CloudWatch Logs** for logging purposes.

Dow Jones Hammer automatically sets up CloudWatch Log Groups and Log Streams for this issue when you deploy Dow Jones Hammer.

### 4.1. Issue Identification Logging

Dow Jones Hammer issue identification functionality uses two Lambda functions:

* Initialization: this Lambda function selects slave accounts to check for this issue as designated in the Dow Jones Hammer configuration files and triggers the check.
* Identification: this Lambda function identifies this issue for each account/region selected at the previous step.

You can see the logs for each of these Lambda functions in the following Log Groups:

|Lambda Function|CloudWatch Log Group Name                   |
|---------------|--------------------------------------------|
|Initialization |`/aws/lambda/initiate-ecs-privileged-access`|
|Identification |`/aws/lambda/describe-privileged-access`|

### 4.2. Issue Reporting Logging

Dow Jones Hammer issue reporting functionality uses ```/aws/ec2/hammer-reporting-remediation``` CloudWatch Log Group for logging. The Log Group contains issue-specific Log Streams named as follows:

|Designation|CloudWatch Log Stream Name                               |
|-----------|---------------------------------------------------------|
|Reporting  |`reporting.create_ecs_privileged_access_issue_tickets`|


### 4.3. Slack Reports

In case you have enabled Dow Jones Hammer and Slack integration, Dow Jones Hammer sends notifications about issue identification and reporting to the designated Slack channel and/or recipient(s).

Check [ticket_owners.json](#43-the-ticket_ownersjson-file) configuration for further guidance.

### 4.4. Using CloudWatch Logs for Dow Jones Hammer

To access Dow Jones Hammer logs, proceed as follows:

1. Open **AWS Management Console**.
2. Select **CloudWatch** service.
3. Select **Logs** from the CloudWatch sidebar.
4. Select the log group you want to explore. The log group will open.
5. Select the log stream you want to explore.

Check [CloudWatch Logs documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html) for further guidance.

## 5. Issue specific details in DynamoDB

Dow Jones Hammer stores various issue specific details in DynamoDB as a map under `issue_details` key. You can use it to create your own reporting modules.

|Key          |Type  |Description                       |Example                                         |
|-------------|:----:|----------------------------------|------------------------------------------------|
|`id`       |string|ecs task definition id                 |`ecs-task_definition-id`                            |
|`tags`       |map   |Tags associated with ECS task definition id |`{"Name": "TestKey", "service": "archive"}`|