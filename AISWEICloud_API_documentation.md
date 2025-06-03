**AiSWEI Cloud API Documentation**
*Version: Consolidated from PDF v1.1 & Reference Python Clients*

---

## 1. Introduction

The AiSWEI Cloud API allows developers to programmatically access photovoltaic (PV) plant data (e.g., station lists, energy generation, device status, and events). You can embed this data into your application or website to build custom dashboards or analytics. This document consolidates and clarifies the low‐quality source PDF (v1.1) and the provided Python client implementations to offer a clean, accurate, and complete reference.

**Key points:**

* All endpoints use HTTPS GET or POST requests to `https://eu-api-genergal.aisweicloud.com/pro/{endpoint}` or `https://eu-api-genergal.aisweicloud.com/{endpoint}` based on which version of the service you have(Yes, *genergal* is correct in the url)..
* Every request must be signed using HMAC-SHA256 with your `AppKey` and `AppSecret`.
* Rate limit: 100 requests per minute per AppKey.

---

## 2. Base URL & Common Headers

* **Base URL:**

  ```
  https://eu-api-genergal.aisweicloud.com/pro
  ```

* **Required HTTP Headers (all requests):**

  ```
  User-Agent:         YourAppName/1.0
  Content-Type:       application/json; charset=UTF-8
  Accept:             application/json
  Date:               <RFC-1123 GMT timestamp>               # e.g. “Tue, 03 Jun 2025 12:34:56 GMT”
  X-Ca-Key:           <Your AppKey>                          # Provided by AiSWEI
  X-Ca-Timestamp:     <Unix time in milliseconds>            # e.g. “1717410896000”
  X-Ca-Nonce:         <UUIDv4>                                # Anti-replay random string
  X-Ca-Signature:     <HMAC-SHA256 Signature>                 # Computed as described below
  X-Ca-Signature-Headers:  X-Ca-Key,X-Ca-Timestamp,X-Ca-Nonce # Always in this order
  ```

  * `Date` must be in UTC, formatted per [RFC 1123](https://tools.ietf.org/html/rfc1123).
  * `X-Ca-Key` is your AppKey (not the plant’s `apikey`/NMI).
  * `X-Ca-Timestamp` is valid for ±15 minutes.
  * `X-Ca-Nonce` + `X-Ca-Timestamp` must be unique per request (within 15 minutes).

---

## 3. Authentication & Signing

Every API call must be signed. The signature is an HMAC-SHA256 over a canonical string (the “StringToSign”). You then base64-encode the HMAC digest and place it in the `X-Ca-Signature` header. Below is a step-by-step summary.

### 3.1 Prepare Components

1. **HTTP Method**: Must be uppercase (e.g., `GET`, `POST`).
2. **Accept Header**: Always `application/json`.
3. **Content-Type Header**: Always `application/json; charset=UTF-8`.
4. **Date Header**: RFC 1123 formatted in UTC (e.g., `Tue, 03 Jun 2025 12:34:56 GMT`).
5. **Canonical Headers**:

   * Include only headers whose names start with `X-Ca-` (case-insensitive), **excluding** `X-Ca-Signature` and `X-Ca-Signature-Headers`.
   * Sort these header keys lexicographically (case-insensitive).
   * For each header in that sorted list, append a line:

     ```
     {header‐name}:{header‐value}\n
     ```
6. **Canonical URL**:

   * Start with the request path (e.g., `/getPlanListPro`).
   * Append a question mark (`?`) followed by all query parameters sorted lexicographically by key, URL-encoded.
   * If there are no query parameters, the canonical URL is just the path (no trailing `?`).

### 3.2 Construct StringToSign

```
StringToSign = 
   HTTP_METHOD + "\n" +
   Accept + "\n" +
   Content-MD5 + "\n" +         # Always empty (“\n”) for JSON/GET requests
   Content-Type + "\n" +
   Date + "\n" +
   CanonicalHeaders + 
   CanonicalURL
```

* **Content-MD5**:

  * Only needed if your request has a non-form JSON body.
  * Compute MD5 over the UTF-8 byte stream of the JSON body, then base64-encode it.
  * For GET requests (no body) or if you omit MD5, treat this line as empty (i.e., just a newline).

### 3.3 Compute HMAC Signature

```python
import hmac, hashlib, base64

secret_bytes = AppSecret.encode("utf-8")
message_bytes = StringToSign.encode("utf-8")
signature = base64.b64encode(
    hmac.new(secret_bytes, message_bytes, digestmod=hashlib.sha256).digest()
).decode("utf-8")
```

* Place the resulting `signature` string in the `X-Ca-Signature` header.
* In `X-Ca-Signature-Headers`, list the header keys you used (in this case: `X-Ca-Key,X-Ca-Timestamp,X-Ca-Nonce`).

---

## 4. Error Codes & Common Responses

* **HTTP Status Codes**:

  * `2xx` => Success
  * `4xx` => Client error (e.g., missing/invalid parameters, unauthorized)
  * `5xx` => Server error

* **JSON Response Envelope** (all endpoints):

  ```json
  {
    "status": <Integer>,
    "info": "<String message>",
    "time": "<yyyy-MM-dd HH:mm:ss>",
    "data": <JSON object or array>
  }
  ```

  * `status`:

    * `200` → OK
    * `10001` → Parameter error
    * `10012` → Device does not exist
    * `10403` → No permission to view this station/device
    * `20006` → User does not exist
    * Other codes as documented per endpoint.
  * `info`: Human-readable status/message.
  * `time`: Server’s current time in `yyyy-MM-dd HH:mm:ss`.
  * `data`: The payload (varies by endpoint).

---

## 5. API Endpoints

Below is the list of all available “Pro” endpoints with complete request/response details.

### 5.1 getPlanListPro (List of PV Plants)

* **Request Path:**

  ```
  GET /getPlanListPro
  ```

  Full URL: `https://eu-api-genergal.aisweicloud.com/pro/getPlanListPro?token={token}&order={order}&pageNum={pageNum}&pageSize={pageSize}`

* **Query Parameters:**

  | Name     | Type    | Description                                                                               | Required |
  | -------- | ------- | ----------------------------------------------------------------------------------------- | -------- |
  | token    | String  | User token (same as “apikey” in code samples)                                             | Yes      |
  | order    | Integer | Sort order: 0 = order by ludt (last update), 1 = order by createTime, 2 = order by status | Yes      |
  | pageNum  | Integer | Page number (1-based)                                                                     | Yes      |
  | pageSize | Integer | Items per page                                                                            | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "totalPages": <Integer>,
    "totalElements": <Integer>,
    "pageNum": <Integer>,
    "pageSize": <Integer>,
    "result": [
      {
        "country": <Integer>,         // Country code (e.g., 86 = China)
        "totalpower": <Float>,        // Total inverter power (kW)
        "apikey": "<String>",         // Plant’s NMI key
        "city": "<String>",
        "accountName": "<String>",    // Owner’s mobile/email
        "ludt": "<String>",           // Last update time (yyyy-MM-dd HH:mm:ss)
        "etoday": <Float>,            // Energy generated today (kWh)
        "createdt": "<String>",       // Station creation time
        "wd": <Double>,               // Latitude
        "etotal": <Float>,            // Total energy generated (kWh)
        "imgurl": "<String>",         // Background image URL
        "province": <Integer>,        // Province code
        "name": "<String>",           // Plant name
        "jd": <Double>,               // Longitude
        "position": "<String>",       // Plant address
        "status": <Integer>           // Status: 0=offline,1=normal,2=warning,3=error
      }
      // … more elements …
    ]
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-16 14:39:25",
    "data": {
      "totalPages": 24,
      "totalElements": 24,
      "pageNum": 1,
      "pageSize": 1,
      "result": [
        {
          "country": 86,
          "totalpower": 21.15,
          "apikey": "be9ca0323e4149758d0b02ea3310cxxx",
          "city": "",
          "accountName": "187xxxxxxxx",
          "ludt": "2022-05-16 14:35:08",
          "etoday": 101.2,
          "createdt": "2022-05-13 21:34:10",
          "wd": 34.174373,
          "etotal": 302.9,
          "imgurl": "https://www.zevercloud.com/upload/station/defultBg.jpg",
          "province": 26,
          "name": "兴平 XYZL",
          "jd": 108.40205,
          "position": "中国陕西省西安市周至县老大路",
          "status": 1
        }
      ]
    }
  }
  ```

---

### 5.2 getPlantOverviewPro (Plant Overview Metrics)

* **Request Path:**

  ```
  GET /getPlantOverviewPro
  ```

  Full URL: `https://eu-api-genergal.aisweicloud.com/pro/getPlantOverviewPro?apikey={apikey}&token={token}`

* **Query Parameters:**

  | Name   | Type   | Description     | Required |
  | ------ | ------ | --------------- | -------- |
  | apikey | String | Plant’s NMI key | Yes      |
  | token  | String | User token      | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "ludt": "<String>",     // Last update time (yyyy-MM-dd HH:mm:ss)
    "apikey": "<String>",   // Plant’s NMI key
    "status": "<String>",   // Plant status: "0"=offline, "1"=normal, "2"=warning, "3"=error
    "E-Month": {
      "unit": "<String>",   // e.g., "KWh"
      "value": <Float>
    },
    "CO2Avoided": {
      "unit": "<String>",   // e.g., "Kg"
      "value": <Float>
    },
    "E-Today": {
      "unit": "<String>",   // e.g., "KWh"
      "value": <Float>
    },
    "TotalYield": {
      "unit": "<String>",   // usually empty string
      "value": <Float>
    },
    "E-Total": {
      "unit": "<String>",   // e.g., "KWh"
      "value": <Float>
    },
    "E-Year": {
      "unit": "<String>",   // e.g., "KWh"
      "value": <Float>
    },
    "Power": {
      "unit": "<String>",   // e.g., "W"
      "value": <Float>
    }
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-09 15:28:22",
    "data": {
      "E-Month": { "unit": "KWh", "value": 42.1 },
      "CO2Avoided": { "unit": "Kg", "value": 6.0 },
      "E-Today": { "unit": "KWh", "value": 3.6 },
      "TotalYield": { "unit": "", "value": 3.2 },
      "E-Total": { "unit": "KWh", "value": 180.6 },
      "E-Year": { "unit": "KWh", "value": 201.0 },
      "ludt": "2022-04-09 14:22:53",
      "Power": { "unit": "W", "value": 42.0 },
      "apikey": "sada34",
      "status": "1"
    }
  }
  ```

---

### 5.3 getPlantOutputPro (Plant Energy Output over Time)

* **Request Path:**

  ```
  GET /getPlantOutputPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getPlantOutputPro?
    apikey={apikey}
    &period={period}
    &date={date}
    &token={token}
  ```

* **Query Parameters:**

  | Name   | Type   | Description                    | Required |
  | ------ | ------ | ------------------------------ | -------- |
  | apikey | String | Plant’s NMI key                | Yes      |
  | period | String | Granularity of output. One of: |          |

  ```
             - `bydays` → daily (requires `date = yyyy-MM-dd`)  
             - `bymonth` → monthly (requires `date = yyyy-MM`)  
             - `byyear` → yearly (requires `date = yyyy`)  
             - `bytotal` → all-time (ignore `date`)                                                       | Yes      |
  ```

  \| date   | String | As described above                                                                                                                                  | Cond.    |
  \| token  | String | User token                                                                                                                                      | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "apikey": "<String>",      // Plant’s NMI key
    "dataunit": "<String>",    // e.g., "KWh"
    "result": [
      {
        "no": "<String>",      // Sequence or code (1,2,3…)
        "time": "<String>",    // Year / Month / Day (depending on period)
        "value": "<String>"    // Energy output value (same unit as dataunit)
      }
      // … more entries …
    ]
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-09 15:28:32",
    "data": {
      "result": [
        { "no": "1", "time": "2022", "value": "4.0" }
      ],
      "apikey": "asd124325",
      "dataunit": "KWh"
    }
  }
  ```

---

### 5.4 getPlantEventPro (Plant Events / Alerts)

* **Request Path:**

  ```
  GET /getPlantEventPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getPlantEventPro?
    apikey={apikey}
    &sdt={startDate}
    &edt={endDate}
    &token={token}
  ```

* **Query Parameters:**

  | Name   | Type   | Description                      | Format     | Required |
  | ------ | ------ | -------------------------------- | ---------- | -------- |
  | apikey | String | Plant’s NMI key                  | –          | Yes      |
  | token  | String | User token                       | –          | Yes      |
  | sdt    | String | Start date/time for query window | yyyy-MM-dd | Yes      |
  | edt    | String | End date/time for query window   | yyyy-MM-dd | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "apikey": "<String>",   // Plant’s NMI key
    "result": [
      {
        "eventCode": "<String>",  // Numeric code (see Event Code table—Section 9)
        "ssno": "<String>",       // Device serial number (inverter or PMU)
        "eventTime": "<String>",  // “yyyy-MM-dd HH:mm:ss”
        "eventType": "<String>"   // 1=message,2=warning,3=error
      }
      // … more events …
    ]
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-09 15:28:37",
    "data": {
      "result": [
        {
          "eventCode": "304",
          "ssno": "SX00040311440018",
          "eventTime": "2022-05-01 13:45:11",
          "eventType": "3"
        }
      ],
      "apikey": "asd45642"
    }
  }
  ```

---

### 5.5 getDeviceListPro (Devices under a Plant)

* **Request Path:**

  ```
  GET /getDeviceListPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getDeviceListPro?
    apikey={apikey}
    &token={token}
  ```

* **Query Parameters:**

  | Name   | Type   | Description     | Required |
  | ------ | ------ | --------------- | -------- |
  | apikey | String | Plant’s NMI key | Yes      |
  | token  | String | User token      | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  [
    {
      "psn": "<String>",           // PMU (collector) serial number
      "pstate": <Integer>,         // PMU state: 0=offline,1=normal,9=not active
      "inverters": [
        {
          "isn": "<String>",       // Inverter serial number
          "ludt": "<String>",      // Last communication time (yyyy-MM-dd HH:mm:ss)
          "istate": <Integer>      // Inverter state: 0=offline,1=normal,2=cache
        }
        // … more inverters under this PMU …
      ]
    }
    // … more PMUs under this plant …
  ]
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-09 15:28:40",
    "data": [
      {
        "pstate": 0,
        "inverters": [
          {
            "isn": "5.0NX12000010",
            "ludt": "2022-05-09 15:02:03",
            "istate": 0
          }
        ],
        "psn": "B3200A2305D7"
      }
    ]
  }
  ```

---

### 5.6 getLocationPro (Device Geolocation)

* **Request Path:**

  ```
  GET /getLocationPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getLocationPro?
    psno={psno}
    &token={token}
  ```

* **Query Parameters:**

  | Name  | Type   | Description       | Required |
  | ----- | ------ | ----------------- | -------- |
  | psno  | String | PMU serial number | Yes      |
  | token | String | User token        | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "address": "<String>",  // Geocoded address (e.g., “中国江苏省镇江市扬中市…”)
    "jd": "<String>",       // Longitude (as string)
    "wd": "<String>"        // Latitude (as string)
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-09 15:28:44",
    "data": {
      "address": "中国江苏省镇江市扬中市港兴路辅路长乐村北 272米",
      "jd": "119.861289",
      "wd": "32.201583"
    }
  }
  ```

---

### 5.7 getLastTsDataPro (Latest Timestamp Data for Inverters)

* **Request Path:**

  ```
  GET /getLastTsDataPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getLastTsDataPro?
    isnos={comma-separated inverter SNs}
    &token={token}
  ```

* **Query Parameters:**

  | Name  | Type   | Description                                               | Required |
  | ----- | ------ | --------------------------------------------------------- | -------- |
  | isnos | String | Comma-separated inverter serial numbers (e.g., “111,222”) | Yes      |
  | token | String | User token                                                | Yes      |

* **Response Fields (`data`):** (Array of objects—one per `isno`)

  ```jsonc
  [
    {
      "sn": "<String>",         // Inverter serial number
      "tmstp": "<String>",      // Unix timestamp in ms (as string)
      "apikey": "<String>",     // Plant’s NMI key
      "tim": "<String>",        // Last update time (“yyyy-MM-dd HH:mm:ss”)
      "pac": "<String>",        // Active power (W)
      "prc": "<String>",        // Reactive power (var)
      "sac": "<String>",        // Apparent power (VA)
      "pf": "<String>",         // Power factor (×0.01)
      "etd": "<String>",        // Daily energy (kWh × 10)
      "eto": "<String>",        // Total energy (kWh × 10)
      "hto": "<String>",        // Grid-connection hours (h)
      "cf": "<String>",         // Heat sink temperature (°C × 10)
      "tu": "<String>",         // U-phase temp (°C × 10)
      "tv": "<String>",         // V-phase temp
      "tw": "<String>",         // W-phase temp
      "cb": "<String>",         // Boost temperature (°C × 10)
      "bv": "<String>",         // Bus voltage (V × 10)
      "er": "<String>",         // Error code (“0” = none)
      "wn0": "<String>",        // Warning code
      "v1": "<String>",         // MPPT1 voltage (V × 10)
      "v2": "<String>",         // MPPT2 voltage
      "v3": "<String>",         // MPPT3 voltage
      "i1": "<String>",         // MPPT1 current (A × 100)
      "i2": "<String>",         // MPPT2 current
      "i3": "<String>",         // MPPT3 current
      "s1":"<String>", … “s9”:  // String currents (A × 10; up to s10
      "va1":"<String>",         // AC voltage phase A (V × 10)
      "va2":"<String>",         // AC voltage phase B
      "va3":"<String>",         // AC voltage phase C
      "ia1":"<String>",         // AC current phase A (A)
      "ia2":"<String>",
      "ia3":"<String>",
      "fac": "<String>"         // Grid frequency (Hz × 100)
    }
    // … more inverter objects …
  ]
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-05-10 19:43:22",
    "data": [
      {
        "sn": "SZ001786221B0635",
        "tmstp": "1652182736000",
        "apikey": "145371",
        "tim": "2022-05-10 19:38:56",
        "pac": "0",
        "prc": "0",
        "sac": "0",
        "pf": "0",
        "etd": "456",
        "eto": "88273",
        "hto": "1439",
        "cf": "278",
        "tu": "232",
        "tv": "232",
        "tw": "232",
        "cb": "232",
        "bv": "5850",
        "er": "0",
        "wn0": "0",
        "v1": "1501",
        "v2": "3527",
        "v3": null,
        "i1": "0",
        "i2": "11",
        "i3": null,
        "s1": null,
        "s2": null,
        "s3": null,
        "va1": "2314",
        "va2": "2361",
        "va3": "2348",
        "ia1": "10",
        "ia2": "9",
        "ia3": "8",
        "fac": "4996"
      }
    ]
  }
  ```

---

### 5.8 getInverterDataPagePro (Historical Inverter Data, Paginated)

* **Request Path:**

  ```
  GET /getInverterDataPagePro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterDataPagePro?
    token={token}
    &apikey={apikey}
    &isnos={comma-sep inverter SNs}
    &startDate={yyyy-MM-dd HH:mm:ss}
    &endDate={yyyy-MM-dd HH:mm:ss}
    &pageNum={pageNum}
    &pageSize={pageSize}
  ```

* **Query Parameters:**

  | Name      | Type    | Description                                            | Required |
  | --------- | ------- | ------------------------------------------------------ | -------- |
  | token     | String  | User token                                             | Yes      |
  | apikey    | String  | Plant’s NMI key                                        | Yes      |
  | isnos     | String  | Comma-separated inverter SNs (e.g., “111,222”)         | Cond.    |
  | startDate | String  | Query start time (“yyyy-MM-dd HH\:mm\:ss”) (inclusive) | Yes      |
  | endDate   | String  | Query end time (“yyyy-MM-dd HH\:mm\:ss”) (inclusive)   | Yes      |
  | pageNum   | Integer | 1-based page number                                    | Yes      |
  | pageSize  | Integer | Items per page                                         | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "totalPages": <Integer>,
    "totalElements": <Integer>,
    "pageNum": <Integer>,
    "pageSize": <Integer>,
    "result": [
      {
        "apikey": "<String>",    // Plant’s NMI key
        "isno": "<String>",      // Inverter serial number
        "dataList": [
          {
            "tu": "<String>",     // U-phase temp (°C × 10)
            "tv": "<String>",     // V-phase temp
            "tmstp": "<String>",  // Timestamp (ms)
            "tw": "<String>",     // W-phase temp
            "insdt": "<String>",  // Record insertion time (“yyyy-MM-dd HH:mm:ss”)
            "fac": "<String>",    // Grid frequency (Hz × 100)
            "psn": "<String>",    // PMU serial number
            "pac": "<String>",    // Active power (W)
            "bv": "<String>",     // Bus voltage (V × 10)
            "etd": "<String>",    // Daily energy (kWh × 10)
            "sac": "<String>",    // Apparent power (VA)
            "smp": "<String>",    // Sampling frequency
            "ia1": "<String>",    // AC current phase A (A)
            "tim": "<String>",    // Last update time (“yyyy-MM-dd HH:mm:ss”)
            "ia3": "<String>",    // AC current phase C
            "ia2": "<String>",    // AC current phase B
            "sn": "<String>",     // Inverter SN
            "s1": "<String>",     // String 1 current (A × 10)
            "cb": "<String>",     // Boost temperature (°C × 10)
            "s2": "<String>",     // String 2 current
            "prc": "<String>",    // Reactive power (var)
            "eto": "<String>",    // Total energy (kWh × 10)
            "hto": "<String>",    // Grid hours (h)
            "cf": "<String>",     // Heat sink temp (°C × 10)
            "va1": "<String>",    // AC voltage phase A (V × 10)
            "va2": "<String>",    // AC voltage phase B
            "va3": "<String>",    // AC voltage phase C
            "i1": "<String>",     // MPPT1 current (A × 100)
            "i2": "<String>",     // MPPT2 current
            "i3": "<String>",     // MPPT3 current
            "itv": "<String>",    // Interval minutes
            "er": "<String>",     // Error code
            "pf": "<String>",     // Power factor (× 0.01)
            "v1": "<String>",     // MPPT1 voltage (V × 10)
            "v2": "<String>",     // MPPT2 voltage
            "v3": "<String>",     // MPPT3 voltage
            "wn0": "<String>",    // Warning code
            // s1 … s6 are string currents (up to s6) if present
          }
          // … more data points …
        ]
      }
      // … more inverter entries if multiple isnos requested …
    ]
  }
  ```

* **Example Response (Truncated):**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-20 15:14:16",
    "data": {
      "totalPages": 1,
      "totalElements": 1,
      "pageNum": 1,
      "pageSize": 3,
      "result": [
        {
          "apikey": "a25a6c9b9f3045e9b4105d1c5bd1fc8b",
          "isno": "xxx4000122300xx",
          "dataList": [
            {
              "tu": "294",
              "tv": "294",
              "tmstp": "1655242295000",
              "tw": "294",
              "insdt": "2022-06-15 05:31:37",
              "fac": "4998",
              "psn": "xx00A230xx",
              "pac": "1251",
              "bv": "6029",
              "etd": "4",
              "sac": "227",
              "smp": "5",
              "ia1": "26",
              "tim": "2022-06-15 05:31:35",
              "ia3": "28",
              "sn": "xxx4000122300xx",
              "ia2": "26",
              "s1": "0",
              "cb": "297",
              "s2": "11",
              "prc": "0",
              "s3": "0",
              "eto": "101576",
              "hto": "674",
              "s4": "0",
              "cf": "273",
              "s5": "0",
              "va2": "2213",
              "va1": "2221",
              "i1": "185",
              "i2": "145",
              "va3": "2218",
              "i3": "100",
              "itv": "331",
              "er": "0",
              "pf": "100",
              "wn0": "0",
              "v1": "4203",
              "v2": "4286",
              "v3": "4190"
            }
          ]
        }
      ]
    }
  }
  ```

---

### 5.9 getInverterETodayPro (Daily Energy for Inverters)

* **Request Path:**

  ```
  GET /getInverterETodayPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterETodayPro?
    token={token}
    &apikey={apikey}
    &isnos={comma-separated inverter SNs}
    &date={yyyy-MM-dd}
  ```

* **Query Parameters:**

  | Name   | Type   | Description                             | Required |
  | ------ | ------ | --------------------------------------- | -------- |
  | token  | String | User token                              | Yes      |
  | apikey | String | Plant’s NMI key                         | Yes      |
  | isnos  | String | Comma-separated inverter serial numbers | Cond.    |
  | date   | String | Date for which to retrieve daily energy | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "date": "<String>",    // Query date (“yyyy-MM-dd”)
    "result": [
      {
        "<Inverter_SN>": <Float>   // Daily energy (kWh; unit implied)
      }
      // … more inverter entries …
    ]
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-20 17:04:30",
    "data": {
      "date": "2022-06-19",
      "result": [
        { "SP00250022240383": 145.3 }
      ]
    }
  }
  ```

---

### 5.10 getInverterHisErrorPagePro (Historical Inverter Errors, Paginated)

* **Request Path:**

  ```
  GET /getInverterHisErrorPagePro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterHisErrorPagePro?
    token={token}
    &apikey={apikey}
    &isno={inverterSN}
    &startDate={yyyy-MM-dd HH:mm:ss}
    &endDate={yyyy-MM-dd HH:mm:ss}
    &pageNum={pageNum}
    &pageSize={pageSize}
  ```

* **Query Parameters:**

  | Name      | Type    | Description                               | Required |
  | --------- | ------- | ----------------------------------------- | -------- |
  | token     | String  | User token                                | Yes      |
  | apikey    | String  | Plant’s NMI key                           | Yes      |
  | isno      | String  | Inverter serial number                    | Cond.    |
  | startDate | String  | Start timestamp (“yyyy-MM-dd HH\:mm\:ss”) | Yes      |
  | endDate   | String  | End timestamp (“yyyy-MM-dd HH\:mm\:ss”)   | Yes      |
  | pageNum   | Integer | 1-based page number                       | Yes      |
  | pageSize  | Integer | Items per page                            | Yes      |

* **Response Fields (`data`):**

  ```jsonc
  {
    "totalPages": <Integer>,
    "totalElements": <Integer>,
    "pageNum": <Integer>,
    "pageSize": <Integer>,
    "result": [
      {
        "errorDate": <Long>,    // UNIX timestamp in ms of error occurrence
        "errorCode": <Integer>, // Numeric error code (see Event Code table)
        "errorType": <Integer>, // 3=error
        "context": "<String>",  // Error description (Chinese/English)
        "isno": "<String>"      // Inverter serial number
      }
      // … more error records …
    ]
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-21 11:06:27",
    "data": {
      "totalPages": 1,
      "totalElements": 1,
      "pageNum": 1,
      "pageSize": 1,
      "result": [
        {
          "errorDate": 1655326167000,
          "errorCode": 35,
          "errorType": 3,
          "context": "35 - 电网丢失",
          "isno": "AR80000112190001"
        }
      ]
    }
  }
  ```

---

### 5.11 getInverterCurrentErrorPro (Current Inverter Errors)

* **Request Path:**

  ```
  GET /getInverterCurrentErrorPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterCurrentErrorPro?
    token={token}
    &isno={inverterSN}
    &apikey={apikey}
  ```

* **Query Parameters:**

  | Name   | Type   | Description            | Required |
  | ------ | ------ | ---------------------- | -------- |
  | token  | String | User token             | Yes      |
  | apikey | String | Plant’s NMI key        | Cond.    |
  | isno   | String | Inverter serial number | Cond.    |

* **Response Fields (`data`):** (Array of current errors—usually 0 or 1 entry)

  ```jsonc
  [
    {
      "context": "<String>",    // Error description
      "errorCode": <Integer>,   // Numeric code
      "errorType": <Integer>,   // 3 = error
      "faultCode": "<String>",  // Unique error code string
      "isno": "<String>",       // Inverter SN
      "errorDate": <Long>       // UNIX timestamp in ms
    }
    // … (possibly more current error entries) …
  ]
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-21 11:16:55",
    "data": [
      {
        "context": "35 - 电网丢失",
        "errorCode": 35,
        "errorType": 3,
        "faultCode": "xxx001xxx",
        "isno": "AR80000112190001",
        "errorDate": 1655758207000
      }
    ]
  }
  ```

---

### 5.12 getInverterOverviewPro (Inverter Summary Metrics)

* **Request Path:**

  ```
  GET /getInverterOverviewPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterOverviewPro?
    token={token}
    &apikey={apikey}
    &date={yyyy-MM}
  ```

* **Query Parameters:**

  | Name   | Type   | Description                                                                               | Required |
  | ------ | ------ | ----------------------------------------------------------------------------------------- | -------- |
  | token  | String | User token                                                                                | Yes      |
  | apikey | String | Plant’s NMI key                                                                           | Yes      |
  | date   | String | Month for aggregation (format: yyyy-MM). If omitted, server may default to current month. | Cond.    |

* **Response Fields (`data`):** (Array of per-inverter summaries)

  ```jsonc
  [
    {
      "apikey": "<String>",       // Plant’s NMI key
      "result": [
        {
          "isno": "<String>",     // Inverter serial number
          "e_today": "<String>",  // Daily energy (kWh)
          "e_month": "<String>",  // Monthly energy (kWh)
          "e_total": "<String>",  // Annual energy (kWh)
          "co2": <Double>,        // CO₂ emission reduction (Kg)
          "yield": <Double>,      // Yield coefficient
          "recvdate": "<String>"  // Last update time (“yyyy-MM-dd HH:mm:ss.SSS”)
        }
        // … more inverter summaries …
      ]
    }
    // … possibly multiple plants if you requested multiple `apikey`s? Usually one‐element array …
  ]
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-21 16:38:04",
    "data": [
      {
        "apikey": "xxx2bec7dfd4da69ef07b3783e43xxx",
        "result": [
          {
            "isno": "SP00200022252664",
            "e_today": "117.60",
            "e_month": "127.600",
            "e_total": "407.90",
            "co2": 326.32,
            "yield": 163.16,
            "recvdate": "2022-06-21 16:33:13.0"
          }
        ]
      }
    ]
  }
  ```

---

### 5.13 getInverterRecoverStatusPro (Inverter Fault Recovery Status)

* **Request Path:**

  ```
  GET /getInverterRecoverStatusPro
  ```

  Full URL:

  ```
  https://eu-api-genergal.aisweicloud.com/pro/getInverterRecoverStatusPro?
    token={token}
    &faultCodes={comma-sep faultCode strings}
  ```

* **Query Parameters:**

  | Name       | Type   | Description                                                                    | Required |
  | ---------- | ------ | ------------------------------------------------------------------------------ | -------- |
  | token      | String | User token                                                                     | Yes      |
  | faultCodes | String | Comma-separated “faultCode” strings returned by current/historical error calls | Yes      |

* **Response Fields (`data`):** (Array, one per `faultCode` requested)

  ```jsonc
  [
    {
      "recoverTime": <Long>,    // UNIX timestamp in ms when fault recovered, or null if not recovered
      "recover": <Integer>,     // Recovery status: 0 = not recovered, 1 = restored
      "faultCode": "<String>"   // Echoed faultCode
    }
    // … more faultCode entries …
  ]
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-06-24 16:56:36",
    "data": [
      {
        "recoverTime": 1656043923000,
        "recover": 1,
        "faultCode": "Q0gwMDUwMDAxMjI1MDA5MnwzNHwxNjU2MDQzNzMzMDAw"
      },
      {
        "recoverTime": null,
        "recover": 0,
        "faultCode": "U1AwMDIzMDAyMjI1MDQwM3wzNXwxNjU1NzU4MDkwMDAw"
      }
    ]
  }
  ```

---

### 5.14 createstationPro (Create/Register a New Station)

* **Request Path:**

  ```
  POST /createstationPro
  ```

  Full URL: `https://eu-api-genergal.aisweicloud.com/pro/createstationPro`

* **Request Body (JSON):**

  ```jsonc
  {
    "token": "<String>",          // User token
    "psno": "<String>",           // PMU serial number
    "registno": "<String>",       // PMU registration code
    "name": "<String>",           // Station name
    "totalpower": <Double>,       // Total inverter power (kW). Default = 0
    "area": "<String>",           // Station type: "1" = household, "2" = commercial
    "timezone": <Long>,           // Time zone offset. China default = 8
    "wkey": <Long>,               // Time zone code. China default = 7800
    "customerflag": <Long>,       // DST flag: 0 = not in effect, 1 = DST. Default = 0
    "co2xs": <Long>,              // CO₂ factor. Default = 0.7
    "gainxs": <Long>,             // Revenue factor. Default = 0.8
    "country": <Long>,            // Country code. Default = 86
    "province": <Long>,           // Province code. Default = 0
    "city": "<String>",           // City (optional)
    "jd": <Double>,               // Longitude. Default = 0
    "wd": <Double>,               // Latitude. Default = 0
    "locale": "<String>"          // Language code. Default = "zh_CN"
  }
  ```

* **Response Fields (`data`):**

  ```jsonc
  {
    "apikey": "<String>"   // Newly created plant’s NMI key (use this for subsequent calls)
  }
  ```

* **Example Request Body:**

  ```json
  {
    "token": "user-token-123",
    "psno": "PMU1234567890",
    "registno": "REGCODE123",
    "name": "My Household PV",
    "totalpower": 5.5,
    "area": "1",
    "country": 86,
    "province": 16,
    "city": "Nanjing",
    "jd": 118.78,
    "wd": 32.04,
    "locale": "zh_CN"
  }
  ```

* **Example Response:**

  ```json
  {
    "status": 200,
    "info": "success",
    "time": "2022-08-05 16:21:50",
    "data": {
      "apikey": "NEWPLANTAPIKEY123"
    }
  }
  ```

---

## 6. Event Code Descriptions

For detailed inverter event codes (e.g., errors, warnings), refer to the table below. These codes appear in `eventCode` (plant events) or `errorCode` (inverter errors). The context string often provides a brief description in Chinese and/or English.

| Error Code | Description                                        |
| ---------- | -------------------------------------------------- |
| 101        | SCI Fault                                          |
| 102        | EEPROM R/W Fault                                   |
| 103        | RLY-Check Fault                                    |
| 104        | DC INJ. High                                       |
| 105        | AUTO TEST FAILED                                   |
| 106        | High DC Bus                                        |
| 107        | Ref. Voltage Fault                                 |
| 108        | AC HCT Fault                                       |
| 109        | GFCI Fault                                         |
| 110        | Device Fault                                       |
| 111        | M-S version unmatched                              |
| 132        | Reserved                                           |
| 133        | Fac Fault                                          |
| 134        | Vac Fault                                          |
| 135        | Utility Loss                                       |
| 136        | Ground fault                                       |
| 137        | PV Over voltage                                    |
| 138        | ISO Fault                                          |
| 140        | Over Temp.                                         |
| 141        | Vac differs for M-S                                |
| 142        | Fac differs for M-S                                |
| 143        | Ground I differs for M-S                           |
| 144        | DC inj. differs for M-S                            |
| 145        | Fac, Vac differs for M-S                           |
| 146        | High DC Bus                                        |
| 147        | Consistent Fault                                   |
| 148        | Average volt of 10 minutes Fault                   |
| 152        | Fuse Fault                                         |
| 153        | ISO check: before enable current (ISO > 300 mV)    |
| 154        | ISO check: after enable current (ISO out of range) |
| 155        | ISO check: N/P relay change (ISO < 40 mV)          |
| 156        | GFCI protect fault: 30 mA                          |
| 157        | GFCI protect fault: 60 mA                          |
| 158        | GFCI protect fault: 150 mA                         |
| 159        | PV1 string current abnormal                        |
| 160        | PV2 string current abnormal                        |
| 161        | DRED Communication Fails (S9 open)                 |
| 162        | Operate the disconnection device (S0 close)        |

---

## 7. Abbreviations & Definitions

* **PV**: Photovoltaic
* **PMU**: Power Monitor Unit (collector)
* **NMI**: National Metering Identifier (used as `apikey`)
* **SN**: Serial Number
* **kWh**: Kilowatt-hour
* **W**: Watt
* **VA**: Volt-ampere
* **GFCI**: Ground Fault Circuit Interrupter
* **ISO**: Insulation Monitoring Device
* **E-Today**: Energy generated today (kWh)
* **E-Total**: Total energy generated (kWh)
* **E-Month**: Energy generated this month (kWh)
* **CO2Avoided**: Carbon dioxide emissions avoided (Kg)

---

## 8. Python Client Examples

Below are minimal snippets based on the provided Python clients to illustrate how to call an endpoint.

### 8.1 Initialization

```python
import asyncio
from aiswei_pro_client import AiSWEICloudClient

APIKEY = "your_plant_apikey"      # Plant’s NMI key
APP_KEY = "your_appkey"           # Obtained from AiSWEI
APP_SECRET = "your_appsecret"     # Obtained from AiSWEI

client = AiSWEICloudClient(apikey=APIKEY, app_key=APP_KEY, app_secret=APP_SECRET)
```

### 8.2 Retrieve Plant List

```python
async def fetch_plants():
    response = await client.get_station_list(page_size=50, page_number=1, order_by=0)
    print(response)

asyncio.run(fetch_plants())
```

* Under the hood, `get_station_list` builds:

  * URL: `/getPlanListPro?order=0&pageNum=1&pageSize=50&token=your_plant_apikey`
  * Fills `base_headers()`, computes `X-Ca-Signature`, then sends a GET to `https://eu-api-genergal.aisweicloud.com/pro/getPlanListPro?...`.

### 8.3 Other Endpoints

Every other endpoint follows the same pattern:

1. Build query parameters dictionary (e.g., `{"apikey": APIKEY, "token": API_TOKEN}`).
2. Call the appropriate method (e.g., `getPlantOverviewPro`, `getPlantEventPro`, etc.).
3. Handle JSON response.

If you need a POST (like `createstationPro`), adapt the code to `client._make_request("POST", "createstationPro", params=None, data=payload)` (or implement a dedicated wrapper).

---

## 9. Country & Province Codes

For Chinese plants (as the PDF listed), use the following in `country` and `province` fields when creating stations. If you operate in other countries, confirm codes with AiSWEI.

| Country | Country Code | Province Code | Province Name     |
| ------- | ------------ | ------------- | ----------------- |
| China   | 86           | 1             | Anhui             |
| China   | 86           | 2             | Beijing           |
| China   | 86           | 3             | Chongqing         |
| China   | 86           | 4             | Fujian            |
| China   | 86           | 5             | Gansu             |
| China   | 86           | 6             | Guangdong         |
| China   | 86           | 7             | Guangxi           |
| China   | 86           | 8             | Guizhou           |
| China   | 86           | 9             | Hainan            |
| China   | 86           | 10            | Hebei             |
| China   | 86           | 11            | Heilongjiang      |
| China   | 86           | 12            | Henan             |
| China   | 86           | 13            | Hong Kong         |
| China   | 86           | 14            | Hubei             |
| China   | 86           | 15            | Hunan             |
| China   | 86           | 16            | Jiangsu           |
| China   | 86           | 17            | Jiangxi           |
| China   | 86           | 18            | Jilin             |
| China   | 86           | 19            | Liaoning          |
| China   | 86           | 20            | Macau             |
| China   | 86           | 21            | Inner Mongolia    |
| China   | 86           | 22            | Ningxia           |
| China   | 86           | 23            | Qinghai           |
| China   | 86           | 24            | Shandong          |
| China   | 86           | 25            | Shanxi            |
| China   | 86           | 26            | Shanxi (Shaanxi?) |
| China   | 86           | 27            | Shanghai          |
| China   | 86           | 28            | Sichuan           |
| China   | 86           | 29            | Taiwan            |
| China   | 86           | 30            | Tianjin           |
| China   | 86           | 31            | Tibet             |
| China   | 86           | 32            | Xinjiang          |
| China   | 86           | 33            | Yunnan            |
| China   | 86           | 34            | Zhejiang          |

> **Note:** The PDF lists “Shanxi” twice for codes 25 & 26—likely a typo. Confirm actual province codes (Shanxi 山西 vs Shaanxi 陕西) with AiSWEI if you intend to register stations in those provinces.

---

## 10. Best Practices & Tips

1. **Time Synchronization:** Ensure your server’s clock is accurate (±5 seconds). Signature verification fails if `X-Ca-Timestamp` drifts > 15 minutes from the server.
2. **Nonce Uniqueness:** Always generate a new UUID per request.
3. **Rate Limits:** Respect 100 requests/minute; implement retry/back-off on HTTP 429.
4. **Error Handling:**

   * Check `status != 200` in the JSON response for parameter or permission errors (`10001`, `10012`, `10403`, `20006`).
   * If HTTP status is ≥ 400, inspect `X-Ca-Error-Message` in response headers for signature issues.
5. **Pagination:** Many endpoints return `totalPages` and `pageNum`. Loop until `pageNum > totalPages`.
6. **Units & Scaling:** Many numeric fields are strings multiplied by a factor (e.g., temperature × 10, current × 100, energy × 10). Always convert to human-readable units (divide accordingly).

---

## 11. Change Log (from Original PDF)

* **v1.0 (2022-02-18)**: Initial release.
* **v1.1**: Added several new “Pro” endpoints; clarified request headers; updated signing method.

---

### End of Documentation

This consolidated reference should enable you to implement a robust AiSWEI Cloud API client in any language (Python examples shown above). If you find discrepancies between this doc and actual behavior, validate against the official AiSWEI support channels.

