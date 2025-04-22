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
    - [Response for All Render Requests](#response-for-all-render-requests)
  - [Submit Upscale Request](#3-submit-upscale-request)
  - [Check Render Status](#4-check-render-status)

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
Submits a rendering request using a previously uploaded image. The endpoint uses a query parameter `field` to differentiate between interior and exterior rendering requests.

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
| `file_name`  | string | Yes      | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `creativity` | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"  |

**Optional Parameters:**

| Parameter          | Type   | Required | Description                  | Constraints                                                                                               |
| ------------------ | ------ | -------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `type`             | string | No       | The interior space type      | Must be one of the supported interior types. Defaults to "no-type" if not specified                       |
| `style`            | string | No       | The desired rendering style  | Must be one of the supported styles. Defaults to "no-style" if not specified                              |
| `daylight`         | string | No       | Time of day for lighting     | Must be one of the supported daylight options                                                             |
| `season`           | string | No       | Season to be depicted        | Must be one of the supported seasons                                                                      |
| `color`            | string | No       | Color palette                | Must be one of the supported color schemes                                                                |
| `prompt`           | string | No       | Additional text instructions | Free-form text                                                                                            |
| `creativity_level` | number | No       | Fine-tuned variation control | Range: 0.5 (conservative) to 1.0 (creative). Defaults to 0.7. Only applies when `creativity` is "precise" |

**Available Options for Interior Rendering:**

<details>
<summary>Type</summary>

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

</details>

<details>
<summary>Style</summary>

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

</details>

<details>
<summary>Daylight</summary>

| Option    | Description |
| --------- | ----------- |
| `midday`  | Midday      |
| `night`   | Night       |
| `sunset`  | Sunset      |
| `sunrise` | Sunrise     |

</details>

<details>
<summary>Season</summary>

| Option   | Description                      |
| -------- | -------------------------------- |
| `spring` | Blooming flowers, fresh greenery |
| `summer` | Vibrant greens, full foliage     |
| `autumn` | Fall colors, orange/red foliage  |
| `winter` | Snow-covered, bare trees         |

</details>

<details>
<summary>Color</summary>

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
<summary>Creativity</summary>

| Option     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `precise`  | Stays very close to the original image                |
| `balanced` | Moderate changes while maintaining original structure |
| `creative` | More significant artistic interpretation              |

</details>

**Note:** When `creativity` is set to "precise", you can fine-tune the variation level using the `creativity_level` parameter. This parameter is ignored for "balanced" and "creative" presets.

**Interior Rendering Example:**

```bash
curl -X POST \
  'https://reroom.ai/api/enterprise/render' \
  -H 'Authorization: Bearer <your_enterprise_api_key>' \
  -H 'Content-Type: application/json' \
  -d '{
    "file_name": "<your_file_name>",
    "creativity": "balanced",
    "type": "living-room",
    "style": "modern",
    "daylight": "midday"
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
    "creativity": "balanced",
    "type": "living-room",
    "style": "modern",
    "daylight": "midday"
}

response = requests.post(url, headers=headers, json=payload)

if response.status_code == 200:
    data = response.json()
    render_id = data["data"]["render_id"]
    print(f"Render request submitted. Render ID: {render_id}")
else:
    print(f"Render request failed: {response.text}")
```

#### 2.2 Exterior Rendering Parameters

Use these parameters when specifying `field=exterior` in the query string.

**Required Parameters:**

| Parameter    | Type   | Required | Description                                  | Constraints                           |
| ------------ | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name`  | string | Yes      | The filename returned by the upload endpoint | Must be a valid UUID+extension format |
| `creativity` | string | Yes      | Variation level preset                       | "precise", "balanced", or "creative"  |

**Optional Parameters:**

| Parameter          | Type   | Required | Description                  | Constraints                                                                                               |
| ------------------ | ------ | -------- | ---------------------------- | --------------------------------------------------------------------------------------------------------- |
| `type`             | string | No       | The building/structure type  | Must be one of the supported types. Defaults to "no-type" if not specified                                |
| `style`            | string | No       | The desired rendering style  | Must be one of the supported styles. Defaults to "no-style" if not specified                              |
| `daylight`         | string | No       | Time of day for lighting     | Must be one of the supported daylight options                                                             |
| `sky`              | string | No       | Sky condition                | Must be one of the supported sky styles                                                                   |
| `season`           | string | No       | Season to be depicted        | Must be one of the supported seasons                                                                      |
| `landscape`        | string | No       | Surrounding environment      | Must be one of the supported landscapes                                                                   |
| `material`         | string | No       | Primary building material    | Must be one of the supported materials                                                                    |
| `prompt`           | string | No       | Additional text instructions | Free-form text                                                                                            |
| `creativity_level` | number | No       | Fine-tuned variation control | Range: 0.5 (conservative) to 1.0 (creative). Defaults to 0.7. Only applies when `creativity` is "precise" |

**Request Body Example:**

```json
{
  "file_name": "<uuid>.<ext>",
  "creativity": "balanced",
  "type": "single-family-home",
  "style": "modern",
  "daylight": "sunset",
  "sky": "clear",
  "season": "summer"
}
```

**Available Options for Exterior Rendering:**

<details>
<summary>Type</summary>

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

</details>

<details>
<summary>Style</summary>

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

</details>
<details>
<summary>Daylight</summary>

| Option    | Description |
| --------- | ----------- |
| `midday`  | Midday      |
| `night`   | Night       |
| `sunset`  | Sunset      |
| `sunrise` | Sunrise     |

</details>

<details>
<summary>Sky</summary>

| Option   | Description |
| -------- | ----------- |
| `clear`  | Clear Sky   |
| `cloudy` | Cloudy Sky  |
| `rainy`  | Rainy Sky   |
| `snowy`  | Snowy Sky   |
| `misty`  | Misty Sky   |

</details>

<details>
<summary>Season</summary>

| Option   | Description |
| -------- | ----------- |
| `spring` | Spring      |
| `summer` | Summer      |
| `autumn` | Autumn      |
| `winter` | Winter      |

</details>

<details>
<summary>Landscape</summary>

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
<summary>Material</summary>

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
<summary>Creativity</summary>

| Option     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `precise`  | Stays very close to the original image                |
| `balanced` | Moderate changes while maintaining original structure |
| `creative` | More significant artistic interpretation              |

</details>

**Note:** When `creativity` is set to "precise", you can fine-tune the variation level using the `creativity_level` parameter. This parameter is ignored for "balanced" and "creative" presets.

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
    "style": "modern",
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
    "style": "modern",
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

### Response for All Render Requests

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

| Status Code | Possible Error Messages                                                                           |
| ----------- | ------------------------------------------------------------------------------------------------- |
| 400         | "Invalid request", "Invalid filename", "Not enough credits", "Maximum concurrent renders reached" |
| 401         | "Unauthorized"                                                                                    |
| 500         | "Image upload failed"                                                                             |

### 3. Submit Upscale Request

**Endpoint:** `POST /api/enterprise/upscale`

**Description:**  
Submits an upscaling request for a previously uploaded image. This enhances the resolution and quality of the image. The upscaled image will have a dimension of 1536 pixels on its shortest side.

**Headers:**

- `Authorization`: `Bearer <your_enterprise_api_key>` **Required**
- `Content-Type`: `application/json` **Required**

**Request Body:**

| Parameter   | Type   | Required | Description                                  | Constraints                           |
| ----------- | ------ | -------- | -------------------------------------------- | ------------------------------------- |
| `file_name` | string | Yes      | The filename returned by the upload endpoint | Must be a valid UUID+extension format |

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

  | Status Code | Possible Error Messages                                                                           |
  | ----------- | ------------------------------------------------------------------------------------------------- |
  | 400         | "Invalid request", "Invalid filename", "Not enough credits", "Maximum concurrent renders reached" |
  | 401         | "Unauthorized"                                                                                    |
  | 500         | "Image upload failed"                                                                             |

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
  "creativity": "<creativity>",
  "creativity_level": 0.7,
  "prompt": "<prompt>",
  "room_id": "<room_id>",
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
  "material": "<material>",
  "creativity": "<creativity>",
  "creativity_level": 0.7,
  "prompt": "<prompt>",
  "room_id": "<room_id>",
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
    "room_id": "<room_id>",
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
