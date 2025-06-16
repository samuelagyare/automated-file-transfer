# automated-file-transfer
Automated File Transfer: Browser → Google Cloud Storage → Google Drive

This repository contains the code and instructions for a Proof of Concept (POC) that demonstrates a robust, automated file transfer system. Users can upload files through a simple web interface directly to a Google Cloud Storage (GCS) bucket. A backend process then automatically and securely transfers the file to a specified Google Drive folder.

This project is an excellent demonstration of a modern, event-driven, serverless architecture on Google Cloud Platform (GCP).

Features
Direct Browser Upload: Files are streamed directly from the user's browser to GCS, bypassing any intermediary server and improving upload speed.

Resumable Uploads: Built on the GCS resumable upload protocol, making it reliable for large files and resilient against unstable network connections.

Serverless Backend: Utilizes an event-driven Google Cloud Run function that only runs when a file is uploaded, making it highly cost-effective and scalable.

Automated Workflow: The entire process, from GCS staging to Google Drive transfer and cleanup, is 100% automated.

Secure by Design: Leverages dedicated IAM service accounts with fine-grained permissions to ensure secure communication between services.

Architecture
This project uses the following Google Cloud services and technologies:

Frontend: A static HTML, CSS, and JavaScript page provides the user interface for file selection and upload.

Staging Storage: Google Cloud Storage (GCS) acts as the initial, high-performance destination for the direct file upload.

Backend Logic: A Google Cloud Run function (2nd Gen) contains the Node.js code that handles the transfer logic.

Event Trigger: Eventarc creates a trigger that invokes the Cloud Run function whenever a new file is successfully created in the GCS bucket.

Final Destination: The Google Drive API (v3) is used to stream the file into the final destination folder.

How It Works
A user selects a file on the frontend web page and clicks "Upload."

The browser initiates a resumable upload session with GCS and sends the file directly to the specified bucket.

Upon successful upload (storage.object.finalized event), the Eventarc trigger fires.

The trigger invokes the Cloud Run function, passing along the event data (file name, bucket, etc.).

The Node.js function authenticates to the Google Drive API, creates a readable stream from the new file in GCS, and streams it to the target Google Drive folder.

After a successful transfer to Drive, the function deletes the temporary file from the GCS bucket to clean up.

Setup and Deployment
To replicate this project, follow these steps:

1. Google Cloud Project Setup
Create a new Google Cloud Project.

Enable the following APIs: Cloud Storage API, Google Drive API, Eventarc API, Cloud Run API, Cloud Build API.

2. Create GCS Bucket and Configure CORS
Create a new GCS bucket (e.g., my-upload-bucket).

Set CORS Policy: This is crucial for browser uploads. Use the Cloud Shell (gcloud / gsutil) to apply the following policy:

# Create cors-config.json
echo '[
  {
    "origin": ["*"],
    "method": ["GET", "POST", "PUT"],
    "responseHeader": ["Content-Type", "Content-Range", "X-Goog-Resumable"],
    "maxAgeSeconds": 3600
  }
]' > cors-config.json

# Apply to your bucket
gsutil cors set cors-config.json gs://YOUR_BUCKET_NAME

(Note: For production, replace "*" with your website's domain).

3. Grant Public Upload Permission
In the GCS bucket permissions, disable "Public access prevention."

Grant the principal allUsers the IAM role of Storage Object Creator. This allows anonymous uploads but prevents reading or listing files.

4. Create Service Accounts & Permissions
This is the most critical step for security.

Create a primary service account for your function (e.g., gcs-to-drive-agent@...).

Grant permissions to several service accounts:

To your primary service account (gcs-to-drive-agent@...), grant the roles:

Storage Object Admin (to read and delete the staged file)

Eventarc Event Receiver (to receive the trigger event)

To the Eventarc Service Account (service-PROJECT_NUMBER@gcp-sa-eventarc.iam.gserviceaccount.com), grant the role:

Viewer (to validate project resources)

To the Cloud Storage Service Account (service-PROJECT_NUMBER@gs-project-accounts.iam.gserviceaccount.com), grant the role:

Pub/Sub Publisher (to publish GCS events)

5. Deploy the Cloud Run Function
Place the index.js and package.json files in a directory.

Update index.js with your target Google Drive Folder ID.

From the directory in Cloud Shell, run the deployment command:

gcloud functions deploy YOUR_FUNCTION_NAME \
  --gen2 \
  --runtime=nodejs20 \
  --entry-point=transferToDrive \
  --region=us-central1 \
  --source=. \
  --service-account=YOUR_PRIMARY_SERVICE_ACCOUNT_EMAIL \
  --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
  --trigger-event-filters="bucket=YOUR_BUCKET_NAME"

6. Configure the Frontend
Open the upload.js file.

Replace the BUCKET_NAME placeholder with your actual GCS bucket name.

Usage
Serve the index.html file using a local web server (like the VS Code Live Server extension) to avoid file:// protocol restrictions.

Open the webpage in your browser.

Select a file and click "Upload File."

Monitor the status on the page and check your target Google Drive folder for the result.
