# Generate and configure an SSL certificate for backend authentication<a name="getting-started-client-side-ssl-authentication"></a>

 You can use API Gateway to generate an SSL certificate and then use its public key in the backend to verify that HTTP requests to your backend system are from API Gateway\. This allows your HTTP backend to control and accept only requests that originate from Amazon API Gateway, even if the backend is publicly accessible\. 

**Note**  
 Some backend servers might not support SSL client authentication as API Gateway does and could return an SSL certificate error\. For a list of incompatible backend servers, see [Amazon API Gateway important notes](api-gateway-known-issues.md)\. 

 The SSL certificates that are generated by API Gateway are self\-signed, and only the public key of a certificate is visible in the API Gateway console or through the APIs\. 

**Topics**
+ [Generate a client certificate using the API Gateway console](#generate-client-certificate)
+ [Configure an API to use SSL certificates](#configure-api)
+ [Test invoke to verify the client certificate configuration](#test-invoke)
+ [Configure a backend HTTPS server to verify the client certificate](#certificate-validation)
+ [Rotate an expiring client certificate](#certificate-rotation)
+ [API Gateway\-supported certificate authorities for HTTP and HTTP proxy integrations](api-gateway-supported-certificate-authorities-for-http-endpoints.md)

## Generate a client certificate using the API Gateway console<a name="generate-client-certificate"></a>

1. Open the API Gateway console at [https://console\.aws\.amazon\.com/apigateway/](https://console.aws.amazon.com/apigateway/)\. 

1. Choose a REST API\.

1. In the main navigation pane, choose **Client Certificates**\.

1. From the **Client Certificates** pane, choose **Generate Client Certificate**\.

1.  Optionally, for **Edit**, choose to add a descriptive title for the generated certificate and choose **Save** to save the description\. API Gateway generates a new certificate and returns the new certificate GUID, along with the PEM\-encoded public key\. 

You're now ready to configure an API to use the certificate\.

## Configure an API to use SSL certificates<a name="configure-api"></a>

These instructions assume that you already completed [Generate a client certificate using the API Gateway console](#generate-client-certificate)\.

1.  In the API Gateway console, create or open an API for which you want to use the client certificate\. Make sure that the API has been deployed to a stage\. 

1. Choose **Stages** under the selected API and then choose a stage\.

1. In the **Stage Editor** panel, select a certificate under the **Client Certificate** section\.

1. To save the settings, choose **Save Changes**\.

   If the API has been deployed previously in the API Gateway console, you'll need to redeploy it for the changes to take effect\. For more information, see [Redeploy a REST API to a stage](how-to-deploy-api-with-console.md#apigateway-how-to-redeploy-api-console)\.

After a certificate is selected for the API and saved, API Gateway uses the certificate for all calls to HTTP integrations in your API\. 

## Test invoke to verify the client certificate configuration<a name="test-invoke"></a>

1. Choose an API method\. In **Client**, choose **Test**\.

1. From **Client Certificate**, choose **Test** to invoke the method request\. 

 API Gateway presents the chosen SSL certificate for the HTTP backend to authenticate the API\. 

## Configure a backend HTTPS server to verify the client certificate<a name="certificate-validation"></a>

These instructions assume that you already completed [Generate a client certificate using the API Gateway console](#generate-client-certificate) and downloaded a copy of the client certificate\. You can download a client certificate by calling [https://docs.aws.amazon.com/apigateway/latest/api/API_GetClientCertificate.html](https://docs.aws.amazon.com/apigateway/latest/api/API_GetClientCertificate.html) of the API Gateway REST API or [https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-client-certificate.html](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-client-certificate.html) of AWS CLI\. 

 Before configuring a backend HTTPS server to verify the client SSL certificate of API Gateway, you must have obtained the PEM\-encoded private key and a server\-side certificate that is provided by a trusted certificate authority\. 

If the server domain name is `myserver.mydomain.com`, the server certificate's CNAME value must be `myserver.mydomain.com` or `*.mydomain.com`\. 

Supported certificate authorities include [Let's Encrypt](https://letsencrypt.org/) or one of [API Gateway\-supported certificate authorities for HTTP and HTTP proxy integrations](api-gateway-supported-certificate-authorities-for-http-endpoints.md)\. 

As an example, suppose that the client certificate file is `apig-cert.pem` and the server private key and certificate files are `server-key.pem` and `server-cert.pem`, respectively\. For a Node\.js server in the backend, you can configure the server similar to the following:

```
var fs = require('fs'); 
var https = require('https');
var options = { 
    key: fs.readFileSync('server-key.pem'), 
    cert: fs.readFileSync('server-cert.pem'), 
    ca: fs.readFileSync('apig-cert.pem'), 
    requestCert: true, 
    rejectUnauthorized: true
};
https.createServer(options, function (req, res) { 
    res.writeHead(200); 
    res.end("hello world\n"); 
}).listen(443);
```



For a node\-[express](http://expressjs.com/) app, you can use the [client\-certificate\-auth](https://www.npmjs.com/package/client-certificate-auth) modules to authenticate client requests with PEM\-encoded certificates\. 

For other HTTPS server, see the documentation for the server\.

## Rotate an expiring client certificate<a name="certificate-rotation"></a>

The client certificate generated by API Gateway is valid for 365 days\. You must rotate the certificate before a client certificate on an API stage expires to avoid any downtime for the API\. You can check the expiration date of certificate by calling [clientCertificate:by\-id](https://docs.aws.amazon.com/apigateway/latest/api/API_GetClientCertificate.html) of the API Gateway REST API or the AWS CLI command of [get\-client\-certificate](https://docs.aws.amazon.com/cli/latest/reference/apigateway/get-client-certificate.html) and inspecting the returned [expirationDate](https://docs.aws.amazon.com/apigateway/latest/api/API_ClientCertificate.html#expirationDate) property\.

To rotate a client certificate, do the following:

1. Generate a new client certificate by calling [clientcertificate:generate](https://docs.aws.amazon.com/apigateway/latest/api/API_GenerateClientCertificate.html) of the API Gateway REST API or the AWS CLI command of [generate\-client\-certificate](https://docs.aws.amazon.com/cli/latest/reference/apigateway/generate-client-certificate.html)\. In this tutorial, we assume that the new client certificate ID is `ndiqef`\.

1.  Update the backend server to include the new client certificate\. Don't remove the existing client certificate yet\.

   Some servers might require a restart to finish the update\. Consult the server documentation to see if you must restart the server during the update\.

1.  Update the API stage to use the new client certificate by calling [stage:update](https://docs.aws.amazon.com/apigateway/latest/api/API_UpdateStage.html) of the API Gateway REST API, with the new client certificate ID \(`ndiqef`\):

   ```
   PATCH /restapis/{restapi-id}/stages/stage1 HTTP/1.1
   Content-Type: application/json
   Host: apigateway.us-east-1.amazonaws.com
   X-Amz-Date: 20170603T200400Z
   Authorization: AWS4-HMAC-SHA256 Credential=...
   
   {
     "patchOperations" : [
       {
           "op" : "replace",
           "path" : "/clientCertificateId",
           "value" : "ndiqef"
       }
     ]
   }
   ```

   or by calling the CLI command of [update\-stage](https://docs.aws.amazon.com/cli/latest/reference/apigateway/update-stage.html)\.

1.  Update the backend server to remove the old certificate\.

1.  Delete the old certificate from API Gateway by calling the [clientcertificate:delete](https://docs.aws.amazon.com/apigateway/latest/api/API_DeleteClientCertificate.html) of the API Gateway REST API, specifying the clientCertificateId \(`a1b2c3`\) of the old certificate:

   ```
   DELETE /clientcertificates/a1b2c3 
   ```

   or by calling the CLI command of [delete\-client\-certificate](https://docs.aws.amazon.com/cli/latest/reference/apigateway/delete-client-certificate.html):

   ```
   aws apigateway delete-client-certificate --client-certificate-id a1b2c3
   ```

To rotate a client certificate in the console for a previously deployed API, do the following:

1. In the main navigation pane, choose **Client Certificates**\.

1. From the **Client Certificates** pane, choose **Generate Client Certificate**\.

1.  Open the API for which you want to use the client certificate\. 

1. Choose **Stages** under the selected API and then choose a stage\.

1. In the **Stage Editor** panel, select the new certificate under the **Client Certificate** section\.

1. To save the settings, choose **Save Changes**\.

   You need to redeploy the API for the changes to take effect\. For more information, see [Redeploy a REST API to a stage](how-to-deploy-api-with-console.md#apigateway-how-to-redeploy-api-console)\.