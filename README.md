# Enterprise API Documentation

This document describes the ReRoom Enterprise API, which allows enterprise users to submit rendering requests, check their status, and retrieve the results. The base URL for all API endpoints is `https://reroom.ai`.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Rate Limits](#rate-limits)
- [Endpoints](#endpoints)
  - [Upload Image](#1-upload-image)
  - [Submit Exterior Render Request](#2-submit-exterior-render-request)
  - [Submit Interior Render Request](#3-submit-interior-render-request)
  - [Check Render Status](#4-check-render-status)
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

**Endpoint:** `POST /api/enterprise/upload`

**Description:**  
Uploads an image to be used in a rendering request. This endpoint _does not_ initiate the rendering process; it only prepares the image for a subsequent render request.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `image/jpeg`, `image/jpg`, or `image/png` (case-insensitive) **Required**

**Request Body:**  
Raw image file data (binary). Do not send base64 encoded data.

**Image Requirements:**

- **Supported Formats:**
  - JPEG/JPG (recommended)
  - PNG
- **Resolution:**
  - Minimum: 512x512 pixels
  - Maximum: 4096x4096 pixels
- **Aspect Ratio:** Between 4:3 and 16:9
- **Maximum File Size:** 15MB

**Response:**

- **Success (200 OK):**

  ```json
  {
    "success": true,
    "fileName": "<uuid>.<ext>"
  }
  ```

  - `fileName`: The unique filename assigned to the uploaded image. This filename should be used in the `/api/enterprise/render` endpoint. The `<uuid>` is a version 4 UUID, and `<ext>` is the file extension (jpg, jpeg, or png).

- **Error:**
  - See [Error Handling](#error-handling) section

**Examples:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/upload' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: image/jpeg' \
  --data-binary '@path/to/your/image.jpg'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/upload"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
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

### 2. Submit Exterior Render Request

**Endpoint:** `POST /api/enterprise/render`

**Description:**  
Submits an exterior rendering request using a previously uploaded image.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `application/json` **Required**

**Request Parameters:**

| Parameter          | Type   | Required | Description                                  | Constraints                                          |
| ------------------ | ------ | -------- | -------------------------------------------- | ---------------------------------------------------- |
| `file_name`        | string | Yes      | The filename returned by the upload endpoint | Must be a valid UUID+extension format                |
| `type`             | string | Yes      | The building/structure type                  | Must be one of the supported types (see table below) |
| `style`            | string | Yes      | The desired rendering style                  | Must be one of the supported styles                  |
| `daylight`         | string | No       | Time of day for lighting                     | Must be one of the supported daylight options        |
| `sky`              | string | No       | Sky condition                                | Must be one of the supported sky styles              |
| `season`           | string | No       | Season to be depicted                        | Must be one of the supported seasons                 |
| `landscape`        | string | No       | Surrounding environment                      | Must be one of the supported landscapes              |
| `material`         | string | No       | Primary building material                    | Must be one of the supported materials               |
| `prompt`           | string | No       | Additional text instructions                 | Free-form text                                       |
| `creativity`       | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"                 |
| `creativity_level` | number | No       | Fine-tuned variation control                 | Range: 0.5 (conservative) to 1.0 (creative)          |

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "type": "single-family-home",
  "style": "modern",
  "daylight": "sunset",
  "sky": "clear",
  "season": "summer",
  "landscape": "countryside",
  "material": "wood",
  "prompt": "With a large porch and circular driveway",
  "creativity": "balanced",
  "creativity_level": 0.75
}
```

**Available Options:**

<details>
<summary>Building Types</summary>

| Type ID               | Description               |
| --------------------- | ------------------------- |
| `no-type`             | No specific building type |
| `single-family-home`  | Single Family Home        |
| `townhouse`           | Townhouse                 |
| `condominium`         | Condominium               |
| `apartment-complex`   | Apartment Complex         |
| `office-building`     | Office Building           |
| `bank`                | Bank                      |
| `shopping-mall`       | Shopping Mall             |
| `hotel`               | Hotel                     |
| `retail-store`        | Retail Store              |
| `restaurant`          | Restaurant                |
| `school`              | School                    |
| `university`          | University                |
| `hospital`            | Hospital                  |
| `library`             | Library                   |
| `police-station`      | Police Station            |
| `courthouse`          | Courthouse                |
| `sports-arena`        | Sports Arena              |
| `concert-hall`        | Concert Hall              |
| `theater`             | Theater                   |
| `museum`              | Museum                    |
| `airport-terminal`    | Airport Terminal          |
| `bus-station`         | Bus Station               |
| `subway-station`      | Subway Station            |
| `train-station`       | Train Station             |
| `swimming-pool`       | Swimming Pool             |
| `resort`              | Resort                    |
| `greenhouse`          | Greenhouse                |
| `distribution-center` | Distribution Center       |

</details>

<details>
<summary>Styles</summary>

| Style ID                 | Description                            |
| ------------------------ | -------------------------------------- |
| `no-style`               | No style                               |
| `modern`                 | Contemporary minimalist design         |
| `nordic`                 | Scandinavian-inspired aesthetic        |
| `japandi`                | Japanese-Scandinavian fusion           |
| `new-chinese`            | Modern Chinese design                  |
| `european`               | Classic European elegance              |
| `american`               | Contemporary American style            |
| `minimalist-haven`       | Minimalist and serene design           |
| `modern-fusion`          | Sleek urban contemporary design        |
| `contemporary-elegance`  | Clean modern elegant design            |
| `industrial-loft`        | Urban warehouse-inspired design        |
| `bohemian-oasis`         | Eclectic bohemian design               |
| `coastal-breeze`         | Airy beach-inspired design             |
| `desert-retreat`         | Warm southwestern design               |
| `mountain-escape`        | Rustic alpine design                   |
| `victorian-elegance`     | Ornate classical design                |
| `art-deco-glamour`       | Geometric luxurious design             |
| `mid-century-modern`     | Retro functional design                |
| `french-country-charm`   | Rustic provincial design               |
| `colonial-classic`       | Traditional stately design             |
| `scandinavian-sanctuary` | Clean Nordic design                    |
| `japanese-zen`           | Minimalist traditional Japanese design |
| `moroccan-mystique`      | Exotic Middle Eastern design           |
| `mediterranean-retreat`  | Sun-drenched coastal design            |
| `indian-exuberance`      | Vibrant traditional Indian design      |
| `travelers-trove`        | Eclectic worldly design                |
| `cyber-eclectic-fusion`  | Futuristic urban design                |
| `neon-noir`              | Dark high-tech design                  |
| `techno-wonderland`      | Vibrant futuristic design              |
| `retro-futurism`         | Space-age nostalgic design             |
| `digital-zen`            | High-tech minimalist design            |
| `rustic`                 | Traditional rustic design              |
| `vintage`                | Classic vintage design                 |
| `shabby-chic`            | Elegant distressed design              |

</details>

<details>
<summary>Daylight Options</summary>

| Option    | Description                       |
| --------- | --------------------------------- |
| `midday`  | Bright, direct overhead sunlight  |
| `night`   | Darkness with artificial lighting |
| `sunset`  | Warm, golden evening light        |
| `sunrise` | Cool, soft morning light          |

</details>

<details>
<summary>Sky Styles</summary>

| Option   | Description                  |
| -------- | ---------------------------- |
| `clear`  | Cloudless blue sky           |
| `cloudy` | Sky with cloud coverage      |
| `rainy`  | Overcast with rain           |
| `snowy`  | Winter sky with snowfall     |
| `misty`  | Foggy atmospheric conditions |

</details>

<details>
<summary>Seasons</summary>

| Option   | Description                      |
| -------- | -------------------------------- |
| `spring` | Blooming flowers, fresh greenery |
| `summer` | Vibrant greens, full foliage     |
| `autumn` | Fall colors, orange/red foliage  |
| `winter` | Snow-covered, bare trees         |

</details>

<details>
<summary>Landscapes</summary>

| Option        | Description                            |
| ------------- | -------------------------------------- |
| `cityscape`   | Urban environment with buildings       |
| `countryside` | Rural setting with fields              |
| `forest`      | Wooded natural environment             |
| `desert`      | Arid landscape with minimal vegetation |
| `mountain`    | Elevated terrain with peaks            |
| `coastal`     | Oceanside environment                  |
| `tropical`    | Lush, exotic vegetation                |
| `grassland`   | Open field with grasses                |
| `tundra`      | Cold, treeless arctic landscape        |
| `wetland`     | Marshy area with water features        |

</details>

<details>
<summary>Materials</summary>

| Option     | Description                    |
| ---------- | ------------------------------ |
| `concrete` | Modern cement-based material   |
| `steel`    | Metal structural elements      |
| `glass`    | Transparent panels and windows |
| `wood`     | Natural timber elements        |
| `stone`    | Natural rock material          |
| `brick`    | Clay building blocks           |
| `bamboo`   | Sustainable reed material      |
| `clay`     | Earth-based material           |
| `gypsum`   | Mineral-based wall material    |
| `plastic`  | Synthetic polymer material     |

</details>

<details>
<summary>Color Schemes</summary>

| Option                   | Description                            |
| ------------------------ | -------------------------------------- |
| `warm-earth-tones`       | Browns, terracottas, and warm neutrals |
| `historical-romance`     | Muted, vintage-inspired colors         |
| `laid-back-blues`        | Calming blue palette                   |
| `palm-springs-modern`    | Bright retro-inspired colors           |
| `sweet-pastels`          | Soft, light color palette              |
| `rich-jewel-tones`       | Deep, saturated colors                 |
| `desert-chic`            | Warm, sand-inspired neutrals           |
| `forest-inspired`        | Natural greens and browns              |
| `high-contrast-neutrals` | Black, white, and gray combinations    |
| `airy-neutrals`          | Light, barely-there neutrals           |
| `coastal-neutrals`       | Beach-inspired light colors            |
| `ecletic-boho`           | Mix of bright, diverse colors          |

</details>

<details>
<summary>Creativity Levels</summary>

| Option     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `precise`  | Stays very close to the original image                |
| `balanced` | Moderate changes while maintaining original structure |
| `creative` | More significant artistic interpretation              |

</details>

<p>

**Response:**

- **Success (200 OK):**

  ```json
  {
    "success": true,
    "data": {
      "render_id": "<render_id>"
    }
  }
  ```

  - `render_id`: The unique ID of the rendering request. Use this ID to check render status.

- **Error Responses:**
  - **401 Unauthorized:** Invalid or missing API key
  - **400 Bad Request:**
    - Missing required fields
    - Invalid parameter values
    - Unsupported option values
  - **500 Internal Server Error:** Server-side processing failure

**Examples:**

```bash
curl -X POST \
  'https://rerender.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "type": "single-family-home",
    "style": "modern",
    "daylight": "sunset",
    "creativity": "balanced"
  }'
```

```python
import requests

url = "https://rerender.ai/api/enterprise/render/"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>",
    "type": "single-family-home",
    "style": "modern",
    "daylight": "sunset",
    "creativity": "balanced"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

### 3. Submit Interior Render Request

**Endpoint:** `POST /api/enterprise/render`

**Description:**  
Submits an interior rendering request using a previously uploaded image.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `application/json` **Required**

**Request Parameters:**

| Parameter          | Type   | Required | Description                                  | Constraints                                   |
| ------------------ | ------ | -------- | -------------------------------------------- | --------------------------------------------- |
| `file_name`        | string | Yes      | The filename returned by the upload endpoint | Must be a valid UUID+extension format         |
| `type`             | string | Yes      | The interior space type                      | Must be one of the supported interior types   |
| `style`            | string | Yes      | The desired rendering style                  | Must be one of the supported styles           |
| `daylight`         | string | No       | Time of day for lighting                     | Must be one of the supported daylight options |
| `season`           | string | No       | Season to be depicted                        | Must be one of the supported seasons          |
| `color`            | string | No       | Color palette                                | Must be one of the supported color schemes    |
| `prompt`           | string | No       | Additional text instructions                 | Free-form text                                |
| `creativity`       | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"          |
| `creativity_level` | number | No       | Fine-tuned variation control                 | Range: 0.5 (conservative) to 1.0 (creative)   |

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "type": "living-room",
  "style": "modern",
  "daylight": "midday",
  "season": "summer",
  "color": "warm-earth-tones",
  "prompt": "With large windows and modern artwork",
  "creativity": "balanced",
  "creativity_level": 0.75
}
```

**Available Options:**

<details>
<summary>Interior Types</summary>

| Type ID             | Description               |
| ------------------- | ------------------------- |
| `no-type`           | No specific interior type |
| `living-room`       | Living Room               |
| `kitchen`           | Kitchen                   |
| `bedroom`           | Bedroom                   |
| `bathroom`          | Bathroom                  |
| `attic`             | Attic                     |
| `dining-room`       | Dining Room               |
| `study-room`        | Study Room                |
| `home-office`       | Home Office               |
| `gaming-room`       | Gaming Room               |
| `house-exterior`    | House Exterior            |
| `outdoor-pool-area` | Outdoor Pool Area         |
| `outdoor-patio`     | Outdoor Patio             |
| `outdoor-garden`    | Outdoor Garden            |
| `meeting-room`      | Meeting Room              |
| `workshop`          | Workshop                  |
| `fitness-gym`       | Fitness Gym               |
| `coffee-shop`       | Coffee Shop               |
| `clothing-store`    | Clothing Store            |
| `walk-in-closet`    | Walk-in Closet            |
| `toilet`            | Toilet                    |
| `restaurant`        | Restaurant                |
| `office`            | Office                    |
| `coworking-space`   | Coworking Space           |
| `hotel-lobby`       | Hotel Lobby               |
| `hotel-room`        | Hotel Room                |
| `hotel-bathroom`    | Hotel Bathroom            |
| `exhibition-space`  | Exhibition Space          |
| `onsen`             | Onsen                     |
| `mudroom`           | Mudroom                   |

</details>

<details>
<summary>Styles</summary>

| Style ID                 | Description                            |
| ------------------------ | -------------------------------------- |
| `no-style`               | No style                               |
| `modern`                 | Contemporary minimalist design         |
| `nordic`                 | Scandinavian-inspired aesthetic        |
| `japandi`                | Japanese-Scandinavian fusion           |
| `new-chinese`            | Modern Chinese design                  |
| `european`               | Classic European elegance              |
| `american`               | Contemporary American style            |
| `minimalist-haven`       | Minimalist and serene design           |
| `modern-fusion`          | Sleek urban contemporary design        |
| `contemporary-elegance`  | Clean modern elegant design            |
| `industrial-loft`        | Urban warehouse-inspired design        |
| `bohemian-oasis`         | Eclectic bohemian design               |
| `coastal-breeze`         | Airy beach-inspired design             |
| `desert-retreat`         | Warm southwestern design               |
| `mountain-escape`        | Rustic alpine design                   |
| `victorian-elegance`     | Ornate classical design                |
| `art-deco-glamour`       | Geometric luxurious design             |
| `mid-century-modern`     | Retro functional design                |
| `french-country-charm`   | Rustic provincial design               |
| `colonial-classic`       | Traditional stately design             |
| `scandinavian-sanctuary` | Clean Nordic design                    |
| `japanese-zen`           | Minimalist traditional Japanese design |
| `moroccan-mystique`      | Exotic Middle Eastern design           |
| `mediterranean-retreat`  | Sun-drenched coastal design            |
| `indian-exuberance`      | Vibrant traditional Indian design      |
| `travelers-trove`        | Eclectic worldly design                |
| `cyber-eclectic-fusion`  | Futuristic urban design                |
| `neon-noir`              | Dark high-tech design                  |
| `techno-wonderland`      | Vibrant futuristic design              |
| `retro-futurism`         | Space-age nostalgic design             |
| `digital-zen`            | High-tech minimalist design            |
| `rustic`                 | Traditional rustic design              |
| `vintage`                | Classic vintage design                 |
| `shabby-chic`            | Elegant distressed design              |

</details>

<details>
<summary>Daylight Options</summary>

| Option    | Description                       |
| --------- | --------------------------------- |
| `midday`  | Bright, direct overhead sunlight  |
| `night`   | Darkness with artificial lighting |
| `sunset`  | Warm, golden evening light        |
| `sunrise` | Cool, soft morning light          |

</details>

<details>
<summary>Seasons</summary>

| Option   | Description                      |
| -------- | -------------------------------- |
| `spring` | Blooming flowers, fresh greenery |
| `summer` | Vibrant greens, full foliage     |
| `autumn` | Fall colors, orange/red foliage  |
| `winter` | Snow-covered, bare trees         |

</details>

<details>
<summary>Color Schemes</summary>

| Option                   | Description                            |
| ------------------------ | -------------------------------------- |
| `warm-earth-tones`       | Browns, terracottas, and warm neutrals |
| `historical-romance`     | Muted, vintage-inspired colors         |
| `laid-back-blues`        | Calming blue palette                   |
| `palm-springs-modern`    | Bright retro-inspired colors           |
| `sweet-pastels`          | Soft, light color palette              |
| `rich-jewel-tones`       | Deep, saturated colors                 |
| `desert-chic`            | Warm, sand-inspired neutrals           |
| `forest-inspired`        | Natural greens and browns              |
| `high-contrast-neutrals` | Black, white, and gray combinations    |
| `airy-neutrals`          | Light, barely-there neutrals           |
| `coastal-neutrals`       | Beach-inspired light colors            |
| `ecletic-boho`           | Mix of bright, diverse colors          |

</details>

<details>
<summary>Creativity Levels</summary>

| Option     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `precise`  | Stays very close to the original image                |
| `balanced` | Moderate changes while maintaining original structure |
| `creative` | More significant artistic interpretation              |

</details>

<p>

**Response:**

- **Success (200 OK):**

  ```json
  {
    "success": true,
    "data": {
      "render_id": "<render_id>"
    }
  }
  ```

  - `render_id`: The unique ID of the rendering request. Use this ID to check render status.

- **Error Responses:**
  - **401 Unauthorized:** Invalid or missing API key
  - **400 Bad Request:**
    - Missing required fields
    - Invalid parameter values
    - Unsupported option values
  - **500 Internal Server Error:** Server-side processing failure

**Examples:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "type": "living-room",
    "style": "modern",
    "daylight": "midday",
    "creativity": "balanced"
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
    "type": "living-room",
    "style": "modern",
    "daylight": "midday",
    "creativity": "balanced"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

### 4. Check Render Status

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
