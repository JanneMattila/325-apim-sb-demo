# Azure API Management & Azure Relay - Hybrid Connections & Web Service demo

This demo tries to illustrate the use of Azure API Management, Azure Relay and Hybrid Connections for
connecting to your on-premises Web Service API.

Same as ascii graphics:

```batch
      /------\     /-------\    /-------\    /---------\
User  | API  |     | AZURE |    | HC    |    | Web     |
  =>  | MGMT |  => | RELAY | => | PROXY | => | Service |
      \------/     \-------/    \-------/    \---------/
```

Steps to implement the demo:

1. Create Azure API Management using portal
2. Create Azure Relay using portal
3. Clone Azure Relay example for `hcreverseproxy`
4. Create Web Service
5. Add API to APIM and policy to connect to relay
6. Run demo apps (`hcreverseproxy` and Web Service)
7. Invoke published API

Create Azure Relay and create new Hybrid Connection to it e.g. `apim`.
Create access policy for `Send` and call it `apimSender`.
Create access policy for `Listen` and call it `intraListener`.

Clone Azure Relay repository to your machine
[https://github.com/Azure/azure-relay](https://github.com/Azure/azure-relay).

Open `Microsoft.Azure.Relay.ReverseProxy.sln` underneath folder `azure-relay/samples/hybrid-connections/dotnet/hcreverseproxy/`.

**Note**: Above example is _not_ Production grade implementation and it lacks
exception handling and offline connection handling logic. You should enhance that
to fit your needs. For example you should add `Offline` handler implementation:

```csharp
public HybridConnectionReverseProxy(string connectionString, Uri targetUri)
{
  this.listener = new HybridConnectionListener(connectionString);
  
  // Add Offline event handler
  this.listener.Offline += Listener_Offline;
  // ...
}

private void Listener_Offline(object sender, EventArgs e)
{
    Console.WriteLine("Listener Offline. Do something smart 😉.");
}
```

Create new "*ASP.NET Web Application (.NET Framework)*" project. Add new "*Web Service (ASMX)*" file to the project.

Place following code to it:

```charp
using System.Collections.Generic;
using System.Web.Services;

namespace DemoWebApp
{
    [WebService(Namespace = "http://tempuri.org/")]
    [WebServiceBinding(ConformsTo = WsiProfiles.BasicProfile1_1)]
    [System.ComponentModel.ToolboxItem(false)]
    public class WebService1 : System.Web.Services.WebService
    {
        public class Product
        {
            public int ID { get; set; }
            public string Name { get; set; }
        }

        [WebMethod]
        public List<Product> GetProducts()
        {
            return new List<Product>()
            {
                new Product() { ID = 1, Name = "AAA" },
                new Product() { ID = 2, Name = "BBB" },
                new Product() { ID = 3, Name = "CCC" },
                new Product() { ID = 4, Name = "DDD" },
                new Product() { ID = 5, Name = "EEE" }
            };
        }
    }
}
```

Add required named values for APIM by setting `apimSender` for `accessKeyName` and access key of that into `accessKey`.
Create new API and `backend` to APIM and place following policy to it (from [API Policy snippets](https://github.com/Azure/api-management-policy-snippets/blob/master/examples/Generate%20Azure%20Relay%20Token.policy.xml)):

```xml
<policies>
    <inbound>
        <!-- Add your wcf relay address as the base URL below -->
        <set-backend-service base-url="https://<your-relay-name-here>.servicebus.windows.net/apim" />
        <!-- verify if there is a relaytoken key stored in cache -->
        <cache-lookup-value key="@("relaytoken")" variable-name="relaytoken" />
        <choose>
            <!-- If there is no key stored in cache -->
            <when condition="@(!context.Variables.ContainsKey("relaytoken"))">
                <set-variable name="resourceUri" value="@(context.Request.Url.ToString())" />
                <!-- Retrieve Shared Access Policy key from  Name Value store -->
                <set-variable name="accessKey" value="{{accessKey}}" />
                <!-- Retrieve Shared Access Policy key name from  Name Value store -->
                <set-variable name="keyName" value="{{accessKeyName}}" />
                <!-- Generate the relaytoken key -->
                <set-variable name="relaytoken" value="@{
                    TimeSpan sinceEpoch = DateTime.UtcNow - new DateTime(1970, 1, 1);
                    string expiry =  Convert.ToString((int)sinceEpoch.TotalSeconds + 3600);
                    string resourceUri = (string)context.Variables["resourceUri"];
                    string stringToSign = Uri.EscapeDataString (resourceUri) + "\n" + expiry;
                    HMACSHA256 hmac = new HMACSHA256(Encoding.UTF8.GetBytes((string)context.Variables["accessKey"]));
                    string signature = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(stringToSign)));
                    string sasToken = String.Format("SharedAccessSignature sr={0}&sig={1}&se={2}&skn={3}",
                    Uri.EscapeDataString(resourceUri), Uri.EscapeDataString(signature), expiry, context.Variables["keyName"]);
                    return sasToken;
                    }" />
                <!-- Store the relaytoken in the cache -->
                <cache-store-value key="relaytoken" value="@((string)context.Variables["relaytoken"])" duration="10" />
            </when>
        </choose>
        <!-- Create the ServiceBusAuthorization header using the relaytoken as value -->
        <set-header name="ServiceBusAuthorization" exists-action="override">
            <value>@((string)context.Variables["relaytoken"])</value>
        </set-header>
        <base />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <set-header name="Content-Type" exists-action="override">
            <value>text/xml</value>
        </set-header>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```

**Note**: If you plan to expose multiple endpoint using the same relay from
APIM then you need to make sure that you create separate tokens for that
connection. Otherwise you will get error like this:

`HttpStatusCode: 401, Code: TokenMissingOrInvalid, Message: InvalidAudience: The authorization header contains a token with a wrong audience`

Here's example that you create token per endpoint (just having own names and own caches are enough):

```xml
<!-- clipped -->
        <cache-lookup-value key="@("relaytoken1")" variable-name="relaytoken1" />
<!-- clipped -->
        <choose>
            <!-- If there is no key stored in cache -->
            <when condition="@(!context.Variables.ContainsKey("relaytoken1"))">
```

Create `POST` operation named `GetProducts` to the `backend` API.

Run this project and take note of the web service url e.g. `http://localhost:2811/WebService1.asmx`.
This means that the url for the `GetProducts` is `http://localhost:2811/WebService1.asmx/GetProducts`.

Edit project `Microsoft.Azure.Relay.ReverseProxy` properties and update the Debug parameters
to match your Azure Relay information for `intraListener`. Example: `Endpoint=sb://<your-relay-name-here>.servicebus.windows.net/;SharedAccessKeyName=intraListener;SharedAccessKey=SGFoISBZb3UgZm91bmQgbXkgZWFzdGVyIGVnZyA6RA==;EntityPath=apim http://localhost:2811/webservice1.asmx/`.

Use Visual Studio Code with [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension.
Here's example code:

```http
@apim = https://<your-apim-instance-here>.azure-api.net/
@subscriptionKey = your-apim-subscription-key-here

### Direct call to the Web Service
POST http://localhost:2811/webservice1.asmx/GetProducts HTTP/1.1

### Call APIM -> Hybrid connection -> Proxy -> Web service
POST {{apim}}/backend/GetProducts HTTP/1.1
Ocp-Apim-Subscription-Key: {{subscriptionKey}}
```

APIM response should look like this:

```http
HTTP/1.1 200 OK
Cache-Control: max-age=0, private
Transfer-Encoding: chunked
Via: 1.1 <your-relay-name-here>.servicebus.windows.net
Content-Type: text/xml
Strict-Transport-Security: max-age=31536000
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Request-Context: appId=cid-v1:03fb3907-8c9d-41bb-910d-d0431618e65f
Date: Wed, 22 Jan 2020 19:27:44 GMT
Connection: close

<?xml version="1.0" encoding="utf-8"?>
<ArrayOfProduct
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:xsd="http://www.w3.org/2001/XMLSchema"
  xmlns="http://tempuri.org/"
  <Product>
    <ID>1</ID>
    <Name>AAA</Name>
  </Product>
  ...
</ArrayOfProduct>
```

Note: If you want to convert that content to json, then you can use policy
to do that as well. Here's example policy:

```xml
<policies>
  <inbound>
    <base />
    <rewrite-uri template="/WebService1.asmx" copy-unmatched-params="false" />
    <set-header name="SOAPAction" exists-action="override">
      <value>"http://tempuri.org/GetProducts"</value>
    </set-header>
    <set-body template="liquid">
      <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/" xmlns="http://tempuri.org/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
        <soap:Body>
          <GetProducts>
          </GetProducts>
        </soap:Body>
      </soap:Envelope>
    </set-body>
    <set-header name="Content-Type" exists-action="override">
      <value>text/xml</value>
    </set-header>
  </inbound>
  <backend>
    <base />
  </backend>
  <outbound>
    <base />
    <choose>
      <when condition="@(context.Response.StatusCode < 400)">
        <set-body template="liquid">
          {
            "getProductsResponse": {
             "GetProductsResult": [
              {% JSONArrayFor item in body.envelope.body.GetProductsResponse.GetProductsResult -%}
               {
                "iD" : {{item.ID}},
                "name" : "{{item.Name}}"
               }
              {% endJSONArrayFor -%}
             ]
            }
          }
        </set-body>
      </when>
      <otherwise>
        <set-variable name="old-body" value="@(context.Response.Body.As<string>(preserveContent: true))" />
        <!-- Error response as per https://github.com/Microsoft/api-guidelines/blob/master/Guidelines.md#7102-error-condition-responses -->
        <set-body template="liquid">
        {
         "error": {
          "code": "{{body.envelope.body.fault.faultcode}}",
           "message": "{{body.envelope.body.fault.faultstring}}"
         }
        }
        </set-body>
        <choose>
          <when condition="@(string.IsNullOrEmpty(context.Response.Body.As<JObject>(preserveContent: true)["error"]["code"].ToString()) && string.IsNullOrEmpty(context.Response.Body.As<JObject>(preserveContent: true)["error"]["message"].ToString()))">
            <set-body>@{
              var newResponseBody = new JObject();
              newResponseBody["error"] = new JObject();
              newResponseBody["error"]["code"] = "InvalidErrorResponseBody";
              if (string.IsNullOrEmpty((string)context.Variables["old-body"]))
              {
                newResponseBody["error"]["message"] = "The error response body was not a valid SOAP error response. The response body was empty.";
              }
              else
              {
                newResponseBody["error"]["message"] = "The error response body was not a valid SOAP error response. The response body was: '" + context.Variables["old-body"] + "'.";
              }
              return newResponseBody.ToString();
            }
            </set-body>
          </when>
        </choose>
      </otherwise>
    </choose>
    <set-header name="Content-Type" exists-action="override">
      <value>application/json</value>
    </set-header>
  </outbound>
  <on-error>
    <base />
  </on-error>
</policies>
```

Output from that response would be then in json format:

```http
HTTP/1.1 200 OK
Cache-Control: max-age=0, private
Transfer-Encoding: chunked
Via: 1.1 <your-relay-name-here>.servicebus.windows.net
Content-Type: application/json
Strict-Transport-Security: max-age=31536000
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Request-Context: appId=cid-v1:03fb3907-8c9d-41bb-910d-d0431618e65f
Date: Wed, 22 Jan 2020 19:27:44 GMT
Connection: close

{
  "getProductsResponse": [
    {
      "id": "1",
      "name": "AAA",
    }
    ...
  ]
}
```
