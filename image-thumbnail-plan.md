You are an expert backend engineer and DevOps specialist tasked with implementing an automatic, asynchronous image thumbnail generation pipeline for an existing Python/Django application running on Google Cloud Platform (GCP).

### Context & Existing Architecture
- Users can upload images, which are saved to a Django `Image` model (IDs prefixed with "IMG_").
- The upload process saves raw, full-resolution images directly to a specific GCS bucket (`original-images-bucket`).
- A separate GCS bucket for thumbnails (`thumbnail-images-bucket`) has already been provisioned via Terraform, but is currently empty.
- The `Image` model currently holds the `storage_url` for the original image.

### Tasks

1. **GCP Background Infrastructure (Terraform)**
   - Update the existing Terraform configuration to wire up the asynchronous event pipeline.
   - Configure a GCS Event Driven trigger or a Cloud Pub/Sub topic that fires a notification whenever a new object is successfully finalized/created in the `original-images-bucket`.
   - Define a GCP Cloud Function (or Cloud Run service, depending on codebase preference) that subscribes to these upload events to act as our background worker.

2. **Thumbnail Generation Worker Logic (Python)**
   - Write the background worker script (optimized for the Cloud Function/Cloud Run environment).
   - The worker must:
     - Listen for the GCS upload event and extract the file path/name.
     - Download the original image into memory/temp storage.
     - Process the image to create a thumbnail version (e.g., maximum bounding box of 300x300 pixels, maintaining aspect ratio) using a lightweight library like `Pillow`.
     - Upload the generated thumbnail to the designated `thumbnail-images-bucket` using a predictable naming convention (e.g., matching the original `IMG_` ID or file name).

3. **Django Application Integration**
   - Update the Django `Image` model to include a new field: `thumbnail_url` (CharField or URLField, nullable).
   - Implement an internal API endpoint or a secure webhook that the background worker can call once thumbnail processing is complete. This endpoint will update the corresponding `Image` record with its new `thumbnail_url`.
   - Ensure this update operation is safe, handles potential race conditions gracefully, and logs successes or failures accurately.