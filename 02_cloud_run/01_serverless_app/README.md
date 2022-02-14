# Build a Serverless App with Cloud Run that Creates PDF Files

Cloud Run is serverless, so it abstracts away all infrastructure management and lets you focus on building your application instead of worrying about overhead. As a Google serverless product, it is able to scale to zero, meaning it won't incur cost when not used. It also lets you use custom binary packages based on containers, which means building consistent isolated artifacts is now feasible.

## Enable the Cloud Run API

Open the navigation menu and select APIs & Services > Library. Then in the search bar, enter in "Cloud Run" and select the Cloud Run API from the results list.

Click Enable and then hit the back button in your browser twice. 

## Deploy a simple Cloud Run service
Ruby has developed a Cloud Run prototype and would like Patrick to deploy it onto Google Cloud. Now help Patrick establish the PDF Cloud Run service for Pet Theory.

Open a new Cloud Shell session and run the following command to clone the Pet Theory repository:

```bash
git clone https://github.com/rosera/pet-theory.git
```

Then change your current working directory to `lab03`:
```bash
cd pet-theory/lab03
```

Edit `package.json` with Cloud Shell Code Editor or your preferred text editor. In the "scripts" section, add `"start": "node index.js"`, as shown below:
```bash
...
"scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
...
```

Now run the following commands in Cloud Shell to install the packages that your conversion script will be using:
```bash
npm install express
npm install body-parser
npm install child_process
npm install @google-cloud/storage
```
Now open the `lab03/index.js` file and review the code.
The application will be deployed as a Cloud Run service that accepts HTTP POSTs. If the POST request is a Pub/Sub notification about an uploaded file, the service writes the file details to the log. If not, the service simply returns the string "OK".

Review the file named `lab03/Dockerfile`.

The above file is called a manifest and provides a recipe for the Docker command to build an image. Each line begins with a command that tells Docker how to process the following information:

The first list indicates the base image should use node v12 as the template for the image to be created.

The last line indicates the command to be performed, which in this instance refers to "npm start".

To build and deploy the REST API, use Google Cloud Build. Run this command to start the build process:
```bash
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```
The command builds a container with your code and puts it in the Container Registry of your project.

Return to the Cloud Console, open the navigation menu, and select Container Registry > Images. You should see your container hosted:


Return to your code editor tab and in Cloud Shell run the following command to deploy your application:

```bash

gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-central1 \
  --no-allow-unauthenticated \
  --max-instances=1
```
Create the environment variable $SERVICE_URL for the app so you can easily access it:
```bash

SERVICE_URL=$(gcloud beta run services describe pdf-converter --platform managed --region us-central1 --format="value(status.url)")
```
```bash
echo $SERVICE_URL
```
Make an anonymous POST request to your new service:
```bash

curl -X POST $SERVICE_URL
```

This will result in an error message saying "Your client does not have permission to get the URL". This is good; you don't want the service to be callable by anonymous users.

Now try invoking the service as an authorized user:
```bash

curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

If you get the response "OK" you have successfully deployed a Cloud Run service. Well done!

## Trigger your Cloud Run service when a new file is uploaded
Now that the Cloud Run service has been successfully deployed, Create a staging area for the data to be converted. The Cloud Storage bucket will use an event trigger to notify the application when a file has been uploaded and needs to be processed.

Run the following command to create a bucket in Cloud Storage for the uploaded docs:

```bash
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload
```
And another bucker for the processed PDFs:
```bash
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed
```
Now return to your Cloud Console tab, open the Navigation menu and select Cloud Storage. Verify that the buckets have been created (there will be other buckets there as well that are used by the platform.)


In Cloud Shell run the following command to tell Cloud Storage to send a Pub/Sub notification whenever a new file has finished uploading to the docs bucket:

```bash
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload
```
The notifications will be labeled with the topic "new-doc".

Then create a new service account which Pub/Sub will use to trigger the Cloud Run services:

```bash
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```
Give the new service account permission to invoke the PDF converter service:

```bash
gcloud beta run services add-iam-policy-binding pdf-converter --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-central1
```
Find your project number by running this command:

```bash
gcloud projects list
```
Look for the project whose name starts with "qwiklabs-gcp-". You will be using the value of the Project Number in the next command.

Create a PROJECT_NUMBER environment variable, replacing [project number] with the Project Number from the last command:

```bash
PROJECT_NUMBER=[project number]
```
Then enable your project to create Cloud Pub/Sub authentication tokens:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```
Finally, create a Pub/Sub subscription so that the PDF converter can run whenever a message is published on the topic "new-doc".

```bash
gcloud beta pubsub subscriptions create pdf-conv-sub --topic new-doc --push-endpoint=$SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

## See if the Cloud Run service is triggered when files are uploaded to Cloud Storage
To verify the application is working as expected, upload some test data to the named storage bucket and then check Cloud Logging.

Copy some test files into your upload bucket:

```bash
gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload
```
Once the upload is done, return to your Cloud Console tab, open the navigation menu, and select Logging from under the Operations section.

In the first dropdown, filter your results to Cloud Run Revision and click Add. Then click Run Query.

In the Query results, look for a log entry that starts with file: and click it. It shows a dump of the file data that Pub/Sub sends to your Cloud Run service when a new file is uploaded.

Can you find the name of the file you uploaded in this object?


>Note: If you do not see any log entries that begin with "file", try clicking on the "load newer logs" button near the bottom of the page.
Now return to the code editor tab and run the following command in Cloud Shell to clean up your upload directory by deleting the files in it:
```bash
gsutil -m rm gs://$GOOGLE_CLOUD_PROJECT-upload/*
```

## Update the Docker container
With all the files identified, the Dockerfile can now be created.

The package for LibreOffice was not included in the container before, which means it now needs to be added. Let's add these as a RUN command within the Dockerfile.

Open the `Dockerfile` manifest and add the command `RUN apt-get update -y && apt-get install -y libreoffice && apt-get clean` line as shown below:
```bash
FROM node:12
RUN apt-get update -y \
    && apt-get install -y libreoffice \
    && apt-get clean
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

## Deploy the new version of the pdf-conversion service
Open the `index.js` file and add the following package requirements at the top of the file:
```bash
const {promisify} = require('util');
const {Storage}   = require('@google-cloud/storage');
const exec        = promisify(require('child_process').exec);
const storage     = new Storage();
```
Replace the `app.post('/', async (req, res)` with the following code:
```js
app.post('/', async (req, res) => {
  try {
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
  }
  catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})
```
Now add the following code that processes LibreOffice documents to the bottom of the file:
```js
async function downloadFile(bucketName, fileName) {
  const options = {destination: `/tmp/${fileName}`};
  await storage.bucket(bucketName).file(fileName).download(options);
}
async function convertFile(fileName) {
  const cmd = 'libreoffice --headless --convert-to pdf --outdir /tmp ' +
              `"/tmp/${fileName}"`;
  console.log(cmd);
  const { stdout, stderr } = await exec(cmd);
  if (stderr) {
    throw stderr;
  }
  console.log(stdout);
  pdfFileName = fileName.replace(/\.\w+$/, '.pdf');
  return pdfFileName;
}
async function deleteFile(bucketName, fileName) {
  await storage.bucket(bucketName).file(fileName).delete();
}
async function uploadFile(bucketName, fileName) {
  await storage.bucket(bucketName).upload(`/tmp/${fileName}`);
}
```
Ensure your `index.js` file looks like the following:
> Note: To avoid any formatting errors, it's recommended you replace all of the code in your index.js file with this example code.
```js
const {promisify} = require('util');
const {Storage}   = require('@google-cloud/storage');
const exec        = promisify(require('child_process').exec);
const storage     = new Storage();
const express     = require('express');
const bodyParser  = require('body-parser');
const app         = express();
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  try {
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
  }
  catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
async function downloadFile(bucketName, fileName) {
  const options = {destination: `/tmp/${fileName}`};
  await storage.bucket(bucketName).file(fileName).download(options);
}
async function convertFile(fileName) {
  const cmd = 'libreoffice --headless --convert-to pdf --outdir /tmp ' +
              `"/tmp/${fileName}"`;
  console.log(cmd);
  const { stdout, stderr } = await exec(cmd);
  if (stderr) {
    throw stderr;
  }
  console.log(stdout);
  pdfFileName = fileName.replace(/\.\w+$/, '.pdf');
  return pdfFileName;
}
async function deleteFile(bucketName, fileName) {
  await storage.bucket(bucketName).file(fileName).delete();
}
async function uploadFile(bucketName, fileName) {
  await storage.bucket(bucketName).upload(`/tmp/${fileName}`);
}
```
The main logic is housed in these functions:
```js
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
```
Whenever a file has been uploaded, this service gets triggered. It performs these tasks, one per line above:

* Extracts the file details from the Pub/Sub notification.
* Downloads the file from Cloud Storage to the local hard drive. This is actually not a physical disk, but a section of virtual memory that behaves like a disk.
* Converts the downloaded file to PDF.
* Uploads the PDF file to Cloud Storage. The environment variable process.env.PDF_BUCKET contains the name of the Cloud Storage bucket to write PDFs to. You will assign a value to this variable when you deploy the service below.
* Deletes the original file from Cloud Storage.

The rest of index.js implements the functions called by this top-level code.

It's time to deploy the service, and to set the PDF_BUCKET environment variable. It's also a good idea to give LibreOffice 2 GB of RAM to work with (see the line with the --memory option).

Run the following command to build the container:
```bash
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

Now deploy the latest version of your application:
```bash
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-central1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --max-instances=1 \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed
```

## Testing the pdf-conversion service
With LibreOffice part of the container, this build will take longer than the previous one. This is a good time to get up and stretch for a few minutes.

Once the deployment commands finish, make sure that the service was deployed correctly by running:
```bash
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```
If you get the response "OK" you have successfully deployed the updated Cloud Run service. LibreOffice can convert many file types to PDF: DOCX, XLSX, JPG, PNG, GIF, etc.

Run the following command to upload some example files:
```bash
gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload
```
Return to the Cloud Console, open the Navigation menu and select Cloud Storage. Open the -upload bucket and click on the Refresh button a couple of times to see how the files are deleted, one by one, as they are converted to PDFs.

Then click Browser from the left menu, and click on the bucket whose name ends in "-processed". It should contain PDF versions of all files. Feel free to open the PDF files to make sure they were properly converted: