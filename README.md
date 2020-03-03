## System Accounts migration to Okta 

Objective is to migrate existing system accounts to Okta IAM solution with *minimal*  impact to existing functionality. This migration is inline with future road map to utilize OAuth2 as authentication and authorization solution.  

 - [x] Create a new developer account at [developer.okta.com](https://developer.okta.com/). After login Okta provides a pre-configured custom Authorization Server with the name `default`
 
  - [x] Create a API token and make a note of the token as it will be the only time that can be viewed.  This API token needs to be passed as an Authorization header for all Okta REST API calls
 
	 ![#Note](https://placehold.it/15/f03c15/000000?text=+)  *Note:* Tokens are valid for 30 days and automatically refresh with each API call. Tokens that aren't used for 30 days expire. The token lifetime is currently fixed and can't be changed. 
	 
 - [x]  Create a new `scope`. Scopes specify what access privileges are being requested as part of the authorization. Do not set it as default scope and force the user to specify the scope on the authorization request 
  
 - [x] Create new **service** client with `grant_type: client_credentials` and `token_endpoint_auth_method: client_secret_post`  using REST API. The response will have `client_id` and `client_secret`. The `client_secret` is shown only on the response of the creation and cannot be retrieved later using the API
 
	 ![#Note](https://placehold.it/15/f03c15/000000?text=+)  *Note:* This will create a new Application in Okta and can be viewed from the dashboard including `client_id` and `client_secret` 
	 
	 ```
	curl --location --request POST '{{OKTA_URL}}/oauth2/v1/clients' \
	--header 'Accept: application/json' \
	--header 'Content-Type: application/json' \
	--header 'Authorization: SSWS {{API_KEY}}' \
	--data-raw '  {
	    "client_name": "batasam",
	    "redirect_uris": [],
	    "response_types": [
	      "token"
	    ],
	    "grant_types": [
	      "client_credentials"
	    ],
	    "token_endpoint_auth_method": "client_secret_post",
	    "application_type": "service"
	  }'
	 ```
	 <br/>
 - [x] Update/Reset client secret using REST API
	 ```
	curl --location --request POST '{{OKTA_URL}}/oauth2/v1/clients/{{client_id}}/lifecycle/newSecret' \
	--header 'Accept: application/json' \
	--header 'Content-Type: application/json' \
	--header 'Authorization: SSWS {{API_KEY}}' \
	--data-raw ''
	 ```
 - [x] ~~Get client secret~~  Okta does not provide this functionality using API. However we can view client secret from the developer dashboard

 - [x] Client authorization using client_id and secret
	 ```
	curl --location --request POST '{{OKTA_URL}}/oauth2/{{AUTH_SERVER}}/v1/token' \
	--header 'accept: application/json' \
	--header 'Content-Type: application/x-www-form-urlencoded' \
	--data-urlencode 'grant_type=client_credentials' \
	--data-urlencode 'scope={{CUSTOM_SCOPE}}' \
	--data-urlencode 'client_id={{CLIENT_ID}}' \
	--data-urlencode 'client_secret={{CLIENT_SECRET}}'
	 ```
- [x] Analyze changes needed for  System Accounts API to integrate with Okta. Refactor following java classes and update the application.properites  file with Okta Authorization server & Token details
	 - SystemAccountRestController.java
	 - SFAController.java
	 - SystemAccountService.java
	 - CwsAppService.java
	
 - [x] Add new column `client_id` to table `cwsflow` to store the client_id generated by Okta during client creation

	![#Note](https://placehold.it/15/f03c15/000000?text=+)  *Note:* When creating new clients Okta generates a client_id **uuid** along with the secret.  This **uuid** is used by Okta REST API to do any client CRUD actions  

 - [x] Analyze UI changes needed for calling updated System Accounts API. Refactor following *html* and *typescript* files based on new design
	 - review.component.html 
	 - review.component.ts
	 - system-accounts-service.ts

### UML diagrams

####   Create new System Accounts client using Okta REST API
New system accounts client is created when `GSA Security Approver` approves a pending request
```mermaid
sequenceDiagram
UI ->> API Gateway: Approve System Account
API Gateway->>CwsAppController: PUT /v1/applications/approve
CwsAppController->> CWSAppService: approveApplication
CWSAppService ->> Okta REST API: POST oauth2/v1/clients
Note right of Okta REST API: Creates new client,<br/>responses with <br/> client_id and secret<br/> 
Okta REST API -->> CWSAppService : Success 200 OK
CWSAppService ->> CwsRepo: Update Status and Save the client_id in DB 
CwsRepo -->> CWSAppService: CwsApplication
CWSAppService -->> CwsAppController: CwsApplication
CwsAppController -->> API Gateway: Success {CwsApplication}
API Gateway -->> UI: Success {CwsApplication}
```
####   Get System Accounts client details
Get system accounts client details from the DB without need to talk to Okta
```mermaid
sequenceDiagram
UI ->> API Gateway: Get client details
API Gateway->>System Accounts API: GET /system-accounts/{id}
System Accounts API ->> CwsAppService:  getApplication(:id)
Note right of CwsAppService: Do not need to go to <br/>Okta for details
CwsAppService ->> CwsRepo: findById(:id)
CwsRepo->> CwsAppService : CwsApplication
CwsAppService -->> System Accounts API: CwsApplication
System Accounts API -->> API Gateway: Success {CwsApplication}
API Gateway -->> UI: Success {CwsApplication}
```
<br/>

#### Reset System Accounts client secret 
Generates a new client secret for the specified client Application

```mermaid
sequenceDiagram
UI ->> API Gateway: Get client details
API Gateway->>System Accounts API: PUT /api/system-account-passwords/forgot/{uid}
System Accounts API ->> SystemAccountService:  
SystemAccountService -->> SystemAccountService: Validate OTP
SystemAccountService ->> Okta REST API: POST {{clientId}}/lifecycle/newSecret
Note right of SystemAccountService : {id} is the uuid that <br/>Okta generates <br/>when creating client<br/> and responses with <br/> new secret
Okta REST API -->> SystemAccountService: 
SystemAccountService -->> System Accounts API: 
System Accounts API -->> API Gateway: Success {CwsApplication}
API Gateway -->> UI: Success {CwsApplication}
```

#### Authenticate  System Accounts client using Okta REST API
Request an access token using the Client Credentials grant flow,
```mermaid
sequenceDiagram
SOAP ->> API Gateway: Authenticate client
Note right of SOAP: HttpHeaders <br/>{"clientId": "ID",<br/> "secret":"SECRET"}
API Gateway->>SFAController: POST sfa/auth
SFAController ->> SystemAccountService: getSystemAccount
SystemAccountService ->> Okta REST API: POST /{{default}}/v1/token
Okta REST API-->> SystemAccountService: Success 200 {{AUTH}}
Note right of SystemAccountService: Repsonse with <br/> access_token and <br/> other details
SystemAccountService ->> CwsRepo: findById(:id)
CwsRepo ->> SystemAccountService: CwsApplication
SystemAccountService -->> SFAController: {CwsApplication, AUTH}
SFAController -->> API Gateway: Success {CwsApplication, AUTH}
API Gateway -->> SOAP: Success {CwsApplication, AUTH}
```
<br/>
<br/>

#### Migration of existing users
 Have to run one time migration job to transfer all system accounts user to Okta. This will generate `client_id` and  `client_secret` for the existing users. This `client_secret` has to shared with the existing clients which is required for authentication 
  
#### Dependencies

 1. Infrastructure team to provide okta instance for both lower and higher environments

#### Questions: 
Below are some questions that were answered by **Ruchir Mehta** and team

1. Do we currently store all the System Accounts related the data in the DB. In other words, do both Forgerock and System Accounts DB have identical data?  

	A : At  each and every POST/PUT calls we keep all the fields mentioned in CWSApplication.java in the SystemAccount DB. (i.e. including status which will change from Draft, Pending Review, Pending Approval, Approved etc etc.).  
	Once GSA Security Approver gives final approval, the System Account DB will be updated with "Approved" Status(Refer : CWSAppConroller.java "/approve" method) and at the same time the Forgerock create api(Refer : SystemAccountService.java --> createAccount() and SystemAccount.java) will be called to create a new System Account.
	
2. Is there a data sync between Forgerock and System Accounts DB? If so, is there a schedule for that?

	A : No currently we do not have any sync job between Forgerock and System Accounts DB
	
3. Are there any other clients (other than the U/I) that utilize the System Accounts API?

	A : System Integration team(Orange) uses /sfa and /system-account endpoints(SystemAccountRestController.java and sfaController.java) for single factor authentication and retrieving System Account Details.

	Also They have some other business logic w.r.t. IP address and Roles

4. During creation of System Accounts, the client provides the IP addresses that can access the System Account APIs. We want to understand who validates the client IP addresses (is it the API gateway or Forgerock)?

	A: 2 Microservices - opportunities_soap_api , interface_api (Orange Team) validates the IP Address associated with System Account(by making sfa/auth call) with the SOAP/REST request IPs. There is no IP-Address Validation logic in forgerock or other microservices apart from (opportunities_soap_api , interface_api).

#### Assumptions:

 1. We are not migrating existing passwords from ForgeRock for POC or for the future implementation  

2. We assume that we are not considering 90 day password expiration policy for this POC implementation. If required, this will be accounted for in the actual implementation once it is approved.  

3. Currently on deactivate, we see that the request is sent to Forgerock to deactivate and the System Account DB is getting updated at the same time. We do not see requirements for Deactivate action for the POC. Our assumption is that this feature will not be part of the POC and will be accounted for in the actual implementation once it is approved.  

4. Based on the POC requirements, we assume that only SOAP services need to change to pass a client secret. Working with client teams for them to make these changes on their side is not part of the scope of this POC. This will be accounted for in the actual implementation once it is approved.

5. Based on POC requirements, we will identify changes needed in the U/I for set password, forgot password and reset password. However, the actual implementation will be accounted for once it is approved.
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTQ4ODEzNzQ1MCwxMzU4MjIyOTA3LC0xND
c3OTM0MTQxLDU4NzYzOTkwNCwtMTczMjY3MDU1LDE0MTIxMDIw
MjcsNDI0NTgyMzQ5LC0xOTkzMjI5MjI2LDU3NTg4MzA3MSwxOT
gyMTYxNjE5LDEyNDcxOTQxNSwxMDEzNjg3NTk4LC0zOTcwNDcy
NDMsLTE5NDg1NzY3NTksNzgyMDgyNjM3LC0xNTQ2NDIzNDcwLD
EyODY1NjA0NTQsLTEzODA0MzI5NzIsMjEyNTI4Mzg2MiwxOTY3
MDM1MjhdfQ==
-->