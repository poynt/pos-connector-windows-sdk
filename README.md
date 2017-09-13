# PoyntPOSBridgeSDK Usage

## Importing the DLL into your project

1. Install *PoyntPOSBridgeSampleInstaller.msi* 

2. Right click on the project, choose Add/Reference, Browse (in the lower right), then navigate to the PoyntPOSBridge.dll and add it 

(the default path will be: `C:\Program Files (x86)\PoyntPOSBridgeSample\PoyntPOSBridge.dll`)

3. Start the Poynt Terminal (or [Poynt Emulator](https://poynt.github.io/developer/tut/setup-poyntos.html) and launch the `POS BRIDGE` application. POS BRIDGE application is availalbe via Poynt Store on the terminal.

3. There is a full working console application at the bottom of this document.


## Logging

The PoyntPOSApi will log requests and responses to the log provided in the constructor. To create a basic file logger use the Logger factory:

```csharp
ILogger apiLogger = LoggerFactory.CreateFileLogger("posApi", LoggingLevel.DEBUG, loggerPath);
```

The following are logged:
 - log level INFO: PoyntPOSApi requests and responses
 - log level ERROR: PoyntPOSFinder and PoyntPOSApi errors
 - log level DEBUG: PoyntPOSFinder

## PoyntPOSFinder

The PoyntPOSFinder object is used to discover terminals on the local network. The network must support UDP multicast.

It runs on the same thread as you instantiate it on so you're responsible for putting it in a worker thread if you don't want to block your UI. When a terminal is discovered the finder will fire a FoundPOSEventHandler event. For example:

```csharp
void Discover()
{
	try
	{
		PoyntPOSFinder finder = new PoyntPOSFinder(finderLogger) {
			PoyntServiceName = "PoyntOSConnector";	
		};
		finder.FoundPoyntPOSEventHandler += Finder_FoundPoyntPOSEventHandler;
		finder.Discover(); // listens on udp multicast 8080 for mdns packets 
	}
	catch (Exception ex)
	{
		System.Diagnostics.Debug.WriteLine("Finder Failed: " + ex.Message);
	}
}

void Finder_FoundPoyntPOSEventHandler(object sender, FoundPoyntPOSEventArgs e)
{
	// the event args will contain the IP address and port discovered
	string ipAddress = e.IPAddress.ToString();
	int port = e.Port;

	String msg = String.Format("Poynt System advertised at {0} port {1}.", ipAddress, port);
	System.Diagnostics.Debug.WriteLine(msg);

	// stop the finder once a terminal is discovered
	e.Stop = true;
}
```

## PoyntPOSApi

The PoyntPOSApi class contains all the methods to perform various operations on the Terminal.

Create a new PoyntPOSApi by providing the IP address and a logger. The default timeout is 2500ms but can be overwritten. Once you have paired with a terminal, set the key to the pairing code. For example,

To pair with a terminal using RSA to exchange the pairing code use IP address to create a PoyntPOSAPI object and then use the PairDeviceWithKey method. 

An Example:

```csharp
var ipFromDiscovery = "192.168.2.133";
posApi = new PoyntPOSApi(ipFromDiscovery, apiLogger)
{
	DefaultTimeoutInMs = 60000,
};

var req = new PairDeviceWithKeyRequest();
req.PairingRequest.PairingCode = "my identifier";

var response = posApi.PairDeviceWithKey("External POS System 0", req);

if (!String.IsNullOrEmpty(response.PairingCode))
{
	System.Diagnostics.Debug.WriteLine("Paired with code " + response.PairingCode);
	// You can now make method calls on the posApi object.
}
else
{
	var errors = string.Join(";", response.Error.Select(x => x.Key + "=" + x.Value).ToArray());
	System.Diagnostics.Debug.WriteLine("Unable to pair with key. Errors: " + errors);
}
```

## A full console application example

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using PoyntPOSBridge;

namespace PoyntTestAppCSharp
{
    class Program
    {
        static string desktopPath = Environment.GetFolderPath(Environment.SpecialFolder.Desktop);
        static ILogger finderLogger = LoggerFactory.CreateFileLogger("finderApi", LoggingLevel.DEBUG, desktopPath);
        static ILogger apiLogger = LoggerFactory.CreateFileLogger("posApi", LoggingLevel.DEBUG, desktopPath);
        static PoyntPOSFinder finder = new PoyntPOSFinder(apiLogger);

        static void Main(string[] args)
        {
            finder.FoundPoyntPOSEventHandler += new FoundPoyntPOSEventHandler(finder_FoundPoyntPOSEventHandler);
            Console.WriteLine("Looking for a Poynt Terminal");
            finder.Discover();
            Console.ReadKey();
        }

        static void finder_FoundPoyntPOSEventHandler(object sender, FoundPoyntPOSEventArgs e)
        {
            var ipAddress = e.IPAddress.ToString();
            int port = e.Port;

            String msg = String.Format("Poynt System advertised at {0} port {1}.", ipAddress, port);
            Console.WriteLine(msg);

            PoyntPOSApi api = new PoyntPOSApi(ipAddress, apiLogger)
            {
                DefaultTimeoutInMs = 60000,
            };

            var req = new PairDeviceWithKeyRequest();
            req.PairingRequest.PairingCode = "my identifier";
            var response = api.PairDeviceWithKey("External0", req);

            e.Stop = true;

            if (!String.IsNullOrEmpty(response.PairingCode))
            {
                Console.WriteLine("Paired with code " + response.PairingCode);
                // You can now make method calls on the posApi object.

                CreateSalesRequest(api);
            }
            else
            {
                var errors = string.Join(";", response.Error.Select(x => x.Key + "=" + x.Value).ToArray());
                Console.WriteLine("Unable to pair with key. Errors: " + errors);
            }

        }

        static void CreateSalesRequest(PoyntPOSApi api)
        {

            List<OrderItem> items = new List<OrderItem>()
            {
                new OrderItem()
                {
                    Name = "item1",
                    Quantity = 1,
                    Tax = 0,
                    Status = "ORDERED",
                    UnitOfMeasure = "EACH",
                    UnitPrice = 100,
                },
            };

            var salesRequest = AuthorizeSalesRequest.Create(60000, "USD", 128);
            salesRequest.Payment.Order.Items.AddRange(items);
            salesRequest.Payment.Order.Notes = "order notes";
            salesRequest.Payment.Notes = "payment notes";
            salesRequest.Payment.ReferenceId = "ExtReferenceID";
            salesRequest.Payment.References = new List<TransactionReference>()
            {
                new TransactionReference()
                {
                    Id = "external-id-3fda",
                    CustomType = "externalReferenceId",
                    Type = "CUSTOM"
                }
            };


            salesRequest.Payment.ReferenceId = "reference-id-333";
            salesRequest.Payment.ManualEntry = true;

            var response = api.AuthorizeSales(salesRequest);

            Console.WriteLine("AuthorizeSales Payment Status: " + response.Payment.Status);
        }
    }
}
```

The following API commands are supported:
 - AuthorizeAdjustment
 - AuthorizeCompletion
 - AuthorizePreSale
 - AuthorizeRefund
 - AuthorizeSales
 - AuthorizeVoid
 - PairDevice
 - PairDeviceWithKey
 - PingDevice
 - PrintReceipt
 - ScanData
 - ShowItemsOnSecondScreen
 - VoiceAuthorizeSales
