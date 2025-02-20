# Enterprise API Documentation

This document describes the ReRoom Enterprise API, which allows enterprise users to submit rendering requests, check their status, and retrieve the results. The base URL for all API endpoints is `https://reroom.ai`.

## Overview

The Enterprise API provides endpoints for:

- Uploading images for processing.
- Submitting rendering requests with specific styles and creativity levels.
- Checking the status of rendering requests.
- Retrieving user account information (API key status).

This API is designed for enterprise users who have been granted a unique API key. All requests _require_ authentication using this key.

## Authentication

All API requests require an `Authorization` header with a bearer token:

```
Authorization: Bearer <your_enterprise_api_key>
```

Replace `<your_enterprise_api_key>` with your actual API key. If the header is missing, malformed, or the key is invalid, the API will return a `401 Unauthorized` error.

## Endpoints

### 1. Upload Image

- **Endpoint:** `POST /api/enterprise/upload`
- **Description:** Uploads an image to be used in a rendering request. This endpoint _does not_ initiate the rendering process; it only prepares the image for a subsequent render request.
- **Headers:**
  - `Content-Type`: `image/jpeg`, `image/jpg`, or `image/png` (case-insensitive). **This header is required.**
- **Request Body:** Raw image file data (binary). Do not send base64 encoded data.
- **Response (Success - 200 OK):**

  ```json
  {
    "success": true,
    "fileName": "<uuid>.<ext>"
  }
  ```

  - `fileName`: The unique filename assigned to the uploaded image. This filename should be used in the `/api/enterprise/render` endpoint. The `<uuid>` is a version 4 UUID, and `<ext>` is the file extension (jpg, jpeg, or png).

- **Response (Error):**

  - **400 Bad Request:**
    - Missing `Content-Type` header.
    - Invalid `Content-Type`.
    - Empty file.
    - File size exceeds 15MB.
    ```json
    { "err": "<error_message>" }
    ```
  - **500 Internal Server Error:**
    ```json
    { "err": "<error_message>" }
    ```
    - Failed to generate a presigned URL.
    - Network error or failure to upload to S3.
    - Other internal errors.

- **Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/upload' \
  -H 'Content-Type: image/jpeg' \
  --data-binary '@path/to/your/image.jpg'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/upload"
headers = {
    "Content-Type": "image/jpeg"
}

with open("path/to/your/image.jpg", "rb") as f:
    response = requests.post(url, headers=headers, data=f)

if response.status_code == 200:
    data = response.json()
    file_name = data["fileName"]
    print(f"Upload successful. File name: {file_name}")
else:
    print(f"Upload failed: {response.text}")
```

### 2. Submit Render Request

- **Endpoint:** `POST /api/enterprise/render`
- **Description:** Submits a rendering request. This endpoint uses the `fileName` returned by the `/api/enterprise/upload` endpoint.
- **Headers:**
  - `Authorization`: `Bearer <your_enterprise_api_key>`
  - `Content-Type`: `application/json`
- **Request Body:**

  ```json
  {
    "file_name": "<uuid>.<ext>",
    "style": "modern" | "nordic" | "japandi" | "new-chinese" | "european" | "american",
    "creativity": number
  }
  ```

  - `file_name`: (Required) The `fileName` returned by the `/api/enterprise/upload` endpoint. Must be a valid UUID v4 with a file extension.
  - `style`: (Required) The desired rendering style. Must be one of the allowed values.
  - `creativity`: (Required) A number between 0.5 and 1 (inclusive) representing the creativity level.

- **Response (Success - 200 OK):**

  ```json
  {
    "success": true,
    "data": {
      "render_id": "<render_id>"
    }
  }
  ```

  - `render_id`: The unique ID of the created rendering request. This ID is used to check the status of the render.

- **Response (Error):**

  - **401 Unauthorized:** Invalid or missing API key.
  - **400 Bad Request:**
    - Invalid request data (e.g., missing fields, invalid `file_name`, invalid `style`, or `creativity` out of range).
    - Invalid file name provided.
  - **500 Internal Server Error:**
    ```json
    { "err": "<error_message>" }
    ```
    - Failed to upload image.

- **Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "style": "modern",
    "creativity": 1
  }'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/render"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>",
    "style": "modern",
    "creativity": 1
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

### 3. Check Render Status

- **Endpoint:** `GET /api/enterprise/render/<render_id>`
- **Description:** Checks the status of a rendering request.
- **Headers:**
  - `Authorization`: `Bearer <your_enterprise_api_key>`
- **Parameters:**
  - `render_id`: (Required) The ID of the rendering request (obtained from the response of `POST /api/enterprise/render`).
- **Response (Success - 200 OK):**

  - **Status: `queued`** (Rendering has not started yet):

    ```json
    {
      "_id": "<render_id>",
      "uid": "<user_id>",
      "source": "<s3_url>",
      "style": "modern",
      "creativity": 1,
      "room_id": "<room_id>",
      "status": "queued",
      "outputs": [],
      "created_at": "<date>",
      "updated_at": "<date>"
    }
    ```

  - **Status: `rendering`** (Rendering is in progress):

    ```json
    {
      "_id": "<render_id>",
      "uid": "<user_id>",
      "source": "<s3_url>",
      "style": "modern",
      "creativity": 1,
      "room_id": "<room_id>",
      "status": "rendering",
      "outputs": [],
      "created_at": "<date>",
      "updated_at": "<date>"
    }
    ```

  - **Status: `completed`** (Rendering is complete):

    ```json
    {
      "_id": "<render_id>",
      "uid": "<user_id>",
      "source": "<s3_url>",
      "style": "modern",
      "creativity": 1,
      "room_id": "<room_id>",
      "status": "completed",
      "outputs": [
        "<s3_url_1>",
        "<s3_url_2>",
        ...
      ],
      "created_at": "<date>",
      "updated_at": "<date>"
    }
    ```

    The `outputs` array contains URLs to the rendered images on S3.

  - **Status: `failed`**
    ```json
    {
      "_id": "<render_id>",
      "uid": "<user_id>",
      "source": "<s3_url>",
      "style": "modern",
      "creativity": 1,
      "room_id": "<room_id>",
      "status": "failed",
      "outputs": [],
      "created_at": "<date>",
      "updated_at": "<date>"
    }
    ```

- **Response (Error):**

  - **401 Unauthorized:** Invalid or missing API key.
  - **404 Not Found:** `room_enterprise` not found.
  - **404 Not Found:** `room` not found.

- **Example:**

```bash
curl -X GET \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  'https://reroom.ai/api/enterprise/render/<your_render_id>'
```

```python
import requests

url = f"https://reroom.ai/api/enterprise/render/<your_render_id>"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>"
}

response = requests.get(url, headers=headers)

if response.status_code == 200:
    data = response.json()
    status = data["status"]
    print(f"Render status: {status}")
    if status == "completed":
        print("Output URLs:", data["outputs"])
else:
    print(f"Status check failed: {response.text}")
```
