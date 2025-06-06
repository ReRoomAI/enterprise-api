# Enterprise API Documentation

This document describes the ReRoom Enterprise API, which allows enterprise users to submit rendering requests, check their status, and retrieve the results. The base URL for all API endpoints is `https://reroom.ai`.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Rate Limits](#rate-limits)
- [Endpoints](#endpoints)
  - [Upload Image](#1-upload-image)
  - [Submit Render Request](#2-submit-render-request)
    - [Interior Rendering](#21-interior-rendering-parameters)
    - [Exterior Rendering](#22-exterior-rendering-parameters)
    - [Sketch](#23-sketch)
    - [Empty Room](#24-empty-room)
    - [Virtual Staging](#25-virtual-staging)
    - [Edit Image](#26-edit-image)
    - [Shared Field Options](#27-shared-field-options)
      - [Interior Options](#271-interior-options)
      - [Exterior Options](#272-exterior-options)
      - [Shared Options](#273-shared-options)
    - [Upscale Image](#28-upscale-image)
  - [Response for All Render Requests](#3-response-for-all-render-requests)
  - [Check Render Status](#4-check-render-status)
  - [Get Credit Status](#5-get-credit-status)

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

- **Error Response:**

  ```json
  { "err": "<error_message>" }
  ```

  | Status Code | Possible Error Messages                                                               |
  | ----------- | ------------------------------------------------------------------------------------- |
  | 400         | "Invalid file format", "JPEG and PNG files only", "Empty file", "Max file size: 15MB" |
  | 500         | "Upload failed. Try again.", "Connection error. Try again."                           |

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

### 2. Submit Render Request

**Endpoint:** `POST /api/enterprise/render`

**Description:**  
Submits a rendering request using either a previously uploaded image (via the upload endpoint) or an external image URL. The endpoint uses a query parameter `field` to differentiate between interior and exterior rendering requests.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `application/json` **Required**

**Query Parameters:**

| Parameter | Type   | Required | Description                         | Default    | Values                 |
| --------- | ------ | -------- | ----------------------------------- | ---------- | ---------------------- |
| `field`   | string | No       | Type of rendering request to submit | `interior` | `interior`, `exterior` |

**Request Body:**  
The request body parameters depend on the value of the `field` query parameter.

#### 2.1 Interior Rendering Parameters

Use these parameters when specifying `field=interior` or omitting the `field` parameter (as `interior` is the default).

**Required Parameters:**

| Parameter    | Type   | Required | Description                                  | Constraints                           |
| ------------ | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name`  | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`   | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                   |
| `creativity` | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"  |

\*Either `file_name` or `file_url` must be provided.

**Optional Parameters:**

| Parameter                | Type   | Required | Description                  | Constraints                                                                                               |
| ------------------------ | ------ | -------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `type`                   | string | No       | The interior space type      | Must be one of the [supported interior types](#interior-type). Defaults to "no-type" if not specified     |
| `style`                  | string | No       | The desired rendering style  | Must be one of the [supported interior styles](#interior-style). Defaults to "no-style" if not specified  |
| `daylight`               | string | No       | Time of day for lighting     | Must be one of the [supported daylight options](#daylight)                                                |
| `season`                 | string | No       | Season to be depicted        | Must be one of the [supported seasons](#season)                                                           |
| `color`                  | string | No       | Color palette                | Must be one of the [supported color schemes](#interior-color)                                             |
| `architectural_material` | string | No       | Primary building material    | Must be one of the [supported architectural materials](#architectural-materials)                          |
| `prompt`                 | string | No       | Additional text instructions | Free-form text                                                                                            |
| `creativity_level`       | number | No       | Fine-tuned variation control | Range: 0.5 (conservative) to 1.0 (creative). Defaults to 0.7. Only applies when `creativity` is "precise" |

#### 2.2 Exterior Rendering Parameters

Use these parameters when specifying `field=exterior` in the query string.

**Required Parameters:**

| Parameter    | Type   | Required | Description                                  | Constraints                           |
| ------------ | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name`  | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`   | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                   |
| `creativity` | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"  |

\*Either `file_name` or `file_url` must be provided.

**Optional Parameters:**

| Parameter                | Type   | Required | Description                  | Constraints                                                                                               |
| ------------------------ | ------ | -------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `type`                   | string | No       | The building/structure type  | Must be one of the [supported exterior types](#exterior-type). Defaults to "no-type" if not specified     |
| `style`                  | string | No       | The desired rendering style  | Must be one of the [supported exterior styles](#exterior-style). Defaults to "no-style" if not specified  |
| `daylight`               | string | No       | Time of day for lighting     | Must be one of the [supported daylight options](#daylight)                                                |
| `sky`                    | string | No       | Sky condition                | Must be one of the [supported sky styles](#exterior-sky)                                                  |
| `season`                 | string | No       | Season to be depicted        | Must be one of the [supported seasons](#season)                                                           |
| `landscape`              | string | No       | Surrounding environment      | Must be one of the [supported landscapes](#exterior-landscape)                                            |
| `architectural_material` | string | No       | Primary building material    | Must be one of the [supported architectural materials](#exterior-material)                                |
| `prompt`                 | string | No       | Additional text instructions | Free-form text                                                                                            |
| `creativity_level`       | number | No       | Fine-tuned variation control | Range: 0.5 (conservative) to 1.0 (creative). Defaults to 0.7. Only applies when `creativity` is "precise" |

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "creativity": "balanced",
  "type": "single-family-home",
  "style": "sleek-international",
  "daylight": "sunset"
}
```

**Exterior Rendering Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render?field=exterior' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "creativity": "balanced",
    "type": "single-family-home",
    "style": "sleek-international",
    "daylight": "sunset"
  }'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/render"
params = {
    "field": "exterior"
}
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>",
    "creativity": "balanced",
    "type": "single-family-home",
    "style": "sleek-international",
    "daylight": "sunset"
}

response = requests.post(url, headers=headers, params=params, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

#### 2.3 Sketch

Use these parameters when specifying `tool=sketch` in the request body. You can also use the `field` query parameter to specify "interior" or "exterior".

**Required Parameters:**

| Parameter   | Type   | Required | Description                                  | Constraints                           |
| ----------- | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name` | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`  | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                   |
| `tool`      | string | Yes      | The tool to use for rendering                | Must be "sketch"                      |

\*Either `file_name` or `file_url` must be provided.

**Optional Parameters:**

| Parameter | Type   | Required | Description                 | Constraints                                                                                                                           |
| --------- | ------ | -------- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `type`    | string | No       | The space or building type  | Must be one of the supported types based on the `field` value specified ([interior](#interior-type) or [exterior](#exterior-type))    |
| `style`   | string | No       | The desired rendering style | Must be one of the supported styles based on the `field` value specified ([interior](#interior-style) or [exterior](#exterior-style)) |
| `view`    | string | No       | The view type for sketch    | One of: "drawing", "floorplan", "section", "elevation"                                                                                |

**Query Parameters:**

| Parameter | Type   | Required | Description                         | Default    | Values                 |
| --------- | ------ | -------- | ----------------------------------- | ---------- | ---------------------- |
| `field`   | string | No       | Type of rendering request to submit | `interior` | `interior`, `exterior` |

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "sketch",
  "type": "living-room",
  "style": "modern-fusion",
  "view": "floorplan"
}
```

**Note:** The `type` and `style` parameters accept the same values as listed in the Interior or Exterior rendering parameters sections, depending on the `field` value.

**Sketch Rendering Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render?field=interior' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "tool": "sketch",
    "type": "living-room",
    "style": "minimalist-haven",
    "view": "floorplan"
  }'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/render"
params = {
    "field": "interior"
}
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>",
    "tool": "sketch",
    "type": "living-room",
    "style": "minimalist-haven",
    "view": "floorplan"
}

response = requests.post(url, headers=headers, params=params, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

#### 2.4 Empty Room

Use these parameters when specifying `tool=empty-room` in the request body.

**Required Parameters:**

| Parameter   | Type   | Required | Description                                  | Constraints                           |
| ----------- | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name` | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`  | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                   |
| `tool`      | string | Yes      | The tool to use for rendering                | Must be "empty-room"                  |

\*Either `file_name` or `file_url` must be provided.

**Description:**  
The empty-room tool removes all furniture, decorations, and personal items from an image, leaving only the architectural elements. This is useful for virtual staging or when you want to start with a clean slate before adding new furniture or decorations. Unlike other rendering tools, the empty-room tool processes images independently of the interior/exterior distinction and does not require or use the `field` parameter.

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "empty-room"
}
```

**Empty Room Rendering Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "tool": "empty-room"
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
    "tool": "empty-room"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Empty room request submitted. Render ID: {render_id}")
else:
    print(f"Empty room request failed: {response.text}")
```

#### 2.5 Virtual Staging

Use these parameters when specifying `tool=virtual-staging` in the request body.

**Required Parameters:**

| Parameter   | Type   | Required | Description                                  | Constraints                           |
| ----------- | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name` | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`  | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                   |
| `tool`      | string | Yes      | The tool to use for rendering                | Must be "virtual-staging"             |
| `action`    | string | Yes      | The staging approach to use                  | Must be "preset" or "reference"       |

\*Either `file_name` or `file_url` must be provided.

**Parameters for Preset Mode:**

When `action` is set to "preset", the following parameters are required:

| Parameter | Type   | Required | Description                          | Constraints                                                    |
| --------- | ------ | -------- | ------------------------------------ | -------------------------------------------------------------- |
| `type`    | string | Yes      | The type of room to stage            | Must be one of the [supported staging types](#staging-types)   |
| `style`   | string | Yes      | The design style for virtual staging | Must be one of the [supported staging styles](#staging-styles) |

**Parameters for Reference Mode:**

When `action` is set to "reference", the following parameters are required:

| Parameter             | Type     | Required | Description                                | Constraints              |
| --------------------- | -------- | -------- | ------------------------------------------ | ------------------------ |
| `reference_file_urls` | string[] | Yes      | URLs of reference images for staging style | Must be valid image URLs |

**Description:**  
The virtual-staging tool adds furniture and decorations to an empty or sparsely furnished room based on either preset design styles or reference images. This tool processes images immediately and returns a completed staged image without queuing, unlike other rendering tools. It's useful for real estate marketing, interior design visualization, or showing clients how a space could look when furnished.

**Request Body Example (Preset Mode):**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "virtual-staging",
  "action": "preset",
  "type": "living-room",
  "style": "modern"
}
```

**Request Body Example (Reference Mode):**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "virtual-staging",
  "action": "reference",
  "reference_file_urls": [
    "https://example.com/reference1.jpg",
    "https://example.com/reference2.jpg"
  ]
}
```

**Virtual Staging Example (Preset Mode):**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "tool": "virtual-staging",
    "action": "preset",
    "type": "living-room",
    "style": "modern"
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
    "tool": "virtual-staging",
    "action": "preset",
    "type": "living-room",
    "style": "modern"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Virtual staging request submitted. Render ID: {render_id}")
else:
    print(f"Virtual staging request failed: {response.text}")
```

<a id="staging-types"></a>

##### Staging Types

| Option         | Description  |
| -------------- | ------------ |
| `living-room`  | Living Room  |
| `dining-room`  | Dining Room  |
| `kitchen`      | Kitchen      |
| `bedroom`      | Bedroom      |
| `bathroom`     | Bathroom     |
| `office`       | Office       |
| `meeting-room` | Meeting Room |
| `restaurant`   | Restaurant   |

<a id="staging-styles"></a>

##### Staging Styles

| Option             | Description      |
| ------------------ | ---------------- |
| `modern`           | Modern           |
| `scandinavian`     | Scandinavian     |
| `mediterranean`    | Mediterranean    |
| `industrial`       | Industrial       |
| `american-vintage` | American Vintage |
| `neo-classical`    | Neo-Classical    |
| `luxury`           | Luxury           |
| `futuristic`       | Futuristic       |

#### 2.6 Edit Image

Use these parameters when specifying `tool=edit` in the request body.

**Required Parameters:**

| Parameter   | Type   | Required | Description                                  | Constraints                                              |
| ----------- | ------ | -------- | -------------------------------------------- | -------------------------------------------------------- |
| `file_name` | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format                    |
| `file_url`  | string | Yes\*    | The URL of the file to be rendered           | Must be a valid URL                                      |
| `tool`      | string | Yes      | The tool to use for rendering                | Must be "edit"                                           |
| `action`    | string | Yes      | The type of edit to perform                  | Must be one of: "object", "remove", "material", "prompt" |
| `mask`      | string | Yes      | Base64-encoded mask image data               | Valid base64 string representing the mask                |
| `field`     | string | No       | Type of space to edit                        | Must be "interior" or "exterior". Defaults to "interior" |

\*Either `file_name` or `file_url` must be provided.

**Action-Specific Parameters:**

Depending on the `action` value, you must provide additional parameters:

1. **For `action="material"`:**

| Parameter  | Type   | Required | Description                       | Constraints                                          |
| ---------- | ------ | -------- | --------------------------------- | ---------------------------------------------------- |
| `material` | string | Yes      | The material to apply to the mask | Must be one of the [Edit Materials](#edit-materials) |

2. **For `action="prompt"`:**

| Parameter | Type   | Required | Description                                        | Constraints    |
| --------- | ------ | -------- | -------------------------------------------------- | -------------- |
| `prompt`  | string | Yes      | Text instructions for what to add in the mask area | Free-form text |

3. **For `action="object"`:**

| Parameter | Type   | Required | Description                            | Constraints                                  |
| --------- | ------ | -------- | -------------------------------------- | -------------------------------------------- |
| `object`  | object | Yes      | The object to place in the masked area | Object with `category` and `name` properties |

The `object` parameter should have the following structure:

```json
{
  "category": "<category>",
  "name": "<name>"
}
```

Where:

- For interiors (`field="interior"`): `category` must be one of: "furniture", "decoration", "creature" (see [Edit Object Categories](#edit-object-categories) and [Edit Object Names (Interior)](#edit-object-names-interior)).
- For exteriors (`field="exterior"`): `category` must be one of: "street_landscape", "street_furniture", "street_creature" (see [Edit Object Categories](#edit-object-categories) and [Edit Object Names (Exterior)](#edit-object-names-exterior)).
- The `name` must be one of the valid options for the selected category.

- For `action="remove"`, no additional parameters are required. The masked area will be removed and filled in based on the surrounding context.

**Description:**  
The edit tool allows targeted modifications to specific areas of an image defined by a mask. You can change materials, add objects, remove elements, or apply custom prompts to the masked region. This is useful for modifying existing designs, testing different materials, or adding/removing specific elements.

**Request Body Example (Material Edit):**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "edit",
  "action": "material",
  "mask": "base64-encoded-mask-data",
  "material": "marble",
  "field": "interior"
}
```

**Request Body Example (Object Edit):**

```json
{
  "file_name": "<uuid>.<ext>",
  "tool": "edit",
  "action": "object",
  "mask": "base64-encoded-mask-data",
  "object": {
    "category": "furniture",
    "name": "sofa"
  },
  "field": "interior"
}
```

**Edit Example (Material):**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render?field=interior' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "tool": "edit",
    "action": "material",
    "mask": "<base64-encoded-mask-data>",
    "material": "marble",
    "field": "interior"
  }'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/render"
params = {
    "field": "interior"
}
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>",
    "tool": "edit",
    "action": "material",
    "mask": "<base64-encoded-mask-data>",
    "material": "marble",
    "field": "interior"
}

response = requests.post(url, headers=headers, params=params, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Edit request submitted. Render ID: {render_id}")
else:
    print(f"Edit request failed: {response.text}")
```

##### Edit Materials

`pine`, `oak`, `walnut`, `marble`, `granite`, `limestone`, `leather`, `fabrics`, `carpet`, `paint`, `plaster`, `wallpaper`, `brick`, `unglazed-tiles`, `glazed-tiles`, `mirror`, `clear-glass`, `frosted-glass`, `steel`, `bronze`, `aluminum`, `polished-concrete`, `exposed-concrete`, `stone-veneer`

##### Edit Object Categories

| Field    | Categories (use in `object.category`)                     |
| -------- | --------------------------------------------------------- |
| interior | `furniture`, `decoration`, `creature`                     |
| exterior | `street_landscape`, `street_furniture`, `street_creature` |

##### Edit Object Names (Interior)

| Category   | Names (use in `object.name`)                                                                    |
| ---------- | ----------------------------------------------------------------------------------------------- |
| furniture  | `sofa`, `table`, `chair`, `cabinet`, `shelves`, `kitchen-island`, `desk`, `bed`                 |
| decoration | `painting`, `wall-clock`, `book`, `bottle`, `fruit`, `potted-plant`, `vase-with-flower`, `lamp` |
| creature   | `child`, `standing-man`, `sitting-man`, `standing-woman`, `sitting-woman`, `dog`, `cat`         |

##### Edit Object Names (Exterior)

| Category         | Names (use in `object.name`)                                                                                                                                       |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| street_landscape | `tree`, `bush`, `potted-plant`, `flower`, `grass`, `bunch-of-rocks`, `pond`                                                                                        |
| street_furniture | `car`, `scooter`, `bicycle`, `a-urban-tree`, `a-urban-seating-unit`, `a-public-bench`, `a-picnic-table`, `a-street-lamp`, `a-public-sculpture`, `a-water-fountain` |
| street_creature  | `one-people`, `one-child`, `one-old-people`, `one-sitting-people-on-an-object`, `a-group-of-people`, `dog`, `cat`, `bird`                                          |

#### 2.7 Shared Field Options

This section organizes the available options for both interior and exterior rendering fields.

##### 2.7.1 Interior Options

<a id="interior-type"></a>

##### Interior Type Options

| Option              | Description               |
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

<a id="interior-style"></a>

##### Interior Style Options

| Option                   | Description            |
| ------------------------ | ---------------------- |
| `no-style`               | No Style               |
| `minimalist-haven`       | Minimalist Haven       |
| `modern-fusion`          | Modern Fusion          |
| `contemporary-elegance`  | Contemporary Elegance  |
| `industrial-loft`        | Industrial Loft        |
| `bohemian-oasis`         | Bohemian Oasis         |
| `coastal-breeze`         | Coastal Breeze         |
| `desert-retreat`         | Desert Retreat         |
| `mountain-escape`        | Mountain Escape        |
| `victorian-elegance`     | Victorian Elegance     |
| `art-deco-glamour`       | Art Deco Glamour       |
| `mid-century-modern`     | Mid-Century Modern     |
| `french-country-charm`   | French Country Charm   |
| `colonial-classic`       | Colonial Classic       |
| `scandinavian-sanctuary` | Scandinavian Sanctuary |
| `japanese-zen`           | Japanese Zen           |
| `moroccan-mystique`      | Moroccan Mystique      |
| `mediterranean-retreat`  | Mediterranean Retreat  |
| `indian-exuberance`      | Indian Exuberance      |
| `travelers-trove`        | Traveler's Trove       |
| `cyber-eclectic-fusion`  | Cyber Eclectic Fusion  |
| `neon-noir`              | Neon Noir              |
| `techno-wonderland`      | Techno Wonderland      |
| `retro-futurism`         | Retro Futurism         |
| `digital-zen`            | Digital Zen            |
| `rustic`                 | Rustic                 |
| `vintage`                | Vintage                |
| `shabby-chic`            | Shabby Chic            |

<a id="interior-color"></a>

##### Interior Color Options

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

##### 2.7.2 Exterior Options

<a id="exterior-type"></a>

##### Exterior Type Options

| Option                | Description               |
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

<a id="exterior-style"></a>

##### Exterior Style Options

| Option                       | Description                |
| ---------------------------- | -------------------------- |
| `no-style`                   | No Style                   |
| `sleek-international`        | Sleek International        |
| `eco-futurism`               | Eco Futurism               |
| `high-tech`                  | High Tech                  |
| `biomorphic`                 | Biomorphic                 |
| `neo-futurism`               | Neo Futurism               |
| `smart-city`                 | Smart City                 |
| `space-colony`               | Space Colony               |
| `glassy-minimalist`          | Minimalisme Verrier        |
| `dynamic-parametric`         | Dynamic Parametric         |
| `eco-friendly-sustainable`   | Eco-friendly Sustainable   |
| `expressive-high-tech`       | Expressive High-tech       |
| `warm-wooden-modern`         | Warm Wooden Modern         |
| `cantilevered-contemporary`  | Cantilevered Contemporary  |
| `warehouse-conversion`       | Warehouse Conversion       |
| `factory-loft`               | Factory Loft               |
| `steel-frame-vintage`        | Steel Frame Vintage        |
| `industrial-chic-townhouse`  | Industrial Chic Townhouse  |
| `repurposed-factory`         | Repurposed Factory         |
| `industrial-farmhouse`       | Industrial Farmhouse       |
| `enchanting-gothic`          | Enchanting Gothic          |
| `opulent-baroque`            | Opulent Baroque            |
| `earthy-vernacular`          | Earthy Vernacular          |
| `graceful-moorish`           | Graceful Moorish           |
| `whimsical-art-deco`         | Whimsical Art Deco         |
| `intricate-tudor`            | Intricate Tudor            |
| `bold-brutalist`             | Bold Brutalist             |
| `playful-post-modern`        | Playful Post-Modern        |
| `serene-scandinavian`        | Serene Scandinavian        |
| `curvaceous-organic`         | Curvaceous Organic         |
| `dramatic-deconstructivist`  | Dramatic Deconstructivist  |
| `romanesque-revival`         | Romanesque Revival         |
| `spanish-revival`            | Spanish Revival            |
| `colonial-revival`           | Colonial Revival           |
| `gothic-revival`             | Gothic Revival             |
| `art-nouveau-revival`        | Art Nouveau Revival        |
| `neogothic-revival`          | Neogothic Revival          |
| `adobe-pueblo`               | Adobe Pueblo               |
| `new-england-cottage`        | New England Cottage        |
| `tropical-bamboo`            | Tropical Bamboo            |
| `mediterranean-village`      | Mediterranean Village      |
| `african-mud-hut`            | African Mud Hut            |
| `scandinavian-log-cabin`     | Scandinavian Log Cabin     |
| `ancient-egyptian`           | Ancient Egyptian           |
| `greek-classical`            | Greek Classical            |
| `roman-imperial`             | Roman Imperial             |
| `pre-columbian-mesoamerican` | Pre-Columbian Mesoamerican |
| `ancient-chinese`            | Ancient Chinese            |
| `ancient-indian`             | Ancient Indian             |
| `islamic-moorish`            | Islamic Moorish            |
| `islamic-ottoman`            | Islamic Ottoman            |
| `islamic-persian`            | Islamic Persian            |
| `islamic-mughal`             | Islamic Mughal             |
| `islamic-fatimid`            | Islamic Fatimid            |
| `islamic-seljuk`             | Islamic Seljuk             |

<a id="exterior-sky"></a>

##### Exterior Sky Options

| Option   | Description |
| -------- | ----------- |
| `clear`  | Clear Sky   |
| `cloudy` | Cloudy Sky  |
| `rainy`  | Rainy Sky   |
| `snowy`  | Snowy Sky   |
| `misty`  | Misty Sky   |

<a id="exterior-landscape"></a>

##### Exterior Landscape Options

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

<a id="exterior-material"></a>

##### Architectural Materials

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

##### 2.7.3 Shared Options

<a id="daylight"></a>

##### Daylight Options

| Option    | Description |
| --------- | ----------- |
| `midday`  | Midday      |
| `night`   | Night       |
| `sunset`  | Sunset      |
| `sunrise` | Sunrise     |

<a id="season"></a>

##### Season Options

| Option   | Description                      |
| -------- | -------------------------------- |
| `spring` | Blooming flowers, fresh greenery |
| `summer` | Vibrant greens, full foliage     |
| `autumn` | Fall colors, orange/red foliage  |
| `winter` | Snow-covered, bare trees         |

<a id="creativity"></a>

##### Creativity Options

| Option     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `precise`  | Stays very close to the original image                |
| `balanced` | Moderate changes while maintaining original structure |
| `creative` | More significant artistic interpretation              |

#### 2.8 Upscale Image

**Endpoint:** `POST /api/enterprise/upscale`

**Description:**  
Submits an upscaling request using either a previously uploaded image (via the upload endpoint) or an external image URL. This enhances the resolution and quality of the image. The upscaled image will have a dimension of 1536 pixels on its shortest side.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `application/json` **Required**

**Request Body:**

| Parameter   | Type   | Required | Description                                  | Constraints                           |
| ----------- | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name` | string | Yes\*    | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `file_url`  | string | Yes\*    | The URL of the file to be upscaled           | Must be a valid URL                   |

\*Either `file_name` or `file_url` must be provided.

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>"
}
```

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

  - `render_id`: The unique ID of the upscaling request. Use this ID to check the upscale status (via the same endpoint used for checking render status).
  - Each upscale request produces 1 output image.

- **Error Responses:**

  ```json
  { "err": "<error_message>" }
  ```

  | Status Code | Possible Error Messages                                                                                                                               |
  | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
  | 400         | "Invalid request", "Invalid filename", "Invalid URL", "No file name or file URL provided", "Not enough credits", "Maximum concurrent renders reached" |
  | 401         | "Unauthorized"                                                                                                                                        |
  | 500         | "Image processing failed"                                                                                                                             |

**Examples:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/upscale' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>"
  }'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/upscale"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>",
    "Content-Type": "application/json"
}
payload = {
    "file_name": "<your_file_name>"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Upscale request submitted. Render ID: {render_id}")
else:
    print(f"Upscale request failed: {response.text}")
```

**Note:** The upscaling process consumes 4 credits per request. Use the same endpoint for checking render status (`GET /api/enterprise/render/<render_id>`) to check the status of upscaling requests.

### 3. Response for All Render Requests

**Success (200 OK):**

```json
{
  "success": true,
  "data": {
    "render_id": "<render_id>"
  }
}
```

- `render_id`: The unique ID of the rendering request. Use this ID to check render status.
- Each render request produces 1 output image.

**Error Responses:**

```json
{ "err": "<error_message>" }
```

| Status Code | Possible Error Messages                                                                                                                               |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| 400         | "Invalid request", "Invalid filename", "Invalid URL", "No file name or file URL provided", "Not enough credits", "Maximum concurrent renders reached" |
| 401         | "Unauthorized"                                                                                                                                        |
| 500         | "Image processing failed"                                                                                                                             |

**Empty Room Response Format:**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "field": "<field>",
  "status": "painted",
  "outputs": ["<s3_url_1>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

**Virtual Staging Response Format:**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "field": "<field>",
  "status": "painted",
  "outputs": ["<s3_url_1>"],
  "reference_file_urls": ["<s3_url_ref1>", "<s3_url_ref2>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

**Edit Image Response Format (action=prompt):**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "field": "<field>",
  "tool": "edit",
  "action": "<action>",
  "prompt": "<prompt>",
  "status": "<status>",
  "outputs": ["<s3_url_1>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

### 4. Check Render Status

- **Endpoint:** `GET /api/enterprise/render/<render_id>`
- **Description:** Checks the status of a rendering request.
- **Headers:**
  - `Authorization`: `Bearer <your_enterprise_api_key>`
- **Parameters:**
  - `render_id`: (Required) The ID of the rendering request (obtained from the response of `POST /api/enterprise/render`).

**Response (Success - 200 OK):**

The response format varies based on the render type (interior or exterior). Both response types include common status values and share base fields, with type-specific fields included based on the initial render request.

**Status Values:**

- `queued`: Rendering has not started yet
- `rendering`: Rendering is in progress
- `completed`: Rendering is complete
- `failed`: Rendering has failed

**Interior Render Response Format:**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "field": "interior",
  "type": "<interior_type>",
  "style": "<style>",
  "daylight": "<daylight>",
  "season": "<season>",
  "color": "<color>",
  "architectural_material": "<architectural_material>",
  "creativity": "<creativity>",
  "creativity_level": 0.7,
  "prompt": "<prompt>",
  "status": "<status>",
  "outputs": ["<s3_url_1>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

**Exterior Render Response Format:**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "field": "exterior",
  "type": "<exterior_type>",
  "style": "<style>",
  "daylight": "<daylight>",
  "sky": "<sky>",
  "season": "<season>",
  "landscape": "<landscape>",
  "architectural_material": "<architectural_material>",
  "creativity": "<creativity>",
  "creativity_level": 0.7,
  "prompt": "<prompt>",
  "status": "<status>",
  "outputs": ["<s3_url_1>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

**Sketch Response Format:**

```json
{
  "_id": "<render_id>",
  "uid": "<user_id>",
  "source": "<s3_url>",
  "type": "<type>",
  "style": "<style>",
  "view": "<view>",
  "prompt": "<prompt>",
  "status": "<status>",
  "outputs": ["<s3_url_1>"],
  "created_at": "<date>",
  "updated_at": "<date>"
}
```

**Upscale Response Format:**

```json
{
  "success": true,
  "data": {
    "_id": "<render_id>",
    "uid": "<user_id>",
    "source": "<s3_url>",
    "status": "<status>",
    "outputs": ["<s3_url_1>"],
    "created_at": "<date>",
    "updated_at": "<date>"
  }
}
```

**Note:** The `outputs` array will be empty for statuses `queued`, `rendering`, and `failed`. When status is `completed`, it will contain URLs to the rendered images on S3.

**Response (Error):**

```json
{ "err": "<error_message>" }
```

| Status Code | Possible Error Messages |
| ----------- | ----------------------- |
| 401         | "Unauthorized"          |
| 404         | "Render not found"      |

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

### 5. Get Credit Status

**Endpoint:** `GET /api/enterprise/status`

**Description:**  
Returns information about the current status of your enterprise account, including available credits, key information, and account status.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**

**Response:**

- **Success (200 OK):**

  ```json
  {
    "_id": "<account_id>",
    "uid": "<user_id>",
    "key": "<api_key>",
    "credits": -1,
    "is_active": true,
    "is_enterprise": true,
    "created_at": "<date>",
    "updated_at": "<date>"
  }
  ```

  - `_id`: The unique identifier for the account record
  - `uid`: The user identifier associated with the account
  - `key`: The API key being used (same as provided in Authorization header)
  - `credits`: Available credits (-1 indicates unlimited credits)
  - `is_active`: Whether the API key is currently active
  - `is_enterprise`: Whether the account has enterprise privileges
  - `created_at`: When the account was created
  - `updated_at`: When the account was last updated

- **Error Responses:**

  ```json
  { "err": "<error_message>" }
  ```

  | Status Code | Possible Error Messages |
  | ----------- | ----------------------- |
  | 401         | "Unauthorized"          |
  | 404         | "Account not found"     |

**Examples:**

```bash
curl -X GET \
  'https://reroom.ai/api/enterprise/status' \
  -H 'Authorization: Bearer <your_enterprise_api_key>'
```

```python
import requests

url = "https://reroom.ai/api/enterprise/status"
headers = {
    "Authorization": "Bearer <your_enterprise_api_key>"
}

response = requests.get(url, headers=headers)

if response.status_code == 200:
    data = response.json()
    credits = data["credits"]
    is_active = data["is_active"]
    print(f"Account status: {'Active' if is_active else 'Inactive'}")
    print(f"Available credits: {'Unlimited' if credits == -1 else credits}")
else:
    print(f"Status check failed: {response.text}")
```
