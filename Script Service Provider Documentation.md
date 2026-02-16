<a id="top"></a>

# Table of Contents

- [Table of Contents](#table-of-contents)
- [1. Script Service Provider Artifact](#1-script-service-provider-artifact)
  - [1.1 Script Record](#11-script-record)
    - [1.1.1 Script Configuration](#111-script-configuration)
      - [Script Input](#script-input)
      - [Script Output](#script-output)
  - [1.2 Service Provider Record](#12-service-provider-record)
    - [1.2.1 Worker Queue Options](#121-worker-queue-options)
      - [Priority Queue Threshold](#priority-queue-threshold)
    - [1.2.2 AuthSettings](#122-authsettings)
    - [1.2.3 Services](#123-services)
      - [Service Inputs](#service-inputs)
        - [Input Presenter Configuration](#input-presenter-configuration)
          - [String Type](#string-type)
          - [Expression Type](#expression-type)
          - [Number Type](#number-type)
          - [Boolean Type](#boolean-type)
          - [DateTime Type](#datetime-type)
          - [ParameterSet Type](#parameterset-type)
      - [Service Outputs](#service-outputs)
- [2. Enums](#2-enums)
  - [InputScope](#inputscope)
  - [InputType](#inputtype)
  - [PresenterType](#presentertype)
  - [OutputType](#outputtype)
  - [DateTimePickerControls](#datetimepickercontrols)
- [3. Example](#3-example)
  - [3.1 Setting Default Value Using ParameterAssignment](#31-setting-default-value-using-parameterassignment)
  - [3.2 Script Example](#32-script-example)
    - [3.2.1 AuthSettings and getOAuthToken Usage](#321-authsettings-and-getoauthtoken-usage)
      - [AuthSettings Configuration](#authsettings-configuration)
      - [Using getOAuthToken in Scripts](#using-getoauthtoken-in-scripts)
    - [3.2.2 fetch Usage](#322-fetch-usage)
      - [Key Differences from Native JavaScript fetch](#key-differences-from-native-javascript-fetch)
      - [Using fetch in Scripts](#using-fetch-in-scripts)
      - [Fetch Function Signature](#fetch-function-signature)
      - [Response Handling Patterns](#response-handling-patterns)
      - [Important Notes](#important-notes)

# 1. Script Service Provider Artifact 

The Script service provider artifact contains all configuration required for Connect to integrate with REST services through custom JavaScript modules. Create a new service provider for each set of scripts. Each provider can define multiple services that share scripts and UI configuration.

The artifact can be imported and exported from the Connect admin area in the Profisee Portal.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| ArtifactVersion | `string` | The version of the artifact. Example: `1.5.1.0` |
| Scripts | List<[Script](#11-script-record)> | A list of script records containing JavaScript code to be executed. Each script must be referenced by at least one service in the service provider record. |
| ServiceProviderRecord | [ServiceProviderRecord](#12-service-provider-record) | Defining service inputs and outputs |

## 1.1 Script Record 
<a href="#top">[top]</a>

A script record contains JavaScript code executed by Connect. Each script must export an `execute` function that receives input parameters defined in the service configuration.

The `Name` is used throughout the administration experience to reference the script. Each script must be referenced by its `Id` in at least one service's `scriptId` field.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Id | `string` | A unique identifier (GUID) for the record. This Id must be referenced in the service's `scriptId` field. |
| Name | `string` | The name of the script. |
| Configuration | [ScriptConfiguration](#111-script-configuration) | The script configuration containing the JavaScript code. |

### 1.1.1 Script Configuration 
<a href="#top">[top]</a>

The script configuration contains the JavaScript code to be executed.

The `scriptContent` field contains the complete JavaScript code as a string. The script must export an `execute` function that accepts an `input` parameter. The input parameter contains all inputs defined in the service configuration, organized by scope (Service and Data).

| Name       | Type       | Description          |
|------------|------------|----------------------|
| scriptContent | `string` | The complete JavaScript code to be executed as a string. The script must export an `execute` function. |

The built-in `fetch` and `getOAuthToken` functions are available to import from the `"Profisee"` module to simplify HTTP requests and OAuth authentication. For detailed examples and usage instructions, see [3.2.1 AuthSettings and getOAuthToken Usage](#321-authsettings-and-getoauthtoken-usage) and [3.2.2 fetch Usage](#322-fetch-usage).

Example script content:
``` javascript
// optional, see example at section 3 Example
import { fetch, getOAuthToken } from "Profisee"; 

export function execute(input) {
    // write your code here
}
```

#### Script Input 

The `input` parameter passed to the JavaScript `execute` function contains all the data needed for script execution. The input structure is automatically constructed based on the [Service Inputs](#service-inputs) configuration and consists of three main components:

| Name       | Type       | Description          |
|------------|------------|----------------------|
| `AuthSettings` | `object` | An object containing authentication settings that are configured in the [AuthSettings Configuration](#122-authsettings). In the script, users can access the settings by `input.authSettings`, then use these settings to retrieve a token by using `getOAuthToken`. See [AuthSettings and getOAuthToken Usage](#321-authsettings-and-getoauthtoken-usage) for more information. |
| Service Scope Inputs | `any` | All inputs defined with `Service` scope in the service configuration are included as top-level properties in the input object. These inputs are evaluated once per operation and are available to all records processed in the batch. Each service scope input is accessible by its input key name. |
| `Records` | `array` | An array of record objects, where each object contains the values from inputs defined with `Data` scope. Each record in the array represents one data record being processed, and the properties within each record object correspond to the Data scope input keys defined in the service configuration. |

Example input structure:

``` json
"input": {
  "AuthSettings": {},
  "SCStringInput1": "hello",
  "SCNumberInput2": 34,
  "Records": [
    {
      "input1": "CompanyXXX",
      "input2": 13.4,
      "input3": "2/5/2024"
    },
    {
      "input1": "CompanyYYY",
      "input2": 18.4,
      "input3": "4/6/2023"
    }
  ]
}
```

In this example, `SCStringInput1` and `SCNumberInput2` are Service scope inputs, while `input1`, `input2`, and `input3` are Data scope inputs that appear within each record in the `Records` array.

#### Script Output 

The `execute` function must return an object that defines the execution results and processed records. The output structure is used to determine success status, retry behavior, error handling, and how processed records are mapped back to entity records based on the [Service Outputs](#service-outputs) configuration.

The output object can include the following properties:

| Name       | Type       | Description          |
|------------|------------|----------------------|
| `IsSuccess` | `boolean` (optional) | A boolean value indicating whether the script execution was successful. If this property is not returned, it defaults to `true`. Set this to `false` to indicate that the operation failed. |
| `ShouldRetry` | `boolean` (optional) | A boolean value indicating whether the script execution should be retried. If this property is not defined, it defaults to `false`. Set this to `true` when the script encounters a transient error that may succeed on retry. |
| `Error` | `string` (optional) | A string value containing any error message or description of issues encountered during script execution. This is typically used in conjunction with `IsSuccess: false` to provide diagnostic information. |
| `Records` | `array` | An array of record objects containing the processed data to be mapped back to entity records. Each object in the array should contain properties that correspond to the output keys defined in the service's outputs dictionary. The mapping of these output properties back to entity fields is configured in the Integration Strategy response mappings. |

Example output structure:

``` json
{
  "IsSuccess": true,
  "ShouldRetry": false,
  "Error": "",
  "Records": [
    { 
      "Output1": "AddressXXX", 
      "Output2": 32.4,
      "Output3": "11/13/2025",
      "Output5": false
    },
    { 
      "Output1": "AddressYYY", 
      "Output2": 15.7,
      "Output3": "11/07/2025",
      "Output5": true
    }
  ]
}
```

In this example, `Output1`, `Output2`, `Output3`, and `Output5` correspond to output keys defined in the service's outputs dictionary. The values in each record object will be mapped back to entity records according to the Integration Strategy response mapping configuration.


## 1.2 Service Provider Record 
<a href="#top">[top]</a>

The service provider record represents the provider in the Profisee Portal and contains the configuration object.

The `Name` is used throughout the administration experience to reference the provider.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| Id | `string` | A unique identifier (GUID) for the record. |
| Name | `string` | The name of the service provider. |
| Type | `string` | Static value, must be set to: `Script`. |
| Configuration | `ServiceProviderConfiguration` | The service provider configuration, consists of [WorkerQueueOptions](#121-worker-queue-options), [AuthSettings](#122-authsettings) (optional), and Dictionary<string, [Service](#123-services)>. |

### 1.2.1 Worker Queue Options 
<a href="#top">[top]</a>

Worker queue options configure how script executions are processed. These settings control the number of worker threads and priority queue thresholds for batch processing.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| workerCount | `int` | The number of worker threads that process script executions. This determines how many batches can run concurrently. |
| priorityQueueThreshold | [PriorityQueueThreshold](#priority-queue-threshold) | Configuration for priority queue thresholds. Batches meeting these thresholds are queued as high priority because they process faster. |

#### Priority Queue Threshold 

Priority queue thresholds determine which batches are processed with high priority. Batches with job counts or record counts below these thresholds are queued as high priority because they process faster.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| maxJobCount | `int` | Batches with a job count below this threshold are queued as high priority. |
| maxRecordCount | `int` | Batches with a record count below this threshold are queued as high priority. |

Example worker queue options configuration:

``` json
{
  "workerQueueOptions": {
    "workerCount": 2,
    "priorityQueueThreshold": {
      "maxJobCount": 1,
      "maxRecordCount": 5000
    }
  }
}
```
### 1.2.2 AuthSettings 
<a href="#top">[top]</a>
This is an optional field. It provides authentication settings that can be used in the script with `getOAuthToken`. If `authSettings` is provided in the `ServiceProviderRecord`, the service configuration will display an "Authentication" section asking for more credential information. See [AuthSettings and getOAuthToken Usage](#321-authsettings-and-getoauthtoken-usage) for more information.
| Name       | Type       | Description          |
|------------|------------|----------------------|
| Type| `string`| Determines the Auth type. Currently only "OAuth2" is supported. This field is required. |
| OAuthUrl|`string`| Provides the OAuth URL. This field can be empty. If you choose to leave it empty, OAuth Scope or OAuth URL will be asked later in service configuration.|
| Scope   | `string` | Optional field. The OAuth scope value as required by the token service. This can be any string value specified by the OAuth provider. |

Example of AuthSettings configuration:
``` json
"authSettings": {
  "Type": "OAuth2",
  "OAuthUrl": "https://oauthUrl.com/v1/token"
}
```

### 1.2.3 Services 
<a href="#top">[top]</a>

Services define executable operations that can be selected during service configuration, only one service can be selected for each service configuration. Each service references a script and defines the inputs and outputs for that script execution.

The dictionary key is an identifier for the service and is not displayed in the portal.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| name | `string` | The display name shown in the Profisee Portal. |
| description | `string` | The description shown in the Profisee Portal. |
| scriptId | `string` | The Id of the script record to execute. Must match the `Id` field of a [Script](#11-script-record) defined in the `Scripts` array. |
| maximumRecordsPerRequest | `int` | The maximum number of records per request that the service supports. |
| inputs | Dictionary<string, [Input](#service-inputs)> | A dictionary of input definitions. The key is the input name available in the script's `execute` function. |
| outputs | Dictionary<string, [Output](#service-outputs)> | A dictionary of output definitions. The key is the output name returned from the script's `execute` function. |

#### Service Inputs 

Inputs define parameters passed to the script's `execute` function. Inputs can be defined with either `Service` or `Data` scope, which determines where they are configured and when they are evaluated.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| scope | [InputScope](#inputscope) | The input scope. `Service` scope inputs are configured in Service Configuration and evaluated once per operation (i.e. strategy invocation). `Data` scope inputs are available in Integration Strategy request mappings and evaluated per record. |
| type | [InputType](#inputtype) | The data type of the input value. Available types: `Date`, `Float`, `String`, `Integer`, `Boolean`. Typically use `String` for most inputs. Change to other types only when you need to specify the data type for Expression presenter type inputs. |
| friendlyName | `string` | The display name shown in the Profisee Portal. This is the input label displayed to users. |
| isRequired | `bool` | When `true`, the input displays an asterisk in the portal, indicating it is mandatory and non-null. Default is `false`. |
| isReadOnly | `bool` | When `true`, the input is not editable in the portal. When `true`, you can provide a default value. Only works with `Service` scope inputs. `Data` scope inputs always treat this as `false`. `isReadOnly` defaults to `false`.|
| defaultValue | `ParameterAssignment` | An optional default value for the input. Typically used when `isReadOnly` is `true` for Service scope inputs. See example of [setting sefault value using ParameterAssignment](#31-setting-default-value-using-parameterassignment) |
| presenterConfiguration | [PresenterConfiguration](#input-presenter-configuration) | Configuration that defines how the input is displayed in the portal, including the UI control type used in request mapping strategy. |

##### Input Presenter Configuration 

Presenter configuration defines how the input is exposed for editing in the Profisee Portal. The `type` determines the UI control type used to present the input. The `settings` object contains additional configuration for the presenter based on the type.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| type | [PresenterType](#presentertype) | The presenter control type. This defines the editor type used in request mapping strategy. Options: `String`, `Expression`, `Number`, `Boolean`, `DateTime`, `ParameterSet`. |
| settings | [JObject](https://www.newtonsoft.com/json/help/html/T_Newtonsoft_Json_Linq_JObject.htm) | Settings for the input presenter. Settings vary based on the presenter type. |

###### String Type 


The String type displays as a text field where users can enter text. User input values are treated as strings. No additional settings are required.

Example:

``` json
{
  "type": "String",
  "settings": {}
}
```

###### Expression Type 

The Expression type provides a dropdown or an expression editor button that opens the expression editor window. User input values are treated according to the input's root type (as defined in the input's `type` property). No additional settings are required.

Example:

``` json
{
  "type": "Expression",
  "settings": {}
}
```

######  Number Type

The Number type displays as a numeric field. User input values are treated as numbers. Use the settings to define optional minimum and maximum values and the number of decimal places.

| Name          | Type    | Description |
|---------------|---------|-------------|
| DecimalPlaces | `uint`  | The maximum number of digits to the right of the decimal point. Default is `0`. |
| MinimumValue  | `double` | The optional minimum allowed value. |
| MaximumValue  | `double` | The optional maximum allowed value. |

Example:

``` json
{
  "type": "Number",
  "settings": {
    "DecimalPlaces": 3,
    "MinimumValue": 10,
    "MaximumValue": 20.500
  }
}
```

######  Boolean Type

The Boolean type displays as a checkbox. User input values are treated as booleans. No additional settings are required.

Example:

``` json
{
  "type": "Boolean",
  "settings": {}
}
```

###### Credential Type

The Credential type presents as a password control that the user fills in. No additional settings are required.

Due to a bug in the current version of Profisee, credential values must be retrieved via the `value` property on the setting, like `input.password.configuration.value`. This will be fixed in 26.1.

Example:

``` json
{
  "type": "Credential",
  "settings": {}
}
```

######  DateTime Type

The DateTime type displays as a date/time picker where users can select a date and time, or date only. User input values are treated as DateTime values. With empty settings, it defaults to a date and time picker. Setting `"Controls": "Date"` changes it to a date-only picker.

| Name     | Type                      | Description |
|----------|---------------------------|-------------|
| Controls | [DateTimePickerControls](#datetimepickercontrols) | Specifies the controls for the DateTimePicker. Default is `DateAndTime` when settings are empty. Setting to `"Date"` changes it to a date-only picker. |

Example with date picker:

``` json
{
  "type": "DateTime",
  "settings": {
    "Controls": "Date"
  }
}
```

Example with date and time picker (default):

``` json
{
  "type": "DateTime",
  "settings": {}
}
```

######  ParameterSet Type

The `ParameterSet` type is only available for `Service` scope inputs. The `Parameters` dictionary maps parameter names within the ParameterSet to input keys defined in the service's inputs dictionary. These grouped inputs appear together in the same section of the web portal, and each input value is passed individually to the script.

| Name       | Type                           | Description |
|------------|--------------------------------|-------------|
| Parameters | Dictionary<string, string>     | A dictionary mapping parameter names within the ParameterSet to input keys defined in the service's inputs. |

Example:

``` json
{
  "type": "ParameterSet",
  "settings": {
    "Parameters": {
      "SCInput1": "SCStringInput1",
      "SCInput2": "SCNumberInput2",
      "SCInput3": "SCReadOnlyInput3"
    }
  }
}
```

In this example, `SCStringInput1`, `SCNumberInput2`, and `SCReadOnlyInput3` are keys that reference inputs defined in the service's inputs dictionary. These inputs will show in the same section in the service configuration.


#### Service Outputs 

Outputs define the response structure returned from the JavaScript script execution. Outputs are used in Integration Strategy response mappings to map data back to Profisee.

| Name       | Type       | Description          |
|------------|------------|----------------------|
| name | `string` | The label shown in the portal strategy mapping. |
| type | [OutputType](#outputtype) | The JTokenType of the output value. Can be: `Integer`, `Float`, `String`, `Boolean`, `Date`. |

The script's `execute` function must return an object with keys matching the output keys defined here. For example:
``` json
{
  "outputs": {
    "Output1": { "name": "Output 1", "type": "String" },
    "Output2": { "name": "Output 2", "type": "Float" },
    "Output3": { "name": "Output 3", "type": "Date" },
    "Output4": { "name": "Output 4", "type": "Integer" },
    "Output5": { "name": "Output 5", "type": "Boolean" }
  }
}
```

---

# 2. Enums 
<a href="#top">[top]</a>

## InputScope 

The input scope determines where it is configured and when it is evaluated.

| Name       | Description          |
|------------|----------------------|
| Service    | Service scope inputs are configurable in Service Configuration and evaluated once per operation. These inputs are available to all records processed in a batch. The `isReadOnly` property works for Service scope inputs. |
| Data       | Data scope inputs are available in Integration Strategy request mappings and evaluated per record. Use this to access data from individual records being processed. The `isReadOnly` property is always treated as `false` for Data scope inputs. |

## InputType 

The data type of an input value. Available types: `Date`, `Float`, `String`, `Integer`, `Boolean`.

| Name       | Description          |
|------------|----------------------|
| String     | String data type. |
| Number     | Number data type. |
| Float      | Float data type. |
| Integer    | Integer data type. |
| Boolean    | Boolean data type. |
| Date       | Date data type. |

## PresenterType 

The UI control type used to present the input in the Profisee Portal. This defines the editor type used in request mapping strategy.

| Name           | Description                |
|----------------|----------------------------|
| [String](#string-type)         | String input control. Displays as a text field where users can enter text. |
| [Number](#number-type)         | Number input control. Displays as a numeric field. |
| [Boolean](#boolean-type)        | Boolean input control. Displays as a checkbox. |
| [DateTime](#datetime-type)       | DateTime input control. Displays as a date/time picker where users can select a date and time, or date only. |
| [Expression](#expression-type)     | Expression input control. Provides a dropdown or an expression editor button that opens the expression editor window. |
| [ParameterSet](#parameterset-type)   | Parameter set input control that groups multiple inputs together. Only works with `Service` scope. |
| [Credential](#credential-type)   | Credential input control. Provides a credential input window. Only works with `Service` scope. | -->

## OutputType 

The JTokenType of the output value returned from script execution. Available types: `Integer`, `Float`, `String`, `Boolean`, `Date`.

| Name       | Description          |
|------------|----------------------|
| Integer    | Integer number type. |
| Float      | Floating point number type. |
| String     | String type. |
| Boolean    | Boolean type. |
| Date       | Date type. |

## DateTimePickerControls 

The date/time picker control type to use.

| Name         | Description |
|--------------|-------------|
| Date         | Provides a date picker for selecting a date with input fields for day, month, and year. |
| DateAndTime  | Provides a date/time picker for selecting a date and time with input fields for day, month, year, hour, and minute. This is the default when settings are empty. |

---

# 3. Example

## 3.1 Setting Default Value Using ParameterAssignment 
<a href="#top">[top]</a>

Example of ParameterAssignment for setting the default string value. Note: the "dataType" value needs to be a JTokenType (i.e. Integer, Float, String ...)

``` json
 "defaultValue": {
    "name": "InputDefaultString",
    "dataType": "string",
    "assignmentType": "Literal",
    "value": "Your default value"
  }
```

Example of ParameterAssignment for setting the default numeric value.

``` json
 "defaultValue": {
    "name": "InputDefaultNumber",
    "dataType":"integer",
    "assignmentType": "Literal",
    "value": 34
  }
```
## 3.2 Script Example
<a href="#top">[top]</a>

This section provides practical examples using the D&B Identity Resolution service provider artifact to demonstrate the usage of script and it's built-in functions.

### 3.2.1 AuthSettings and getOAuthToken Usage

The `AuthSettings` configuration in the service provider artifact defines the authentication method and endpoint, while the `getOAuthToken` function in the script retrieves the authentication token using these settings. Together, they enable secure API access without hardcoding credentials in the script.

#### AuthSettings Configuration

The `authSettings` object is defined at the service provider level in the artifact configuration. This configuration determines what authentication information will be requested from users during service configuration setup.

**Example from D&B Identity Resolution artifact:**

``` json
"authSettings": {
  "Type": "OAuth2",
  "OAuthUrl": "https://plus.dnb.com/v3/token"
}
```

In this example:
- **`Type`**: Specifies the authentication type. Currently, only `"OAuth2"` is supported.
- **`OAuthUrl`**: The OAuth token endpoint URL. If provided, this URL is used for token requests. If left empty, users will be prompted to provide the OAuth URL during service configuration.

When `authSettings` is configured in the service provider, Connect automatically displays an "Authentication" section in the service configuration UI, prompting users to enter their OAuth credentials (such as Client ID and Client Secret).

#### Using getOAuthToken in Scripts

The `getOAuthToken` function is imported from the `"Profisee"` module and is used within the script's `execute` function to retrieve an OAuth token. The function takes `input.authSettings` as a parameter, which contains the authentication settings configured by the user.

**Example from IdentityResolution.js:**

``` javascript
import { fetch, getOAuthToken } from "Profisee";

export function execute(input) {
  // Retrieve OAuth token using the authSettings from input
  const oAuthResponse = getOAuthToken(input.authSettings);
  
  // Check if token retrieval was successful
  if (!oAuthResponse.ok) {
    return {
      isSuccess: false,
      error: oAuthResponse.error,
      shouldRetry: false,
    };
  }

  // Use the token in API requests
  const fetchResponse = fetch(`${url}?${queryString}`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${oAuthResponse.token}`,
      Accept: "application/json",
    },
  });
  
  // ... rest of the script logic
}
```

**How it works:**

1. **Token Retrieval**: `getOAuthToken(input.authSettings)` is called at the beginning of the `execute` function. The `input.authSettings` object contains the authentication configuration, including the OAuth URL and user-provided credentials.

2. **Response Handling**: The function returns an object with the following properties:
   - **`ok`**: A boolean indicating whether the token retrieval was successful.
   - **`token`**: The OAuth access token (only present when `ok` is `true`).
   - **`error`**: An error message string (only present when `ok` is `false`).

3. **Error Handling**: If token retrieval fails (`oAuthResponse.ok` is `false`), the script should return an error response immediately, as shown in the example. This prevents unnecessary API calls when authentication cannot be established.

4. **Token Usage**: When successful, the token is included in API request headers using the `Authorization: Bearer {token}` format. The token is automatically managed by Connect, including token refresh when necessary.

**Key Points:**

- The `authSettings` object is automatically populated in the `input` parameter based on the service provider configuration and user-provided credentials.
- The `getOAuthToken` function handles the OAuth2 flow, including token requests and refresh logic, so scripts don't need to implement OAuth flows manually.
- Always check `oAuthResponse.ok` before using the token to handle authentication failures gracefully.
- The token should be included in the `Authorization` header of subsequent API requests to the external service.


### 3.2.2 fetch Usage

The `fetch` function is imported from the `"Profisee"` module and provides a simplified, synchronous API for making HTTP requests to external services. Unlike the native JavaScript `fetch` API, Profisee's `fetch` is synchronous and returns a response object directly, making it easier to use within script execution contexts.

#### Key Differences from Native JavaScript fetch

**Native JavaScript `fetch`:**
- Returns a `Promise` that resolves to a `Response` object
- Requires `await` or `.then()` to handle the asynchronous response
- Response body must be read using methods like `.json()`, `.text()`, etc.
- Network errors throw exceptions that must be caught

**Profisee's `fetch`:**
- **Synchronous execution**: Returns the response object immediately (no Promise)
- **Direct access to response body**: The response body is available as a string property
- **Built-in error handling**: Network errors are captured in the response object rather than throwing exceptions
- **Simplified API**: No need for async/await or Promise chains

#### Using fetch in Scripts

**Example from IdentityResolution.js:**

``` javascript
import { fetch, getOAuthToken } from "Profisee";

export function execute(input) {
  // Get OAuth token first
  const oAuthResponse = getOAuthToken(input.authSettings);
  if (!oAuthResponse.ok) {
    return {
      isSuccess: false,
      error: oAuthResponse.error,
      shouldRetry: false,
    };
  }

  // Build query string from input data
  const queryString = getQueryString({
    ...input.records[0],
    candidateMaximumQuantity: 3,
  });

  // Make synchronous fetch request
  const fetchResponse = fetch(`${url}?${queryString}`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${oAuthResponse.token}`,
      Accept: "application/json",
    },
  });

  // Check response status
  if (!fetchResponse.ok) {
    // Handle exceptions (network errors, timeouts, etc.)
    if (fetchResponse.exceptionType) {
      return {
        isSuccess: false,
        error: fetchResponse.error,
        shouldRetry: fetchResponse.shouldRetry,
      };
    }

    // Handle HTTP error responses
    const dnbError = tryGetDnbError(fetchResponse);
    const shouldRetry = retryErrorCodes.includes(fetchResponse.status);

    return {
      isSuccess: false,
      error: dnbError,
      shouldRetry: shouldRetry,
    };
  }

  // Parse successful response
  try {
    const data = JSON.parse(fetchResponse.body);
    const record = getRecord(data);

    return {
      isSuccess: true,
      records: [record],
    };
  } catch (error) {
    return {
      isSuccess: false,
      error: "Failed to parse response: " + error.message,
      shouldRetry: false,
    };
  }
}
```

#### Fetch Function Signature

``` javascript
fetch(url, options)
```

**Parameters:**
- **`url`** (required): A string containing the URL to request
- **`options`** (optional): An object containing request configuration:
  - **`method`**: HTTP method (e.g., `"GET"`, `"POST"`, `"PUT"`, `"DELETE"`). Defaults to `"GET"`.
  - **`headers`**: An object containing HTTP headers to include in the request
  - **`body`**: Request body as a string (typically used for POST/PUT requests)

**Returns:** A response object with the following properties:

- **`ok`**: A boolean indicating whether the request was successful (status code 200-299)
- **`status`**: The HTTP status code (e.g., 200, 404, 500)
- **`body`**: The response body as a string (ready to parse with `JSON.parse()` if JSON)
- **`exceptionType`**: Present only when a network exception occurred (e.g., timeout, connection failure)
- **`error`**: Error message (present when `exceptionType` exists or when request failed)
- **`shouldRetry`**: A boolean indicating whether the request should be retried (typically set for transient errors)

#### Response Handling Patterns

**1. Successful Response:**
``` javascript
const response = fetch(url, options);
if (response.ok) {
  const data = JSON.parse(response.body);
  // Process data
}
```

**2. HTTP Error Response:**
``` javascript
const response = fetch(url, options);
if (!response.ok) {
  // Check for retryable errors
  const shouldRetry = [408, 429, 500, 502, 503, 504].includes(response.status);
  return {
    isSuccess: false,
    error: `HTTP ${response.status}: ${response.body}`,
    shouldRetry: shouldRetry
  };
}
```

**3. Network Exception:**
``` javascript
const response = fetch(url, options);
if (response.exceptionType) {
  // Network error occurred (timeout, connection failure, etc.)
  return {
    isSuccess: false,
    error: response.error,
    shouldRetry: response.shouldRetry
  };
}
```

**4. POST Request Example:**
``` javascript
const response = fetch("https://api.example.com/data", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "Authorization": `Bearer ${token}`
  },
  body: JSON.stringify({
    key: "value",
    number: 123
  })
});
```

#### Important Notes

- **Synchronous Execution**: Profisee's `fetch` executes synchronously, so you don't need `await` or Promise handling. The function blocks until the request completes.

- **Response Body**: The `body` property contains the raw response as a string. For JSON responses, use `JSON.parse(response.body)` to convert it to an object.

- **Error Handling**: Always check `response.ok` to determine if the request was successful. Additionally, check for `response.exceptionType` to handle network-level errors separately from HTTP errors.

- **Retry Logic**: The `shouldRetry` property can help implement retry logic for transient errors. Common retryable status codes include 408 (Request Timeout), 429 (Too Many Requests), and 5xx server errors.

- **Headers**: Header names are case-sensitive. Common headers include `Authorization`, `Content-Type`, `Accept`, etc.

- **No CORS Issues**: Since the fetch is executed server-side by Connect, there are no CORS (Cross-Origin Resource Sharing) restrictions.

