# Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub

System can be designed in which Pet Theory is able to:

1. Receive the HTTP POST request and confirm receipt to the medical lab.
1. Email the test result to the client.
1. Send a text message (SMS) and an email to the client with the test result.

The design isolates each of the above activities and requires:

* A service to perform the request and response for the medical result(s)
* A service to email test results to the client
* A service to send a text message (SMS) to the client
* Pub/Sub to be used for inter-service communication
* Serverless infrastructure to be used for the application architecture

## Create a Pub/Sub topic
When a service publishes a Pub/Sub message, that message must be tagged with a topic. The Lab Report is consumed via the service to be created and publish a message for each report found.

First you need to create a topic that can be used for this task.

Run the following command to create a Pub/Sub topic:


```bash
gcloud pubsub topics create new-lab-report
```
Any service subscribed to the topic "new-lab-report" will be able to consume the message published by the Lab Report Service. In the above diagram you can see two such consumers, Email Service and SMS Service.

Then enable Cloud Run, which will run your code in the cloud:

```bash
gcloud services enable run.googleapis.com
```

## Build the Lab Report Service
This service will serve the purpose of prototyping, so it will only do two things:

* Receive the lab report HTTPS POST containing the report data.

* Publish a message on Pub/Sub.

Back in Cloud Shell, clone the repository needed for this lab:

```bash
git clone https://github.com/rosera/pet-theory.git
```
Move to the lab-service directory:

```bash
cd pet-theory/lab05/lab-service
```
Install the following packages that will be needed to receive incoming HTTPS requests and publish to Pub/Sub:

```bash
npm install express
npm install body-parser
npm install @google-cloud/pubsub
```
These commands update the file package.json to indicate the dependencies required for this service.

You will now edit the package.json file so that Cloud Run knows how to start your code.

Open the package.json file.

In the "scripts" section, add the "start": "node index.js" line as shown below and then save the file.

```json
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```
Create a new file named index.js and add this code to it:

```js
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  try {
    const labReport = req.body;
    await publishPubSubMessage(labReport);
    res.status(204).send();
  }
  catch (ex) {
    console.log(ex);
    res.status(500).send(ex);
  }
})
async function publishPubSubMessage(labReport) {
  const buffer = Buffer.from(JSON.stringify(labReport));
  await pubsub.topic('new-lab-report').publish(buffer);
}
```
The heart of the code is this section:
```js

    const labReport = req.body;
    await publishPubSubMessage(labReport);
```
These two lines do the main work of the service:

1. Extract the lab report from the POST request.
1. Publish a PubSub message containing the newly posted lab report.

Now create a file named Dockerfile and add the code below into it:

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```
This file defines how to package up the Cloud Run service into a container.

## Deploy the lab-report-service
Create a script named `deploy.sh` and paste these commands into it:

```bash
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service
gcloud run deploy lab-report-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --max-instances=1
```
Run the following to make this file executable:

```bash
chmod u+x deploy.sh
```
It's time to deploy the Lab Report Service! Run the deployment script:

```bash
./deploy.sh
```
Due to timing issues, you may get an error the first time you run this command. If you do, simply rerun deploy.sh.

When the deployment has successfully completed, you will see a message similar to this:

```bash
Service [lab-report-service] revision [lab-report-service-00001] has been deployed and is serving traffic at https://lab-report-service-[hash].a.run.app
```
Nice work, the Lab Report Service has been deployed and will consume medical lab results over HTTP. You can now test if the new service is up and running.

## Test the Lab Report Service
To validate the Lab Report Service, simulate three HTTPS POSTs made by the lab company, each containing one lab report. For the purpose of testing, the lab reports created will only contain an ID.

First, put the URL to the report in an environment variable, to make it easier to work with.

```bash
export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service --platform managed --region us-central1 --format="value(status.address.url)")
```
Confirm the LAB_REPORT_SERVICE_URL has been captured:

```bash
echo $LAB_REPORT_SERVICE_URL
```
Create a new file named `post-reports.sh` and add the code below into it:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 12}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 34}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 56}" \
  $LAB_REPORT_SERVICE_URL &
```
The above script will use the curl command to post three distinct ID's to the Lab Service URL. Each command will be run individually in the background.

Make the post-reports.sh script executable:

```bash
chmod u+x post-reports.sh
```
Now test the Lab Report Service endpoint by posting three lab reports to it using the script outlined above:

```bash
./post-reports.sh
```
This script posted three lab reports to your Lab Report Service. Check the logs to see the results!

From the Navigation menu click Cloud Run.

You should now see your newly deployed lab-report-service. Click it.

The next page shows details about your lab-report-service. Click the Logs tab.

On the Logs page are the results of the three test reports that you just posted with the script. Hopefully the returned HTTP codes are 204, meaning OK - not content, shown below. If you don’t see any entries, try scrolling up and down using the scrollbar to the right. This reloads the log.

The next task is to write the SMS and Email services. These services will be triggered when the Lab Report Service publishes a Pub/Sub message on the "new-lab-report" topic.

## The Email Service
Move to the Email Service directory:

```bash
cd ~/pet-theory/lab05/email-service
```
Install these packages so that the code can handle incoming HTTPS requests.

```bash
npm install express
npm install body-parser
```
The above command will update the package.json file, which describes the app and its dependencies. Cloud Run needs to know how to run the code, so add start instruction so that it knows what to do.

Open the package.json file.

In the "scripts" section, add the `"start": "node index.js"` line as shown below and save the file.

```bash
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },

```
Create a new file called index.js and add the following to it:

```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`Email Service: Report ${labReport.id} trying...`);
    sendEmail();
    console.log(`Email Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendEmail() {
  console.log('Sending email');
}
```
This code will run when Pub/Sub posts a message to the service. This is what it does:

* It decodes the Pub/Sub message and then tries to call the sendEmail() function.
* If that succeeds and no exception is thrown, it will return status code 204 so Pub/Sub knows that the message was processed.
* If there is an exception, the service will return status code 500 so that Pub/Sub knows the message was not processed and it should re-post it to the service later.
* Once the communication between services is working, we will add code to the sendEmail() function to actually send the email.

Now create a file named Dockerfile and add the code below into it:

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```
This file defines how to package up the Cloud Run service into a container.

Deploy the Email Service
Create a new file called `deploy.sh` and add the following to it:

```bash
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service
gcloud run deploy email-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated \
  --max-instances=1
```
Make deploy.sh executable:

```bash
chmod u+x deploy.sh
```
Deploy the Email Service:

```bash
./deploy.sh
```
When the deployment is complete, you will see a message similar to this:

```bash
Service [email-service] revision [email-service-00001] has been deployed and is serving traffic at https://email-service-[hash].a.run.app
```
The service has been successfully deployed. You now need to ensure the Email Service is triggered when a Pub/Sub message is available.

### Configure Pub/Sub to trigger the Email Service
Whenever a new Pub/Sub message is published using the "new-lab-report" topic, it should trigger the Email Service. To achieve this task, configure a service account to automatically handle the associated requests for this service.

Create a new service account that will be used to trigger the services responding to Pub/Sub messages:

```bash
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

Give the new service account permission to invoke the Email Service:

```bash
gcloud run services add-iam-policy-binding email-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
```
Next, tell Pub/Sub to invoke the SMS Service when a "new-lab-report" message is published.

Put the project number in an environment variable for easy access:

```bash
PROJECT_NUMBER=$(gcloud projects list --filter="qwiklabs-gcp" --format='value(PROJECT_NUMBER)')
```
Next, enable the project to create Pub/Sub authentication tokens.

Run the code below:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```
Put the URL of the Email Service in another environment variable:

```bash
EMAIL_SERVICE_URL=$(gcloud run services describe email-service --platform managed --region us-central1 --format="value(status.address.url)")
```
Confirm the EMAIL_SERVICE_URL has been captured:

```bash
echo $EMAIL_SERVICE_URL
```
Create a Pub/Sub subscription for the Email Service.

```bash
gcloud pubsub subscriptions create email-service-sub --topic new-lab-report --push-endpoint=$EMAIL_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
Nice work, the service is now set up to respond to Cloud Pub/Sub messages, as a next step validate the code to confirm it meets requirements.

### Test the Lab Report Service and the Email Service together
Using the script created earlier, post to the lab reports again:

```bash
~/pet-theory/lab05/lab-service/post-reports.sh
```
Then open the log (Navigation menu > Cloud Run). You will see the two Cloud Run services in your account.

Click email-service and then click Logs. You will see the result of this service being triggered by Pub/Sub. If you don’t see the messages you expect, you may need to scroll up and down with the scrollbar to get the log to refresh.

Great job! The Email service is now able to write information to the log whenever a message is processed from the Cloud Pub/Sub topic queue! The last task is to write the SMS Service.

## The SMS Service

Create a directory for the SMS Service:

```bash
cd ~/pet-theory/lab05/sms-service
```
Install the packages required to receive incoming HTTPS requests:

```bash
npm install express
npm install body-parser
```
Open the package.json file.

In the "scripts" section, add the "start": "node index.js" line as shown below and save the file.

```json
...
"scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
...
```
Create a new file called index.js and add the following to it:

```js
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`SMS Service: Report ${labReport.id} trying...`);
    sendSms();
    console.log(`SMS Service: Report ${labReport.id} success :-)`);    
    res.status(204).send();
  }
  catch (ex) {
    console.log(`SMS Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendSms() {
  console.log('Sending SMS');
}

```
Now create a file named Dockerfile and add the code below into it:

```Dockerfile
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]

```
This file defines how to package up the Cloud Run service into a container. Now the code has been created, the next step is to deploy the service.

### Deploy the SMS Service
Create a file named `deploy.sh` and add this code into it:

```bash
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service
gcloud run deploy sms-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated \
  --max-instances=1
```
Make deploy.sh executable:

```bash
chmod u+x deploy.sh
```
Deploy the SMS Service:

```bash
./deploy.sh
```
When the deployment is complete, a message similar to this is displayed:

```bash
Service [sms-service] revision [sms-service-00001] has been deployed and is serving traffic at https://sms-service-[hash].a.run.app=
```
The SMS Service is successfully deployed, but it isn't linked to the Cloud Pub/Sub service. Correct that in the next section.

### Configure Cloud Pub/Sub to trigger the SMS Service
As with the Email Service, the link between Cloud Pub/Sub and the SMS service needs to be configured so that messages can be consumed.

Set the permissions to allow Pub/Sub to trigger the SMS Service:

```bash
gcloud run services add-iam-policy-binding sms-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-central1 --platform managed
```
Next, tell Pub/Sub to invoke the SMS Service when a “new-lab-report” message is published.

The first step is to put the URL address of the SMS Service in an environment variable:

```bash
SMS_SERVICE_URL=$(gcloud run services describe sms-service --platform managed --region us-central1 --format="value(status.address.url)")
```
Confirm the SMS_SERVICE_URL has been captured:

```bash
echo $SMS_SERVICE_URL
```
Then create the Pub/Sub subscription:

```bash
gcloud pubsub subscriptions create sms-service-sub --topic new-lab-report --push-endpoint=$SMS_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
Run the test script again to post three lab reports to the Lab Report Service:

```bash
~/pet-theory/lab05/lab-service/post-reports.sh
```
Then open the log (Navigation menu > Cloud Run). You will see the three Cloud Run services in your account.

Click sms-service, then click Logs. You will see the result of this service being triggered by Pub/Sub.

The prototype system has been created and successfully tested. However, resilience, as part of the initial validation process, hasn't been tested.

## Test the resiliency of the system
What happens if one of the services goes down?

Ensure the system can handle this scenario. She wants to test what happens when a service fails by deploying a bad version of the Email Service.

Go back to the email-service directory:

```bash
cd ~/pet-theory/lab05/email-service
```
Add some invalid text to the Email Service application to cause an error.

Edit index.js and add the throw line to the sendEmail() function, as shown below. This will throw an exception, as if the email server was down:

```js
...
function sendEmail() {
  throw 'Email server is down';
  console.log('Sending email');
}
...
```
The addition of this code will crash the service when it is invoked.

Deploy this bad version of the Email Service:

```bash
./deploy.sh
```
When the Email Service deployment has successfully completed, post data to the lab reports again, then go and watch the email-service log status closely:

```bash
~/pet-theory/lab05/lab-service/post-reports.sh
```
Open the log for the bad Email Service: Navigation menu > Cloud Run. When you see the three Cloud Run services in your account, click email-service.

The Email Service is being invoked, but it will keep crashing. If you scroll back a bit in the logs you will find the root cause: “Email server is down”. You can also see that the service returns status code 500, and that Pub/Sub keeps retrying calling the service.

If you look at the logs from the SMS service, you will see that it operates successfully.

Now fix the error in the Email Service to restore the application!

Open the index.js file and remove the throw line you previously entered, then save the file.

Your index.js sendEmail function show now look similar to this:

```js
function sendEmail() {
  console.log('Sending email');
}
```
Deploy the fixed version of the Email Service:

```bash
./deploy.sh
```
When the deployment has finished, click the refresh icon in the top right corner.

You will see how the emails for report 12, 34 and 56 were finally sent, the Email Service returned the status code 204, and Pub/Sub stopped invoking the service. No data was lost; Pub/Sub kept retrying until it was finally successful. This is the foundation of a robust system!
