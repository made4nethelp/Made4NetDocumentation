# UPS REST integration

## Setup

This is meant to be used on a client with the packstation. Ensure the packstation is installed first if required.


Start by running the necessary scripts in the client's DB.

AppScripts are used to add needed views and fields for UPS 

SysScripts are used to configure sys params for log path and the base url we point to (UPS sandbox vs prod)

DuplicateScripts are needed for both UPS and FedEx (For example the shippingAccount table and the outboundorder shipper service/account fields). If the client already has a carrier integration these may not be needed

Next, add the FedExRest.cs to the App.Logic and rebuild.

Finally, on the client's packstation update the create shipment buttons to call the UPS rest integration. The button logic should query the FedExShipmentInfo view based on orderID or packID and pass the resulting DataSet to FedEx.Logic.sendShipment() (this function should handle all authentication and response handling)




For example, from TMUWS/App.Logic - ExecOrderPack.cs
```C#

case "sendcarriershipment":
    //Get carrier from header DR


    foreach (DataRow dr in ds.Tables[0].Rows)
    {
        string carrier = dr["UDF_CARRIER"].ToString();
        if (carrier.ToLower() == "ups")
        {
            Made4Net.Shared.Logging.LogFile logger = new Made4Net.Shared.Logging.LogFile(Made4Net.Shared.Util.GetSystemParameterValue("UPSLogPath"), 
                $"CreateUPSShipment_{DateTime.Now.ToString("ddMMyyyy")}_{new Random().Next(1000).ToString()}");
            DataSet UPSDS = getUPSPayloadDataOrder(dr["ORDERID"].ToString(), logger);
            pMessage = App.Logic.UPSWebService.SendUPSShipment(UPSDS, logger);
        }
        else if (carrier.ToLower() == "fedex")
        {
            Made4Net.Shared.Logging.LogFile logger = new Made4Net.Shared.Logging.LogFile(Made4Net.Shared.Util.GetSystemParameterValue("FedExLogPath"),
                $"CreateFedexShipment_{DateTime.Now.ToString("ddMMyyyy")}_{new Random().Next(1000).ToString()}");

            DataSet FedExDS = getFedexPayloadDataOrder(dr["MASTERORDER"].ToString(), logger);
            pMessage = Fedex.Logic.FedexRest.sendShipment(FedExDS, logger);
        }
        else
        {
            throw new Exception("No carrier selected!");
        }
    }
    break;

```

### IMPORTANT FOR UPS ###

Also be sure to add VOID logic to the packstation - this sample is from DBBWS/AppScreens/PackOrder.aspx.vb
```C#

	Case "voidshipment"
        Dim UPSShipmentInfoDS As DataSet = getUPSShipmentInfoFromOrderID(ds.Tables(0).Rows(0)("ORDERID"))
        Message = App.Logic.UPSWebService.sendVoidShipmentRequest(UPSShipmentInfoDS)
End Select

```

### Scripts
There are 3 folders included for the UPS integration

1. App

The application scripts needed specific to UPS. These include 

Creating the carrier services codelist (UPS Expects to receive an ID for each service instead of the service name itself. So here we use CodeListDetail, the CODE is the internal UPS ID for each service, and the Description is the human readable. For example for UPS Ground UPS expects to receive an ID of 03 in the service field)

CarrierShipment counter is needed by some older packstation versions and is included here if necessary.

UPS Shipment view is used to map outbound order, contact, and other fields to send to UPS to create the shipment on their end

2. Sys

This is just to add sys_params for logging, the base URL used by UPS (Either to point to UPS test or prod), and the endpoint version to use for shipping

3. Duplicate Scripts

These are used by both FedEx and UPS integrations and may not be needed if the client already has an integration installed.

These include the ShippingAccount table create statement, The Shipping udf fields on outboundorheader (keeps track of which carrier, service, and account# are to be used on each order)

And the BillToUDFs on outboundorheader (in case the client wants to use 3rd party or receiver billing. If client is only doing BillTo sender or shipper these won't be needed)


### Application

The UPS Logic is contained in one file UPSRest.cs

This contains all necessary logic and functions for getting payload data, authenticating with UPS, building the createShipment request, and handling the response (either printing labels or throwing errors for the user to see)


One important difference between UPS and FedEx - FedEx charges per shipment shipped. UPS Charges per shipment created. Because of this, the UPS integration also includes code to VOID a shipment (delete it from UPS for the client). Ensure any UPS testing is done against the test endpoint. This can be confirmed from the URL sys param, and from the tracking info returned from UPS in test (All valid orders in test should get a tracking number back of 1ZXXXXXXXXXX)

VOID cannot be used against the test endpoint however. (This is due to the fact that shipments created in UPS Test don't generate a valid tracking number).

In order to test void - switch the sys_param to point to UPS prod, create a shipment, then void the shipment soon afterwards. 

The void message is fairly simple however, it doesn't send any payload and just includes the tracking number to be voided in the URL
