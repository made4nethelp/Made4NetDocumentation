# Shared Project Code Notes

This document gives a quick overview of some of the common classes in the shared project and what they are used for.

The shared project contains code that can be reused by different customer projects so we do not have to build the same import, export, API, or EDI logic again.

## FilesImporterPlugin

`FilesImporterPlugin` is used for file imports.

It checks a configured folder for files based on the file filter. Once it finds a file, it renames the file while it is being processed so the same file does not get picked up again.

The file is then read and translated into XML using the configured `ImportTranslationFile`. After that, the XML can go through `PreProcessFile()` and `ProcessFile()` before it is sent to SCExpert Connect.

The import runs on a timer. The interval comes from either:

- `ImportInerval`
- `ImportTimerInterval`

Some of the common parameters used by this plugin are:

- `FilesFilter`
- `ImportFileNameFilter`
- `InputFilesPath`
- `ImportDirectory`
- `ImportTranslationFile`
- `SuccessFilePath`
- `ErrorFilePath`
- `WaitComplete`
- `SyncFilter`

Once the XML is ready, it is sent to Connect using:

```csharp
NotifyImportProcessComplete(oTranslatedFiles);
```

If processing fails, the error is written to the log and a failed Connect transaction is posted.

The main methods that can be overridden for customer-specific logic are:

```csharp
PreProcessFile(...)
ProcessFile(...)
```

## FileExport

`FileExport` is used for outbound file integrations.

SCExpert Connect passes an XML document into the `Export()` method.

![FileExport plugin configuration](media/image1.png)

The plugin gets the output filename and then either writes the XML directly to the file or runs an XSLT before creating the file.

If `ExportCustomTranslationFile` is configured, the XML is translated before the file is written.

There is also a `PostProcess(...)` method. This can be overridden if we need to do anything after the outbound file has been created.

For example, a customer project could use it to update a database field or run some additional logic after the export.

## HttpExport

`HttpExport` is used when Made4net needs to send data to an external API.

![HttpExport plugin configuration](media/image2.png)

![HttpExport plugin parameters](media/image3.png)

The XML comes from SCExpert Connect and first goes through:

```csharp
DoPreProcess(...)
```

This is available so customer projects can modify the XML before the request is sent.

If `ExportCustomTranslationFile` is configured, the XML is also passed through the configured XSLT.

The request is normally sent as JSON.

The XML can be converted to JSON using:

```csharp
JsonConvert.SerializeXmlNode(...)
```

or through an XSLT.

If the plugin is configured with:

```text
ConvertToJSON = XSLT
```

then the `JSONXslt` file is used to create the JSON.

This is useful when the JSON format required by the external system is very different from the XML coming from Made4net.

### Authentication

`HttpExport` currently supports:

- OAuth 1.0
- Bearer Token
- OAuth 2.0
- Basic Authentication

The authentication method comes from the `AuthType` plugin parameter.

Depending on the authentication type, other parameters such as these may also be used:

- `ConsumerKey`
- `ConsumerSecret`
- `AccessToken`
- `TokenSecret`
- `UserName`
- `password`
- `AuthEndPoint`
- `Scope`
- `Grant_Type`

After the API request is sent, the response goes through:

```csharp
DoPostProcess(...)
```

By default, the response is treated as a failure if it is empty or contains `error` or `Failed`.

Customer projects can override `DoPostProcess()` if the external API has a different response format.

## HttpExportToXML

`HttpExportToXML` is very similar to `HttpExport`.

The main difference is that this class can send either XML or JSON.

![HttpExportToXML plugin configuration](media/image4.png)

The format is controlled by `OutputFormat`.

If:

```text
OutputFormat = XML
```

the request body is:

```csharp
xml.OuterXml
```

Otherwise, the plugin follows the normal JSON conversion logic.

So normally:

```text
HttpExport      -> JSON API exports
HttpExportToXML -> JSON or XML API exports
```

## HttpListenerPlugin

`HttpListenerPlugin` is used for inbound HTTP/API integrations.

![HttpListenerPlugin configuration](media/image5.png)

The plugin creates an HTTP listener using:

- `IP`
- `PORT`
- `PATH`

These values are used to build the listener URL.

When the plugin starts, it listens for incoming HTTP requests and processes the requests using worker threads.

The request body is first read as text and then passed through:

```csharp
handleInputData(...)
```

After that it is loaded into an `XmlDocument`.

The general flow is:

```text
External system
      |
HTTP request
      |
HttpListenerPlugin
      |
Read request
      |
Pre-process / validate
      |
SCExpert Connect
      |
WMS
      |
Return response
```

### handleInputData()

`handleInputData(...)` is used when we need to change the raw request before loading it into XML.

The base implementation just returns the same input.

A customer plugin can override this if it needs to convert another format into XML.

### GetDocumentData()

`GetDocumentData(...)` can be used when an API request is only retrieving information.

If this method returns data, the plugin can send the result back without continuing with the normal Connect import process.

### PreProcessDocument()

`PreProcessDocument(...)` is used when something needs to be changed before the XML is sent to Connect.

This is normally where customer-specific processing can be added.

For example:

- Add missing XML values
- Change a value
- Look up something from the database
- Route the request
- Run additional validation

### ValidateData()

`ValidateData()` does some common validation before the data goes into Connect.

For inbound and outbound orders, the shared code checks things such as:

- Consignee
- Company
- Contact
- SKU

If a required SKU does not exist, the request is stopped instead of sending invalid data into the WMS.

### ValidateLines()

`ValidateLines()` is used when an existing inbound or outbound order is being updated.

For outbound orders, the current logic allows the lines to be refreshed when the order is in an allowed status such as `NEW` or `WAVEASSIGNED`.

For inbound orders, the order needs to still be `NEW`.

The existing detail lines are removed before the incoming lines are processed again.

### PostProcessDocument()

`PostProcessDocument()` runs after Connect processing.

The shared code uses this method to update UDF values from the incoming XML.

It currently handles UDF updates for:

- Outbound Orders
- Inbound Orders
- SKU
- Flowthrough
- Receipts

For example, receipt UDF values are updated in:

- `RECEIPTHEADER`
- `RECEIPTDETAIL`

## HttpListenerPluginNonXML

`HttpListenerPluginNonXML` inherits from `HttpListenerPlugin`.

It is used when the incoming request is not already XML.

![HttpListenerPluginNonXML configuration](media/image6.png)

In the current code, this is mainly used for CSV imports.

The plugin reads `CSVTranslationFile` and uses that translation file to convert the incoming CSV into XML.

After the CSV has been converted, the normal `HttpListenerPlugin` processing continues.

```text
CSV request
     |
CSVTranslationFile
     |
XML
     |
HttpListenerPlugin
     |
SCExpert Connect
```

This lets us reuse the normal HTTP listener code without changing the rest of the import process.

## SCERestApiLandingPage

`SCERestApiLandingPage` is used as a routing point for REST API requests.

Instead of the external system needing to know the URL for every individual plugin, the request can come into the landing page first.

The landing page reads the warehouse from the incoming XML and uses the BO type to find the correct destination plugin.

The routing information comes from `RestApiPluginRegistry`.

![SCERestApiLandingPage plugin configuration](media/image7.png)

![SCERestApiLandingPage plugin type](media/image8.png)

The code uses information such as:

- Warehouse
- BO Type
- IP
- Port
- URL Path
- HTTP Method

![RestApiPluginRegistry entries](media/image9.png)

Once the destination plugin is found, the landing page forwards the original XML to that plugin.

The flow is roughly:

```text
External system
      |
SCERestApiLandingPage
      |
Read warehouse / BO type
      |
RestApiPluginRegistry
      |
Find destination plugin
      |
Forward request
      |
Receive result
      |
Return response
```

If more than one matching plugin is returned, the transaction results are combined into one response.

## Quick Reference

| Class | Used for |
|---|---|
| `FilesImporterPlugin` | Importing files from a folder into Made4net |
| `FileExport` | Creating outbound files |
| `HttpListenerPlugin` | Incoming API requests |
| `HttpListenerPluginNonXML` | Incoming requests such as CSV that need to be converted into XML first |
| `HttpExport` | Sending data to an external API, normally as JSON |
| `HttpExportToXML` | API exports that can send either XML or JSON |
| `SCERestApiLandingPage` | Routing an API request to the correct plugin |
