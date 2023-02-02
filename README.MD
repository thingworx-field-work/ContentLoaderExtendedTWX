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

After cloning the repository, please add the following files into `lib/common`. These files can be
obtained from the ThingWorx installation directory,
in `/apache-tomcat/webapps/Thingworx/WEB-INF/lib`.
The version numbers listed below should serve as just a recommendation, as the actual file versions
found in the ThingWorx installation directory might be different.

* `json-20171018.jar`
* `thingworx-common-9.2.3-b189.jar`
* `thingworx-platform-common-9.2.3-b189.jar`
* `joda-time-2.9.jar`
* `slf4j-api-1.7.12.jar`
* `logback-core-1.0.13.jar`
* `logback-classic-1.0.13.jar`

The folder layout can be modified by editing the project.ext section in the build.gradle file. The
default layout is as following:

```gradle
	uiDir = "/ui" // if there are any widgets
	localJarDir = "/lib/local" // if you have any local jars that need to be included in the project, add them here
	srcDir = "${baseDir}/src/main" // where are the sources located
	buildDir = "${baseDir}/build" // where is the build saved
	configDir = "${baseDir}/configfiles" // folder location of the metadata.xml file
	entitiesDir = "${baseDir}/configfiles/Entities" // folder location Entities that are included with the extension
	zipDir = "${baseDir}/zip" // where to store the generated zip 
	thingworxSdkDir = "${baseDir}/lib/common" // where the thingworx sdk is located
```

The thingworx related gradle tasks are:

* `packageExtension`: creates in the zipDir an zip archive with the extension
* `upload`: uploads the extension to the specified TWX server.

# Disclaimer

This Extension is provided as-is and without warranty or support. It is not part of the PTC product
suite. This project is licensed under the terms of the MIT license
