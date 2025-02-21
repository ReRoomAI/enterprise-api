# Enterprise API Documentation

This document describes the ReRoom Enterprise API, which allows enterprise users to submit rendering requests, check their status, and retrieve the results. The base URL for all API endpoints is `https://reroom.ai`.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Rate Limits](#rate-limits)
- [Endpoints](#endpoints)
  - [Upload Image](#1-upload-image)
  - [Submit Render Request](#2-submit-render-request)
  - [Check Render Status](#3-check-render-status)
- [Error Handling](#error-handling)

## Overview

The Enterprise API provides endpoints for:

- Uploading images for processing (supports JPEG and PNG formats)
- Submitting rendering requests with specific styles and creativity levels
- Checking the status of rendering requests
- Retrieving user account information (API key status)

This API is designed for enterprise users who have been granted a unique API key. All requests _require_ authentication using this key.

## Authentication

All API requests require an `Authorization` header with a bearer token:

```
Authorization: Bearer <your_enterprise_api_key>
```

Replace `<your_enterprise_api_key>` with your actual API key. If the header is missing, malformed, or the key is invalid, the API will return a `401 Unauthorized` error.

## Rate Limits

- Maximum file size: 15MB per upload
- Maximum concurrent renders: 10 per account
- Request rate: 100 requests per minute

## Endpoints

### 1. Upload Image

- **Endpoint:** `POST /api/enterprise/upload`
- **Description:** Uploads an image to be used in a rendering request. This endpoint _does not_ initiate the rendering process; it only prepares the image for a subsequent render request.
- **Headers:**
  - `Content-Type`: `image/jpeg`, `image/jpg`, or `image/png` (case-insensitive). **Required**
- **Request Body:** Raw image file data (binary). Do not send base64 encoded data.
- **Supported Image Formats:**
  - JPEG/JPG (recommended)
  - PNG
- **Image Requirements:**

  - Minimum resolution: 512x512 pixels
  - Maximum resolution: 4096x4096 pixels
  - Aspect ratio: Between 4:3 and 16:9

- **Response (Success - 200 OK):**

  ```json
  {
    "success": true,
    "fileName": "<uuid>.<ext>"
  }
  ```

  - `fileName`: The unique filename assigned to the uploaded image. This filename should be used in the `/api/enterprise/render` endpoint. The `<uuid>` is a version 4 UUID, and `<ext>` is the file extension (jpg, jpeg, or png).

- **Response (Error):**

  - See [Error Handling](#error-handling) section

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
- **Description:** Submits a rendering request using a previously uploaded image.
- **Headers:**
  - `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
  - `Content-Type`: `application/json` **Required**
- **Request Body:**

  ```json
  {
    "file_name": "<uuid>.<ext>",
    "style": "modern" | "nordic" | "new-chinese" | "european" ,
    "creativity": number
  }
  ```

  - `file_name`: (Required) The `fileName` returned by the upload endpoint
  - `style`: (Required) The desired rendering style
  - `creativity`: (Required) A number between 0.5 and 1
    - 0.5: More conservative, closer to original
    - 1.0: More creative, larger variations

- **Available Styles:**

  - `modern`: Contemporary minimalist design
  - `nordic`: Scandinavian-inspired aesthetic
  - `japandi`: Japanese-Scandinavian fusion
  - `new-chinese`: Modern Chinese design
  - `european`: Classic European elegance
  - `american`: Contemporary American style

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

## Error Handling

All error responses follow this format:

```json
{
  "err": "<error_message>"
}
```

Common error codes:

- **400 Bad Request**
  - Invalid request parameters
  - File size exceeds 15MB
  - Unsupported image format
  - Invalid image dimensions
- **401 Unauthorized**
  - Missing or invalid API key
- **404 Not Found**
  - Resource not found
- **429 Too Many Requests**
  - Rate limit exceeded
- **500 Internal Server Error**
  - Server-side processing error
