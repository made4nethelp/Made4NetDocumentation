# Create Loads Importer

## Overview

The **Create Loads Importer** is a reusable, a file-based (CSV) **SCExpert Connect** importer that creates inventory loads.
It validates incoming data (SKU, UOM, Location) against existing master data and creates loads.  
This is a **configuration-only install** per client (no code changes required). All behavior is controlled through **database configuration**.

- `Made4net.412.US.Shared` (Shared Project)
- Services: `Made4net.412.US.Shared.dll` + `Made4net.412.US.Shared.pdb`
- SQL install scripts stored under the shared project **Documentation** folder
- XSLT assets stored under the shared project **XSLTs** folder 

This is a **configuration-only install** per client (no code changes required). 


---

## Purpose

- Import inventory loads from inbound CSV files
- Validate incoming data (SKU, UOM, Location, etc.) against master data
- Create loads only when validations pass

---

## Flow

1. CSV file is dropped into the configured **ImportDirectory**
2. SCExpert Connect service detects the file based on:
    - **ImportDirectory**
    - **ImportFileNameFilter**
3. The configured XSLT (`CSVLoadImport.xslt`) translates CSV → XML format
4. Plugin runs **PREPROCESS validations** (SKU/UOM/Location + handling unit checks)
5. If validations pass:
    - Loads are created in SCExpert
    - Error is logged in Connect logs + SCT transaction tables
6. If validations fail:
    - File is moved to ErrorFilePath
    - Error is logged in Connect logs + SCT transaction tables
7. File is moved to SuccessFilePath or ErrorFilePath accordingly

---

## Scheduler Implementation Details

| Item | Value |
|----|----|
| DLL | `Made4net.412.US.Shared.dll` |
| Class | `Made4net._412.US.Shared.CreateLoads` |
| Translation File | `CSVLoadImport.xslt` |


---

## How the Load Importer Works

The Load Importer performs the following actions:

1. The Connect service checks the configured ImportDirectory for incoming files.
2. XSLT translation converts CSV → XML
3. Plugin preprocess validates data
4. Plugin creates loads using SCExpert Logic
6. Transaction behavior:
   - Success → commit and move file to SuccessFilePath
   - Failure → rollback and move file to ErrorFilePath
7. Logs are written to:

```text
C:\Program Files (x86)\Made4Net\SCExpert\Logs\Connect\
```

---

## Configuration Setup


### sys_Applications

This **configuration** will be created **once per environment**.

###Configuration Variables

```sql
DECLARE @dbname VARCHAR(100) = 'SCEGLB';
DECLARE @logpath VARCHAR(100) = 'C:\Program Files (x86)\Made4Net\SCExpert\Logs\Connect\';
DECLARE @plugintypeid = '62';--PluginTypeID specific for the create loads importer
DECLARE @pluginid = '59';--PluginID specific for the create loads importer
DECLARE @ImportCustomTranslationFile = 'C:\Program Files (x86)\Made4Net\SCExpert\SCExpertConnect\XSLT\CSVLoadImport.xslt'; --XSLT Path
DECLARE @ImportDirectory = 'C:\ALB\IMPORTS\'; --Import files from here
DECLARE @ImportFileNameFilter = '*_CreateLoadsW04_*.csv'; --Filter for files with this name. For multi warehouse - add warehouse name to the fileName and change for each plugin. For example in ALB, they have SCEALBW04 and SCEALBW09. So there's 2 plugins, one for each warehouse. FileNameFilter is '*_CreateLoadsW04_*.csv' and '*_CreateLoadsW09_*.csv' so each file can point to a different warehouse
DECLARE @InputFilesPath = 'C:\ALB\IMPORTS\'; --Another import path directory (Not sure what difference is but this and import direcrtory are both in base)
DECLARE @MoveOnFailureFolder = 'C:\ALB\IMPORTS\ERROR\'; --Where to move file on failure (Base param, match errorfilepath)
DECLARE @MoveOnSuccessFolder = 'C:\ALB\IMPORTS\PROC\'; --Where to move file on success (Base param, match SuccessFilePath)
DECLARE @SuccessFilePath = 'C:\ALB\IMPORTS\PROC\'; -- Base param, match to MoveOnSuccessFolder
DECLARE @ErrorFilePath = 'C:\ALB\IMPORTS\ERROR\'; --Move file here on error, match to MoveOnFailureFolder
DECLARE @PostProcess = 'N'; -- BaseParam, not relevant for Create Loads
DECLARE @DefaultHandlingUnitType = 'PICKCONT'; --Specific to create loads! If a load is in a container on import, and that container does not exist, what type of container should be created?
DECLARE @WaitComplete = 'Y'; --Base system param
DECLARE @PREPROCESS = 'Y'; --Enable pre processing (Important for create loads! Pre processing has some basic data validations before going into the file itself)
```

###Plugins
```sql
INSERT INTO [dbo].[SCEXPERTCONNECTPLUGINS] ([PLUGINID],[PLUGINDESCRIPTION],[PLUGINTYPEID],[PLUGINTRANSACTIONTYPE], [BOTYPE], [LOGPATH], [LOGLEVEL], [COMMITBOONBOELEMENTFAILURE], [ALERTPROCESSRESULT], 
				[ALERTRECIPIENT], [IMPORTTRANSACTIONCONTENTFILTER], [IMPORTTRANSLATIONFILE], [EXPORTTRANSACTIONCONTENTFILTER], [EXPORTTRANSLATIONFILE], [WAREHOUSEID], [ACTIVE], [ISALIVE], [LASTHEARTBEAT])
     VALUES(@pluginid, 'Loads Importer', @plugintypeid, 'IMPORT', 'LOADS', @logpath, '1', '0', 'None', NULL, , '', NULL, NULL, @dbname, '1', '0', NULL)
```


###Plugin Types
```sql
INSERT INTO [dbo].[SCEXPERTCONNECTPLUGINTYPES]
           ([PLUGINTYPEID], [PLUGINTYPE], [PLUGINNAME], [ASSEMBLYDLL], [CLASSNAME])
     VALUES (@plugintypeid, 'FILE', 'LoadImport', 'Made4net.412.US.Shared.dll', 'Made4net._412.US.Shared.CreateLoads')
```

###Plugin Params
```sql
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'CheckFileLock', '5')
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ErrorFilePath', @ErrorFilePath)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ImportCustomTranslationFile', @ImportCustomTranslationFile)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ImportDirectory', @ImportDirectory)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ImportEncoding', '')
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ImportFileNameFilter', @ImportFileNameFilter)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'ImportTimerInterval', '5')
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'InputFilesPath', @InputFilesPath)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'MoveOnFailureFolder', @MoveOnFailureFolder)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'MoveOnSuccessFolder', @MoveOnSuccessFolder)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'PostProcess', @PostProcess)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'SuccessFilePath', @SuccessFilePath)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'DefaultHandlingUnitType', @DefaultHandlingUnitType)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'WaitComplete', @WaitComplete)
INSERT INTO SCEXPERTCONNECTPLUGINPARAMS (PLUGINID, PARAMNAME, PARAMVALUE) VALUES (@pluginid, 'PREPROCESS', @PREPROCESS)
```

###Plugin Transaction Keys
```sql
INSERT INTO SCEXPERTCONNECTPLUGINTRANSACTIONKEYS (PLUGINID, TRANSACTIONKEY) VALUES (@pluginid, 'CONSIGNEE')
INSERT INTO SCEXPERTCONNECTPLUGINTRANSACTIONKEYS (PLUGINID, TRANSACTIONKEY) VALUES (@pluginid, 'LOADID')
```


### Note
-This is a shared, reusable importer (installable for any client using configuration)



