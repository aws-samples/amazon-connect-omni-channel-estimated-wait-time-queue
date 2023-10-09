# Omni-channel estimated wait time in queue for Amazon Connect

## Introduction

In a digital world, customers demand the ability to communicate using the channels they want including websites, mobile device, social media, email, and voice calling. With the growing complexity of channel diversity, businesses are challenged with providing simple, fast, and economical access to customer service agents. When a commercial customer or state/municipality constituent chooses to connect with a contact center, they have initiated the need for assistance. Despite providing self-service solutions, including virtual agents or bots, there is a prominent demand for live agent interaction, which can lead to extended wait times in queue. 

## Customer challenges
There are many commercial and public sector business cases where customers must interact with a live agent. This could be as simple as a detail requiring negotiation, or as complex as dealing with an immigration or state benefits issue. When a customer calls into a support line (contact center), they expect to receive timely service and are willing to wait for a live interaction. Contact centers want to route calls or chats efficiently to agents, but in many cases they are staffed with fewer agents than incoming contacts. This means that callers or contacts must wait for a live interaction with times that can vary based on conditions like time of day, day of week, promotional marketing ad, or even natural disasters. Unlike in a voice call, chat does not provide session entertainment (like music on hold) while waiting, and when queue times are protracted, waiting without an update or an estimate of wait time can be very frustrating.

## Estimated wait time solution

Estimated Wait Time (EWT) predicts the amount of time an interaction must wait in a queue before being answered by an agent. These predictions consider the queue length, agent availability, and handle time, as well as other system behaviors and variability at a given moment in time. EWT is a valuable metric in a contact center’s customer experience and can be strategically used to enhance customer satisfaction. In this blog, you will learn to deploy an Amazon Connect Customer Queue Flow that provides a smart estimated wait time for voice or chat channels. The EWT metric is unique for each queue or channel. The calculation uses Amazon Connect real-time metrics (Average Queue Answer Time), calculates EWT in AWS Lambda functions, and stores data in Amazon DynamoDB tables as a metric repository. By utilizing DynamoDB, it mitigates Amazon Connect API service quota limits at scale. 

Providing a periodic EWT informs a customer and, used in conjunction with the option for a callback, provides convenience for the customer. Using EWT with a callback also reduces connected call cost and telco cost in the contact center by decreasing real-time queuing.

## Overview
![Architecture Diagram](images/image1.png?raw=true)

In this architecture, you will use an Amazon Connect Customer Queue flow with a prebuilt AWS Lambda function to process a [GetMetricData](https://docs.aws.amazon.com/connect/latest/APIReference/API_GetMetricData.html) API call for QUEUE_ANSWER_TIME Average. The sequence of events is as follows:

1.	The Lambda function performs a QUERY against a DynamoDB table for the metric associated with the specific queue (queue name and channel) that is no older than 1 minute. This will ensure that the metric that the Customer Queue flow is using for the customer is almost real-time.
2.	If there is a value that meets that criteria, the value is returned to Amazon Connect and will be played to the caller while in queue.
3.	If there is not a value that meets the criteria, the Lambda function will then make the GetMetricData API operation call to Amazon Connect to get the real-time metric. The Lambda will put the Queue Answer Time value into the DynamoDB table for future reference. The value is then returned to Amazon Connect to be played and will be played to the caller while in queue.

The metric that Amazon Connect provides for QUEUE_ANSWER_TIME Average is returned in seconds. The Lambda function will round up this value to the nearest minute to normalize the language response that is played to the contact. Example: Value returned from the API call = 336 seconds. This will be rounded up to the nearest minute {=roundup(336/60),0 = 6 minutes}.

DynamoDB is used to store this metric due to its ability to support a large-scale volume of requests. The maximum throughput you can provision per table is 40,000 RCUs (read per second) and 40,000 WCUs (writes per second) by default. These are only soft limits to prevent users from accidentally provisioning excessive throughput. You can request a higher limit to be applied to the account, if needed. The GetMetricData API has a service quota limit of five requests per second, and a BurstLimit ([See API Throttling Quotas](https://docs.aws.amazon.com/connect/latest/adminguide/amazon-connect-service-limits.html)) of eight requests per second. By only calculating the EWT when needed, the maximum number of API calls to Amazon Connect is one per minute per queue, thus allowing the solution to scale to support tens of thousands of calls across hundreds of queues. 

It is recommended that the EWT is played once, as the contact enters the Customer Queue Flow. Since the Queue Answer Time Average will change with new contacts entering the queue and waiting as well as agents answering queued contacts, the EWT time is relevant to the contact at the time of entry to the queue.

## Prerequisites
The solution presented in this blog has the following prerequisites:<br/>
•	An [Amazon Connect](https://aws.amazon.com/connect/) instance  <br/> 
•	Basic knowledge of [AWS Lambda](https://aws.amazon.com/lambda/), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), [AWS CloudFormation](https://aws.amazon.com/cloudformation/), and [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/)  <br/>
•	An active AWS account with a user that has permission to create and modify AWS IAM roles.  <br/>

## Walkthrough

The solution is deployed using an AWS CloudFormation template. This template creates the AWS Lambda function, Amazon DynamoDB table, and AWS IAM Role and Policy. To deploy the template, follow these 6 steps:  <br/> 
Step 1 – Sign in to the AWS Management Console in the same Region that your Amazon Connect instance is in.   <br/>
Step 2 – Download [CloudFormation template](QueueEWT.yaml).  <br/>
Step 3 – AWS CloudFormation: In your preferred Region, create a CloudFormation stack using the template file downloaded in step 2:  <br/>
1.	Select Create Stack.
2.	Select With new resources (standard) from dropdown menu.
3.	In the Template Source box, select Upload a template file radio button.
4.	Select Choose File button. 
5.	Navigate to the downloaded file on your computer QueueEWT.yaml and select/open the file.
6.	Select Next.
![Architecture Diagram](images/image2.png?raw=true)

7.	Enter a unique stack name (e.g. QueueEWT).
8.	Enter the Amazon Connect Instance ID used in this deployment (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx).
This information can be obtained from the Amazon Connect Account Overview page.
![Architecture Diagram](images/image3.png?raw=true)

9.	Set a value for DynamoDbRefreshRate. Note: DynamoDB data refresh rate is 1 minute by default (recommend keeping this as 1, as it will adjust the interval for the refresh of the value in table).
10.	Set a value for Hours. Note: The Hours value is the trailing period of time that the solution will use to determine the Queue Answer Time Average. This default is 2 hours to align to the default real-time metrics in the Amazon Connect console. This can be changed as needed between 1–24.
11.	Select Next.
![Architecture Diagram](images/image4.png?raw=true)

12.	On the Configure Stack Options page, keep all defaults. Select Next. 
13.	On the Review QueueEWT page, keep all defaults.
14.	At the bottom of Review QueueEWT page, select two (2) check boxes to:
o	Acknowledge that CloudFormation will create IAM resources.
o	Verify that the changes are intentional.
15.	Select Create Stack.
![Architecture Diagram](images/image5.png?raw=true)

16.	Under the Event tab, it may take several minutes to create the stack, so refresh this screen to watch for completion.
![Architecture Diagram](images/image6.png?raw=true)

17.	Select the Resources tab to view status.
![Architecture Diagram](images/image7.png?raw=true)

18.	Once Status shows Create_Complete, you can move on to next steps.

Step 4 – Amazon Connect: Associate the new Lambda Function with your Connect instance.
1.	From the Amazon Connect console, select the instance alias that that was specified previously.
![Architecture Diagram](images/image8.png?raw=true)

2.	Select Contact Flows. 
3.	Scroll to AWS Lambda section and select the dropdown arrow for Lambda Functions. 
4.	Find connect-queue-ewt-function and select it. 
5.	Select the + Add Lambda Function button.
![Architecture Diagram](images/image9.png?raw=true)

Step 5 - Amazon Connect – Create a new Customer Queue Flow
1.	Download the sample [A Demo – EWTQueue customer queue flow](ADemo-EWTQueue.json).
2.	From the Routing Menu   – select Contact flows.
3.	Select the dropdown arrow next to the Create contact flow button. 
4.	Select Create Customer queue flow. <br/>
![Architecture Diagram](images/image10.png?raw=true)

5.	A new blank Customer Queue flow template will be opened. Select the arrow next to the Save button. 
![Architecture Diagram](images/image11.png?raw=true)

6.	Choose Import flow (beta) from the dropdown list.
7.	From the pop-up window, select the Select button. <br/>
![Architecture Diagram](images/image12.png?raw=true)

8.	Navigate to the customer queue flow file that was downloaded previously, and then select it.
9.	Select the Import button.<br/>
![Architecture Diagram](images/image13.png?raw=true)

![Architecture Diagram](images/image14.png?raw=true)
NOTE: Contact Flow blocks in box A above represents the voice flow blocks. Contact Flow blocks in box B represents chat flow blocks.

NOTE: The Loop Prompt block will import and show a warning. Publish without editing this block and the prompt will be resolved by name automatically.
![Architecture Diagram](images/image15.png?raw=true)

10.	There are 2 Invoke AWS Lambda function blocks in this customer queue flow: <br/>
![Architecture Diagram](images/image16.png?raw=true)

11.	Open each block and find connect-queue-ewt-function from the Function ARN dropdown list and select it. 
![Architecture Diagram](images/image17.png?raw=true)

12.	Select the Save button to save the block configuration.
13.	Select the Publish button to complete the configuration and publishing of the Customer Queue flow.
![Architecture Diagram](images/image18.png?raw=true)

Step 6 - Amazon Connect: Create a new Contact Flow (inbound)
1.	Download the sample [A Demo – Direct to Agent with EWTQueue contact flow (inbound)](InboundFlow.json).
2.	From the Routing Menu   – select Contact flows.
3.	Select the Create contact flow button.<br/>
![Architecture Diagram](images/image19.png?raw=true)

4.	A new blank contact flow (inbound) flow template will be opened. 
5.	Select the arrow next to the Save button. <br/>
![Architecture Diagram](images/image20.png?raw=true)

6.	Select Import flow (beta) from the dropdown list.
7.	From the pop-up window, select the Select button. <br/>
![Architecture Diagram](images/image21.png?raw=true)

8.	Navigate to the contact flow file that was downloaded in step 6.1 and select it.
9.	Select the Import button.<br/>
![Architecture Diagram](images/image22.png?raw=true)

![Architecture Diagram](images/image23.png?raw=true)

10.	Find the Set customer queue flow block in this Contact flow (inbound).<br/>
![Architecture Diagram](images/image24.png?raw=true)

11.	Open this block. Select the Customer queue flow dropdown. Select A Demo - EWTQueue from the list. 
![Architecture Diagram](images/image25.png?raw=true)

12.	Select the Save button at the bottom of the menu to save the block configuration.
13.	Select the Publish button to complete the configuration and publishing of the Contact flow (inbound).
![Architecture Diagram](images/image26.png?raw=true)

## Validation	
To validate the confirmation, generate data by receiving calls or chats in Amazon Connect. Open the Amazon Connect Contact Control Panel (CCP) to receive those calls or chats.

Initially, make sure that your agents are not in the available state. As there are no Queue Answer Time metrics in the DynamoDB, callers will hear an estimated wait time of 1 minute until the first call is answered and a value is written to the DynamoDB table. Subsequent calls into the contact center with no agents available will drive up the wait time metrics.

### Amazon Connect: Test the inbound contact flow – VOICE
Step 1 - From the Phone numbers menu, assign a phone number (claim a new number or use an existing number already claimed).
![Architecture Diagram](images/image27.png?raw=true)

1.	Select a phone number.
2.	Select A Demo – Direct to Agent with EWTQueue in the Contact flow/IVR dropdown menu.
![Architecture Diagram](images/image28.png?raw=true)

3.	Select – Save button.

Step 2 – Test inbound caller experience
1.	Open the Amazon Connect Contact Control Panel (CCP) and set the status to a non-routable state to have callers be placed in queue.
2.	Place a call into the phone number assigned to the inbound contact flow with no agents available.  Note: Let the call sit in queue for several minutes (5+ minutes is suggested) to build up the Average Queue Answer Time.
3.	After a call has been in queue for several minutes, change the agent’s status to Available and answer the call.
4.	Repeat these steps several times. Each time, listen to the caller experience while in queue – estimated wait time will change as more calls sit in queue and change up the Average Queue Answer Time.

### Amazon Connect – Test the inbound contact flow – CHAT 

Step 1 – Assign the Amazon Connect Test Chat widget to the A Demo – Direct to Agent with EWTQueue.
1.	From the Amazon Connect Dashboard, select Test chat.
![Architecture Diagram](images/image29.png?raw=true)

2.	On the Test Chat page, select Test Settings.
![Architecture Diagram](images/image30.png?raw=true)

3.	From the dropdown, select a demo – Direct to Agent with EWTQueue. 
![Architecture Diagram](images/image31.png?raw=true)

4.	Select the Apply button.

Step 2 – Test inbound chat experience
1.	Open the Amazon Connect Contact Control Panel (CCP) and set the agent’s status to a non-routable state to have the chat be placed in queue.
2.	Using the Test Chat Widget, start a chat using the inbound contact flow with no agents available. Note: Let the chat sit in queue for several minutes (5+ minutes is suggested) to build up the Average Queue Answer Time.
3.	After a chat has been in queue for several minutes, change the status to Available and answer the chat session.
4.	Repeat these steps several times. Each time, watch the chat experience while in queue – estimated wait time will change as more calls sit in queue and change the Average Queue Answer Time.

### Cleanup

In order to delete the resources created by the stack, delete the CloudFormation template. This will remove the Lambda functions, DynamoDB tables, and the IAM Role and Policy. Deleting the Phone number from within the Amazon Connect console that was reserved and used for testing, will ensure that no additional charges are incurred.

### Conclusion

In this post, you have learned how to use services like AWS Lambda and Amazon DynamoDB in conjunction with Amazon Connect. We have also demonstrated how to build a customer queue flow and associate it with the inbound contact flow to provide an estimated wait time for omni-channel engagements. 
By configuring this solution for your contact centers, it will better inform your customers what estimated wait times are and allow them to choose other functions instead of just waiting. 
Try deploying Estimated Wait Time in your contact center queues today along with other customer-centric functions in order to achieve a higher level of customer experience.       

Check the Amazon Connect web page for more information on solutions and the suite of features available for you. Some options that can be presented in lieu of contacts just waiting in queue, are Queued Callback, QnAbot, or other self-service interactions (like scheduling appointments). 
