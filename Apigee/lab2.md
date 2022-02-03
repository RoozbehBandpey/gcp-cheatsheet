Task 1. Create a prod environment and environment group
In this task, you create a new environment group and environment for an organization.

Examine eval environment group and environment
Select the Apigee console tab in your browser window.

On the left navigation menu, select Admin > Environments > Overview.

This lab uses an Apigee evaluation organization. By default, an evaluation organization has a single environment named "eval."
Click on the eval environment.

This page shows the details for the eval environment. The eval environment is in the eval-group environment group, and the environment is ready for deployment.

eval environment

"Ready for deployment" means that API proxies can be deployed to the environment. It does not mean, however, that the runtime is ready to run API proxies that are deployed to the eval environment. If the runtime is not yet available, you will be able to deploy API proxies to the environment, but the proxies will not accept API requests until the runtime is up, and the eval environment is attached. Once the runtime is up and the environment is attached, the runtime will query the management plane for the current API proxy deployments, and the runtime will host the API proxy and accept traffic.

Navigate to Admin > Environments > Groups.

The page should show the default environment group named eval-group. It shows the hostnames and environments that are in the environment group. You should see the eval environment, as well as a long hostname.

environment groups

Click in the eval-group box, and then select the edit (pencil) icon (edit icon).

On this page you can modify the hostnames and environments associated with the environment group. Leave them as specified.

Create a prod environment
Navigate to Admin > Environments > Overview.

Click on +Environment.

Specify the following environment settings:

Property	Value
Display name	prod
Environment name	prod
Proxy type	select Programmable (policy-based development)
Deployment type	select Proxy
The "Proxy type" and "Deployment type" settings control the type of proxy development and deployment you will do within an environment. "Programmable" policy-based development, developed using the "Proxy" deployment type, can be done using the Apigee UI, and is the type of development and deployment used in the labs for this course.
Click Create.

You should get a message that the environment has been defined. The new prod environment is marked Pending Provisioning.

pending provisioning

Shortly after, you should see the message that the environment is ready for use.

ready for use

Create a prod-group environment group
Navigate to Admin > Environments > Groups.

You should see a warning that the prod environment is unassigned.

Click on +Environment Group.

Name the group prod-group, and click Add.

Click in the prod-group box, and then select the edit (pencil) icon (edit icon).

Replace the hostname with:

api-[PROJECT].apigee-apijam.dev
Copied!
You must replace [PROJECT] with your Google Cloud Project ID. You may copy the project ID from the box on the left:

copy project ID

The hostname for the prod environment starts with "api-", and the eval environment hostname starts with "api-test-".

Click Save.

In the Environments pane on the same page, click the plus button (plus button) to add an environment to the environment group.

Select the prod environment, and click Add.

If prod does not show as an option in the "Add Environment" box, force a hard refresh of the web browser tab. In Chrome, you can do this by holding the shift key and clicking the refresh icon.
Task 2. Create a new target server for the eval environment
In this task, you create a new target server configuration for the eval environment.

Create a new target server
On the left navigation menu, select Admin > Environments > Target Servers.

The eval environment should be selected in the upper left. If the prod environment is selected, select the eval environment.

Click +Target server.

Specify the following target server values:

Property	Value
Enabled	selected
Name	TS-Retail
Host	gcp-cs-training-01-test.apigee.net
Protocol	select HTTP
Port	443
SSL	selected
When the SSL box is selected, the dialog shows additional configuration fields. Accept their default values.

Confirm that you have selected the Enabled checkbox. If you do not enable the target server, deploying the API proxy will return an error indicating that the target server does not exist.
Click Add.

Confirm that your values match the values in the image.

target server configuration

Task 3. Create a target server in the prod environment
In this task, you create a target server for the prod environment.

Navigate to Admin > Environments > Target Servers.

Select the prod environment in the dropdown.

Click +Target server.

Specify the following target server values:

Property	Value
Enabled	selected
Name	TS-Retail
Host	gcp-cs-training-01-prod.apigee.net
Protocol	select HTTP
Port	443
SSL	selected
These settings are identical to the settings for the eval target server, except for the Host.

When the SSL box is selected, the dialog shows additional configuration fields. Accept their default values.

Confirm that you have selected the enabled checkbox. If you do not enable the target server, deploying the API proxy will return an error indicating that the target server does not exist.
Click Add.

Task 4. Update the API proxy to use the target server
In this task, you modify your retail API proxy to use the target server instead of the hardcoded target URL.

Update the API proxy
Navigate to Develop > API Proxies.

Select the retail-v1 proxy, and then click the Develop tab.

You are modifying the version of the retail-v1 proxy that was created during Lab 1.

In the Navigator pane on the left, in the Target Endpoints box, click default.

The TargetEndpoint configuration is displayed in the Code box.

In the HTTPTargetConnection section, on line 13, is the hardcoded URL.

Replace the hardcoded URL with a reference to the target server you created.

Replace

    <HTTPTargetConnection>
        <URL>https://gcp-cs-training-01-test.apigee.net/training/db</URL>
    </HTTPTargetConnection>
Copied!
with

    <HTTPTargetConnection>
        <LoadBalancer>
            <Server name="TS-Retail"/>
        </LoadBalancer>
        <Path>/training/db</Path>
    </HTTPTargetConnection>
Copied!
Note that /training/db is now specified in the Path element. The target server, TS-Retail, specifies the hostname that will be used. Combined, the URL used for the call to the backend will be the same as the hardcoded URL in the eval environment. When deployed to the prod environment, the hostname will automatically change for the prod backend without any changes to the code.

Click Save to save the changes.

Because the first revision of the proxy was deployed, you will get a message box indicating that a deployed revision cannot be overridden, and that a new revision was created. This means that the previous revision is still deployed.

saved as new revision

Click Deploy to eval, and to confirm that you want the new revision deployed to the eval environment, click Deploy.

Leave the "Service Account" field empty.
Check deployment status
A proxy that is deployed and ready to take traffic will show a green status.

deployed

When a proxy is marked as deployed but the runtime is not yet available and the environment is not yet attached, you will see a yellow caution sign. Hold the pointer over the Details text to see the current status.

not deployed

If the proxy is deployed and shows as green, your proxy is ready for API traffic. If your proxy is not deployed because there are no runtime pods, you can check the status of provisioning.

Check provisioning dashboard
In the Google Cloud Console, navigate to Compute Engine > VM instances.

Click on the External IP for the lab-startup VM.

external IP

If you see a redirect notice page, click the link to the external IP address.

A new browser window will open. Lab startup tasks are shown with their progress.

Create proxies, shared flows, target servers should be complete when you first enter the lab, allowing you to use the Apigee console for tasks like proxy editing.
Create API products, developers, apps, KVMs indicates when the runtime is available and those assets may be saved.
Create KVM data, proxies handle API traffic indicates when the eval environment has been attached to the runtime and the deployed proxies can take runtime traffic.
lab startup dashboard

In this case, you need to wait for Create KVM data, proxies handle API traffic to complete.

While you are waiting
Load balancing across backend servers — documentation on load balancing across multiple backends using TargetServer

Environments and environment groups

Task 5. Add the prod environment to the runtime instance
You could now deploy proxies to the prod environment, but they will not run unless the prod environment is associated with an instance.

Add the prod environment to the instance
In Cloud Shell, make the following curl call to create an environment variable with the name of the instance in the organization:

export INSTANCE_NAME=$(curl -q -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances" | jq --raw-output '.instances[0].name'); echo "export INSTANCE_NAME=${INSTANCE_NAME}" >> ~/.profile; echo "INSTANCE_NAME=${INSTANCE_NAME}"
Copied!
Check the status of the instance:

curl -q -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances"
Copied!
A state of CREATING indicates that the runtime instance is not yet available, and the state of ACTIVE indicates that the runtime instance is available. Once the runtime instance is ACTIVE, the eval environment will begin attaching to the instance.

Check the status of the eval attachment using this command:

export ATTACHING_ENV=eval; echo "waiting for ${ATTACHING_ENV} attachment"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ATTACHING_ENV}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ATTACHING_ENV}" ]] || break; echo -n "."; sleep 5; done; echo; echo "${ATTACHING_ENV} environment attached";
Copied!
The attachment of the eval environment must be complete before you can attach the prod environment. Wait for the eval environment to be attached.

Once the eval environment has finished attaching to the runtime instance, attach the prod environment to the instance:

curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" -X POST "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances/${INSTANCE_NAME}/attachments" -d '{ "environment": "prod" }' | jq
Copied!
If you get an error message, look at the details of the message.

A NOT_FOUND error might indicate that the prod environment was not created successfully.

A FAILED_PRECONDITION error with a message that "the resource is locked by another operation indicates that the full provisioning of the eval environment may not have yet completed. In this case, you need to wait until the eval environment is attached to the runtime.

When you have successfully begun the process of attaching the prod environment to the runtime, you should see a return message similar to this:

{
  "name": "organizations/qwiklabs-gcp-01-819691af8473/operations/c4e1a09f-05d2-4c46-95ed-559457507379",
  "metadata": {
    "@type": "type.googleapis.com/google.cloud.apigee.v1.OperationMetadata",
    "operationType": "INSERT",
    "targetResourceName": "organizations/qwiklabs-gcp-01-819691af8473/instances/eval-europe-west1-d/attachments/c2e04a79-15e6-4656-9d25-f618080b57fb",
    "state": "IN_PROGRESS"
  }
}
Copied!
Check the status of the prod attachment with this command:

export ATTACHING_ENV=prod; echo "waiting for ${ATTACHING_ENV} attachment"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ATTACHING_ENV}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ATTACHING_ENV}" ]] || break; echo -n "."; sleep 5; done; echo; echo "${ATTACHING_ENV} environment attached";
Copied!
Leave this command running in Cloud Shell.

The eval environment should be ready to serve traffic, so you can test the API proxy in the eval environment now.

Deploy the proxy to the prod environment
In the Apigee console, click the Develop tab.

Click the Deploy to eval dropdown, and click Deploy next to prod to deploy to the prod environment.

Confirm that you want to deploy the API proxy to prod by clicking Deploy.

The proxy will not finish deploying until the prod environment has fully attached to the runtime instance.

Task 6. Test the modified API proxy in the eval environment
In this task, you use the debug tool to test that the updated proxy still successfully calls the backend service.

Select the Apigee console tab in your browser window, and return to the retail-v1 proxy.

Click the Debug tab.

In the Start a debug session pane, on the environment (Env.) dropdown, select eval.

Click Start Debug Session.

If you get error messages in red boxes toward the top of the screen, with descriptions like "Error fetching debug transactions" or "List debug session transaction error," your debug session may still work correctly.
In Cloud Shell, send a request to your API proxy by using this curl command:

curl -X GET "https://api-test-${GOOGLE_CLOUD_PROJECT}.apigee-apijam.dev/retail/v1/categories"
Copied!
A transaction for this request should appear in the Transactions pane on the left. When a transaction is selected, you'll see a trace of the request and response through Apigee. You should see a 200 Status code if your backend URL was correctly set and you correctly updated the Send Requests URL. In the Phase Details pane you should see the Response Content containing the list of categories:

"debug response content"

Selecting the Request sent to target server shows that the request was sent to the test backend:

"eval backend"

Apigee debug traffic is retrieved by asynchronously polling for new API calls, so there will be a delay between when an API request is completed, and when it shows in the debug tool.
Task 7. Test the API proxy in the prod environment
In this task, you use the debug tool to test that the proxy, when deployed to the prod environment, calls the prod backend service.

Check to see whether the prod environment is attached yet
In Cloud Shell, wait for the prod environment to finish attaching to the runtime instance.

You ran a command at the end of Task 5 that indicates when the prod environment is attached.
In the Apigee console, click the Debug tab.

In the Start a debug session pane, on the environment (Env.) dropdown, select prod.

Click Start Debug Session.

In Cloud Shell, send the following curl command:

curl -X GET "https://api-${GOOGLE_CLOUD_PROJECT}.apigee-apijam.dev/retail/v1/categories"
Copied!
A transaction for this request should appear in the Transactions pane on the left. When a transaction is selected, you'll see a trace of the request and response through Apigee. You should see a 200 Status code if your prod backend URL was correctly set and you correctly updated the Send Requests URL. You can also see that the prod hostname (gcp-cs-training-01-prod.apigee.net) is used for the call.

"prod backend"

