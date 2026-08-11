# FedEx REST integration

## Setup

This is meant to be used on a client with the packstation. Ensure the packstation is installed first if required.


Start by running the necessary scripts in the client's DB.

AppScripts are used to add needed views and fields for FedEx 

SysScripts are used to configure sys params for log path and the base url we point to (FedEx sandbox vs prod)

DuplicateScripts are needed for both FedEx and UPS (For example the shippingAccount table and the outboundorder shipper service/account fields). If the client already has a carrier integration these may not be needed

Next, add the FedExRest.cs to the App.Logic and rebuild.

Finally, on the client's packstation update the create shipment buttons to call the FedEx rest integration. The button logic should query the FedExShipmentInfo view based on orderID or packID and pass the resulting DataSet to FedEx.Logic.sendShipment() (this function should handle all authentication and response handling)

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


## How to use


### Scripts
There are 3 folders included for the FedEx integration

1. App

The application scripts needed specific to FedEx. These include 

BillToFields used for 3rd party billing

FedEx services codelist (Defines the various service methods that we can send to FedEx. Maps a human readable name to the value FedEx expects)

vFedexShipmentInfo - a new view that maps outbound order, contact, consignee, and any other fields to the expected field names for the integration. Any customizations or new mappings should be made to this view

2. Sys

This is just to add sys_params for logging, and the base URL used by FedEx (Either to point to FedEx test or prod)

3. Duplicate Scripts

These are used by both FedEx and UPS integrations and may not be needed if the client already has an integration installed.

These include the ShippingAccount table create statement, The Shipping udf fields on outboundorheader (keeps track of which carrier, service, and account# are to be used on each order)

And the BillToUDFs on outboundorheader (in case the client wants to use 3rd party or receiver billing. If client is only doing BillTo sender or shipper these won't be needed)


### Application

The FedExRest Logic is contained in one file FedExRest.cs

This contains all necessary logic and functions for getting payload data, authenticating with FedEx, building the createShipment request, and handling the response (either printing labels or throwing errors for the user to see)
