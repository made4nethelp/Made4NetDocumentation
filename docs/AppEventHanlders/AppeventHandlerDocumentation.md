# Creating and Installing a Made4net AppEventHandler

*Simple step-by-step guide*

This is the basic process we followed for the BES AppEventHandler. The service listens to the AppEventHandler MSMQ queue, receives Made4net events, and runs our custom code based on the event number.

## Step 1 - Create the project

1. In Visual Studio, create a C# Windows Service (.NET Framework) project.
2. Select .NET Framework 4.8.
3. Add references to the required Made4net DLLs and to `System.Configuration`, `System.Messaging`, `System.ServiceProcess`, and `System.Configuration.Install`.

## Step 2 - Add the handler

Create a `Handler` class that inherits from `Made4Net.Shared.QHandler`. `AppEventHandler` is the queue name.

```csharp
public partial class Handler : Made4Net.Shared.QHandler
{
    public Handler() : base("AppEventHandler", false)
    {
        LoggingEnabled = Convert.ToInt32(
            ConfigurationManager.AppSettings["UseLogs"]);
        LogDirectory = ConfigurationManager.AppSettings["LogPath"];
    }

    protected override void ProcessQueue(Message qMsg, QMsgSender qSender,
        PeekCompletedEventArgs e)
    {
        int eventId;
        if (!Int32.TryParse(Convert.ToString(qSender.Values["EVENT"]), out eventId))
            return;

        switch (eventId)
        {
            case 41: // CREATELOAD
                // Add the custom logic here.
                break;
        }
    }
}
```

Inside `ProcessQueue`, read the message values we need, validate them, and then run the logic for that event. Log the full exception if something fails so the service does not fail silently.

## Step 3 - Add ProjectInstaller

This step is required. Without `ProjectInstaller`, `InstallUtil` may finish without actually creating a Windows service.

```csharp
[RunInstaller(true)]
public partial class ProjectInstaller : Installer
{
    public ProjectInstaller()
    {
        InitializeComponent();
    }
}
```

In `ProjectInstaller.Designer.cs`, configure the service like this:

```csharp
serviceProcessInstaller1.Account = ServiceAccount.LocalSystem;
serviceInstaller1.ServiceName = "ExpertAppEventHandler";
serviceInstaller1.DisplayName = "Expert App Event Handler";
serviceInstaller1.Description = "Processes Made4net application events.";
serviceInstaller1.StartType = ServiceStartMode.Automatic;

Installers.AddRange(new Installer[] {
    serviceProcessInstaller1, serviceInstaller1
});
```

> **Note:** Make sure `serviceInstaller1` and `serviceProcessInstaller1` are declared as fields in the partial `ProjectInstaller` class. If they are missing, Visual Studio shows `CS1061` errors.

## Step 4 - Add App.config

The config file controls logging. After building, it must be beside the EXE as `BESAppEventHandler.exe.config`.

```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
  <appSettings>
    <add key="UseLogs" value="1" />
    <add key="LogPath" value="C:\Program Files (x86)\Made4Net\SCExpert\Logs\AppEventHandler" />
  </appSettings>
  <startup useLegacyV2RuntimeActivationPolicy="true">
    <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.8" />
  </startup>
</configuration>
```

## Step 5 - Build and copy the files

1. Build the project in Release mode.
2. Copy `AppEventHandler.exe`, `AppEventHandler.exe.config`, and any required custom DLLs to the server.
3. On the server, place them in the following folder:

   ```text
   C:\Program Files (x86)\Made4Net\SCExpert\Services
   ```

4. Make sure the service account has permission to write to the configured log folder.

## Step 6 - Install the service

Open Command Prompt as Administrator and run:

```bat
C:\Windows\Microsoft.NET\Framework\v4.0.30319\InstallUtil.exe "C:\Program Files (x86)\Made4Net\SCExpert\Services\BESAppEventHandler.exe"
```

The output should contain these lines:

```text
Installing service ExpertAppEventHandler...
Service ExpertAppEventHandler has been successfully installed.
```

## Step 7 - Start and verify the service

```bat
sc query ExpertAppEventHandler
sc qc ExpertAppEventHandler
net start ExpertAppEventHandler
```

Use `ExpertAppEventHandler` because that is the `ServiceName` configured in `ProjectInstaller`. After starting it, the service should show `RUNNING` in Services or in the `sc query` output.

## Step 8 - Test it

1. Trigger one controlled Made4net event. For BES, we used event 41 by creating a test load.
2. Open the AppEventHandler log and confirm that the event and message values were received.
3. Check that the database update or export happened once.
4. If the message keeps retrying, check the exception in the handler log and the Windows Event Viewer.

## Quick troubleshooting

- **Service does not exist:** `ProjectInstaller` was probably missing or `InstallUtil` did not find it.
- **Service name is invalid:** use `ExpertAppEventHandler`, not `BESAppEventHandler`.
- **Service starts and stops:** check Event Viewer, the `.config` file, missing DLLs, and queue configuration.
- **MSMQ format name is invalid:** verify the private queue name and Made4net queue configuration.
- **Access denied:** give the service account permission to the log or export folder.
