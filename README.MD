# Content Loader Extended

Extension to improve how ThingWorx can consume external REST APIs, adding functionality missing from
the OOTB `Resources.ContentLoaderFunctions`.

## Usage

The extension exposes a new ThingWorx Resource called `ContentLoaderExtended`.

The core service this exposes is `ExecuteHttpRequest`. This service can be used to execute a request
of any type to a server. The HTTP verb can be specified using a service parameter.

Here are the parameters the service accepts:

| Name                | Description                                                                              | BaseType |
|:--------------------|:-----------------------------------------------------------------------------------------|:---------|
| url                 | URL to load                                                                              | STRING   |
| method              | HTTP method to use                                                                       | STRING   |
| content             | Payload that should be sent to the endpoint. Only some methods support setting a payload | STRING   |
| contentType         | ContentType of the payload to use. Must be specified if content is specified             | STRING   |
| username            | Optional user name credential                                                            | STRING   |
| password            | Optional password credential                                                             | STRING   |
| headers             | Optional HTTP headers                                                                    | JSON     |
| ignoreSSLErrors     | Ignore SSL Certificate Errors                                                            | BOOLEAN  |
| withCookies         | Include cookies in response                                                              | BOOLEAN  |
| withResponseStatus  | Include response status                                                                  | BOOLEAN  |
| withResponseHeaders | Include response headers                                                                 | BOOLEAN  |
| timeout             | Optional timeout in seconds                                                              | NUMBER   |
| useNTLM             | Use NTLM Authentication                                                                  | BOOLEAN  |
| workstation         | Auth workstation                                                                         | STRING   |
| domain              | Auth domain                                                                              | STRING   |
| useProxy            | Use Proxy server                                                                         | BOOLEAN  |
| proxyHost           | Proxy host                                                                               | STRING   |
| proxyPort           | Proxy port                                                                               | INTEGER  |
| proxyScheme         | Proxy scheme                                                                             | STRING   |
| keyStorePath        | Absolute path to the client SSL keystore                                                 | STRING   |
| keyStorePassword    | SSL Keystore Password                                                                    | STRING   |
| trustStorePath      | Absolute path to the SSL client truststore                                               | STRING   |
| trustStorePassword  | SSL Truststore Password                                                                  | STRING   |

Usage examples:

- Executing a GET request:

```javascript
let result = Resources.ContentLoaderExtended.ExecuteHttpRequest(
    {
      url: url,
      withResponseStatus: true,
      method: "GET",
      headers: {
        Prefer: "odata.maxpagesize=100",
        Accept: "application/json;odata.metadata=full",
      },
      ignoreSSLErrors: true,
      password: "test",
      username: "test"
    },
);
```

- Executing a PATCH request with body and mutual SSL auth:

```javascript
let result = Resources.ContentLoaderExtended.ExecuteHttpRequest(
    {
      url: url,
      withResponseStatus: true,
      method: "PATCH",
      contentType: "application/json",
      content: JSON.stringify({prop: "Value"}),
      headers: {
        ContentType: "application/json",
        CSRF_NONCE: me.csrfNonce,
      },
      ignoreSSLErrors: true,
      trustStorePassword: "***",
      keyStorePassword: "***",
      trustStorePath: "absoute path to file on disk",
      keyStorePath: "absoute path to file on disk "
    },
);
```

- Executing multiple requests in parallel

```javascript
// Execute multiple requests simultaneously by executing in a separate thread for each request.
// This approach is much faster than calling multiple 'ExecuteHttpRequest()' in a row,
// since each request blocks the execution of the script.
let result = Resources.ContentLoaderExtended.ExecuteHttpRequests(
    {
      array: [
        {
          url: url1,
          method: "GET"
        },
        {
          url: url2,
          method: "GET"
        },
        {
          url: "invalid ⚠️",
          method: "GET"
        }
      ]
    }
);

// Array of all results (in this example there are three result objects)
let allResults = result.array;

// Array of all (fulfilled) responses. Each object's structure is exactly the same as that of 'ExecuteHttpRequest'.
let responses = result.array.filter(e => !e.error);

// In case of an error ('ExecuteHttpRequest' would throw an error), an object with the property "error" of
// type "string" is returned, which describes the error. In this example, they are simply logged.
result.array.filter(e => e.error).forEach(e => logger.error(e.error));
```

- Sending a Multipart request with multiple files in the body:

```javascript
let files = DataShapes.MultipartFile_DS.CreateValues();
files.AddRow({
  multipartFileName: "Name of multipart file",
  repository: "ThingName of the FileRepository thing where the file is stored",
  pathOnRepository: "path to the file in the file repository",
});
let bodyParts = Resources.InfoTableFunctions.CreateInfoTable();
bodyParts.AddField({name: "PART_NAME", baseType: "STRING"});
bodyParts.AddField({name: "SECOND_PART_NAME", baseType: "STRING"});
bodyParts.AddRow({
  PART_NAME: "PART1_VALUE",
  SECOND_PART_NAME: "PART2_VALUE",
});
let result = Resources.ContentLoaderExtended.ExecuteHttpMultipartRequest(
    {
      url: url,
      method: "POST",
      filesToSend: multipartFiles,
      partsToSend: multipartParts,
      ignoreSSLErrors: true,
      password: "test",
      username: "test"
    }
);
```

## Development

The build uses the Gradle wrapper (Gradle 9) and needs a JDK 17 or newer to run.

### ThingWorx SDK

The ThingWorx SDK is proprietary to PTC and is **not** included in this repository. After cloning, you must
manually copy the following files into `lib/twxSdk`. They can be found in the ThingWorx installation
directory, in `/apache-tomcat/webapps/Thingworx/WEB-INF/lib`:

* `thingworx-common-<version>.jar`
* `thingworx-platform-common-<version>.jar`

The build is known to work with the 9.5 and 10.1 SDKs. The extension is compiled for Java 11, so it
still loads on older ThingWorx versions. Other libraries that ThingWorx already provides (HttpClient,
commons-io, org.json, slf4j, logback) are downloaded from Maven Central and are not bundled with the extension.

The GitLab pipeline downloads the SDK from the project's generic package registry instead, from the package
`thingworx-sdk` with the version set in `TWX_SDK_VERSION` in `.gitlab-ci.yml`. To use a different SDK version,
upload the jars once with a personal access token that has the `api` scope, then update `TWX_SDK_VERSION`:

```bash
for jar in thingworx-common thingworx-platform-common; do
  curl --fail --header "PRIVATE-TOKEN: <token>" --upload-file "lib/twxSdk/${jar}-9.5.0-b143.jar" \
    "https://gitlab.com/api/v4/projects/43130121/packages/generic/thingworx-sdk/9.5.0-b143/${jar}-9.5.0-b143.jar"
done
```

To share one SDK package between several extensions, upload it to a single project, set `TWX_SDK_PROJECT_ID`
to that project's id, and allow this project's CI job token to access it.

### Build

The folder layout can be modified by editing the `ext` section in the build.gradle file. The
default layout is as following:

```gradle
	uiDir = "$baseDir/ui" // UI widgets folder
	localJarDir = "$baseDir/lib/local" // Local jars folder, packaged with the extension
	srcDir = "$baseDir/src/main" // Source directory
	configDir = "$baseDir/configfiles" // Config files folder, including any Entities
	zipDir = "$baseDir/zip" // Output zip folder
	thingworxSdkDir = "$baseDir/lib/twxSdk" // Thingworx SDK folder
```

The version of the extension is read from `packageVersion` in `configfiles/metadata.xml`.

The thingworx related gradle tasks are:

* `packageExtension`: creates `zip/ContentLoaderExtended_Extension-v<version>.zip` with the extension
* `upload`: uploads the extension to the TWX server set in `thingworxServerRoot`
* `incrementExtensionVersion`: increments the patch version in `configfiles/metadata.xml`

### Releasing

The GitLab pipeline builds the extension on every push. To publish a release, push a tag named
`release/<version>`, where `<version>` matches `packageVersion` in `configfiles/metadata.xml`
(for example `release/2.2.0`). The pipeline then uploads the zip to the package registry and creates a
GitLab release.


## Unit testing framework

This repository contains now a test suite that can be executed to verify compatibility with the desired ThingWorx version. These tests are intended to be executed each time when testing a new ThingWorx version or when adding/changing functionalities.

### Testing framework reference

When adding a new end-user intended Service, a unit test should be added for that functionality.

The testing framework has the following characteristics:

- each test is defined as a ThingWorx service
- tests have access to the previous test results inside of them; they pass an empty `INFOTABLE` for this purpose, to which rows are added after each test execution
- there are pre- and post- services: `CreateTestData` and `DeleteTestData`; `CreateTestData` returns a JSON configuration field that is intended to be consumed by the tests themselves.
- all tests are executed via the `RunAllTests` (or via `RunAllTestsFormatResult` that returns the result as a `HTML` that can be saved as test result artefact).

#### How to run the tests:

1. Install [Mockoon](https://mockoon.com/). This is an open-source application that allows designing and running mock REST APIs. It offers desktop clients for Windows, Linux and Mac.A list of test endpoints has been created for the tests defined in this extension. 
2. Copy in clipboard all the content of the `testEntities/mockoon_test_endpoints.json`
3. In Mockoon, select File/New local environment from clipboard ![alt text](image.png)
4. Download the `testEntities/ContentLoaderExtended.Tests.xml` and Import / From File.
5. In Thingworx, navigate to `ContentLoaderExtended.Tests` Thing
6. Execute the service `RunAllTests:INFOTABLE`

#### How to **add a new test**:

1. **Navigate** to `ContentLoaderExtended.Tests` Thing
2. **Edit** the `CreateTestData` Service. Duplicate a test and add it at the end.

   Increment the test number and its associated variable names. Modify the `TestName`, `ServiceName`, `InputParams` and `ExpectedOutput` fields as needed for your test - all are mandatory.

   ```javascript
   //test 7
   let test7 = {
       TestName: "7. Basic Authenticated Post",
       ServiceName: "BasicAuthPost",
       InputParams: {
            URLEndpoint: str_HTTP+str_BaseURL+"TestPostServiceBasicAuth",
            User:"myUser",
            Password:"myPassword"
       },
       ExpectedOutput: {
            Response: "{\"MySampleKey\":\"MySampleValue\"}"
       }
   };
   result.tests.push(test7);
   ```

3. **Duplicate** one of the existing test services (eg: `BasicAuthGet`).

   Use for the new service the `ServiceName` chosen above.
   In your test implementation use the `InputParams` and `ExpectedOutput` defined above.
   Note that you have access to the `PreviousTestResults`, but typically you should use only the `InputParams` and `ExpectedOutputs`.

4. **Save** all your modifications and **execute** `RunAllTests`. All tests should pass.


# Disclaimer
By downloading this software, the user acknowledges that it is unsupported, not reviewed for security purposes, and that the user assumes all risk for running it.

Users accept all risk whatsoever regarding the security of the code they download.

This software is not an official PTC product and is not officially supported by PTC.

PTC is not responsible for any maintenance for this software.

PTC will not accept technical support cases logged related to this Software.

This source code is offered freely and AS IS without any warranty.

The author of this code cannot be held accountable for the well-functioning of it.

The author shared the code that worked at a specific moment in time using specific versions of PTC products at that time, without the intention to make the code compliant with past, current or future versions of those PTC products.

The author has not committed to maintain this code and he may not be bound to maintain or fix it.


# License
I accept the MIT License (https://opensource.org/licenses/MIT) and agree that any software downloaded/utilized will be in compliance with that Agreement. However, despite anything to the contrary in the License Agreement, I agree as follows:

I acknowledge that I am not entitled to support assistance with respect to the software, and PTC will have no obligation to maintain the software or provide bug fixes or security patches or new releases.

The software is provided “As Is” and with no warranty, indemnitees or guarantees whatsoever, and PTC will have no liability whatsoever with respect to the software, including with respect to any intellectual property infringement claims or security incidents or data loss.
