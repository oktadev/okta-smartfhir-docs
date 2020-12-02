# Okta SMART on FHIR Setup Guide
## Introduction
This guide is intended to walk you through how to setup your very own reference SMART on FHIR implementation using Okta as the identity platform.
For an overview of this project and all of the components, please see here: [Project Introduction](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/README.md)

## Guide Index
* [Prerequisites](#prerequisites)
* [Solution Deployment and Initial Okta Setup](#solution-deployment-and-initial-okta-setup)
* [Okta Profile Setup](#okta-profile-attribute---patient-id)
* [Okta Token Hook Configuration](#okta-token-hook-configuration)
* [Okta Authorization Server Configuration](#okta-authorization-server-configuration)
* [SMART Client Registration - Confidential](#smart-client-registration---confidential)
* [SMART Client Registration - Public](#smart-client-registration---public)

## Prerequisites
* An Okta tenant ([Get one here free](https://developer.okta.com/signup))
* Node.js 12+
* Serverless framework ([Get started here](https://www.serverless.com/framework/docs/getting-started/#via-npm))
* OpenSSL libraries (included on any *nix distribution)
* Postman, or other API invocation tool (not strictly required- but will make configuration easier)

## Solution Deployment and Initial Okta setup
In this section, we'll do some initial Okta setup, as well as deploy all of the requisite endpoints to our chosen cloud serverless platform.

### Step 1- Install and configure the serverless framework
- [Get started here](https://www.serverless.com/framework/docs/getting-started/#via-npm)
- [Provide the platform with AWS credentials](https://www.serverless.com/framework/docs/providers/aws/guide/credentials/)

### Step 2- Clone the reference implementation repository into your development machine filesystem.
```
git clone https://github.com/dancinnamon-okta/okta-smartfhir-demo.git
cd okta-smartfhir-demo
```

### Step 3- Generate an SSL public/private key that will be used by the token endpoint to authenticate with Okta.
```
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -out public_key.pem -pubout -outform PEM
```
***Note - at this time the file names are hard-coded, so please name them exactly as-is (or submit a PR to make this configurable)***

### Step 4- Install all of the libraries, and prepare for configuration
```
mv serverless.yml.example serverless.yml
npm install
```
### Step 5- Create the authorization server in Okta
In Okta, create a custom authorization server (in the Security->API menu) that you'll be using to authorize users in the demo.
Give the server whatever name you'd like.  Put in a placeholder value in the "audience" field for now- we'll update it later.

Update the serverless.yml with the proper details:
```
AUTHZ_ISSUER: https://_YOUR_ORG_.oktapreview.com/oauth2/_YOUR_AUTHZ_SERVER_
AUTHZ_SERVER: _YOUR_AUTHZ_SERVER_
OKTA_ORG: _YOUR_ORG_.oktapreview.com
```

### Step 6- Create the Patient Picker application in Okta
In Okta, create a new OIDC web application (in the applications menu), using the authorization code flow only.  Remember to assign your users to this app.
Update the serverless.yml file with the proper details:
```
PICKER_DISPLAY_NAME: Patient Picker
PICKER_CLIENT_ID: _CLIENT_ID_FOR_PATIENT_PICKER_
PICKER_CLIENT_SECRET: _CLIENT_SECRET_FOR_PATIENT_PICKER_
```

### Step 7- Create an API key for the Patient Picker
At this time, the Patient Picker application uses an API key to read authorization server details, so we need an API key minted. PR's are welcome to update to use OAuth2 instead of an API key. Use the Security->API->Tokens menu to create this token.

Update the serverless.yml file with the proper details:
```
API_KEY: _AN_API_KEY_
```

### Step 8- Deploy!
To deploy this example, run the following command:
```
serverless deploy -v
```

At this point you should have a number of serverless functions in AWS (until other clouds are supported).  Continue on to setup the rest of the assets in Okta to support this reference implementation.

## Okta Profile Attribute - Patient ID
A key requirement of a SMART/FHIR deployment is the ability to associate a patient id with a user record.  To satisfy this requirement, we need an Okta profile attribute that will hold a patient id for each patient user within the system.
In the Okta profile editor, create a string attribute called "patient_id" as shown:
![Profile Editor Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/profile_editor_sample.png "Profile Editor Example")

## Okta Token Hook Configuration
The token hook is executed at runtime by Okta, and is responsible for ensuring that the custom consent process was properly followed, and in addition it ensures that the consent selections made by the user "which patient, which scopes" are honored.

If an attacker (or curious user) attempts to alter the authorization request data, or bypass the custom consent screen altogether- the token hook will fail validation, and then entire authorization request will fail.

To configure the token hook, use the Workflows->Inline Hooks menu to create a "Token Inline Hook" as shown:
Note- the value you'll use is the URL for your Okta-SMART token endpoint you deployed in the prerequisites.
![Token Hook Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/token_hook_example.png "Token Hook Example")

## Okta Authorization Server Configuration
A key element in this reference SMART/FHIR implementation is an OAuth2 authorization server supplied by Okta. This authorization server was created in the prerequisites when the Okta-SMART endpoints were deployed.  In this section we'll edit that same authorization server.

### Scopes
The first piece of configuration required is to setup the valid SMART authorization scopes in Okta as valid scopes.

The following document contains a sample API call that can be used to create all of the claims/scopes necessary.
<<TODO: Generate this script>>

Here are a few example scopes that can be configured manually:
![Scopes Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/scopes_example.png "Scopes Example")

### Claims
Given that the bulk of the SMART specification relies on OAuth2 (and supports opaque tokens), there are minimal requirements for setting up claims in Okta.

Only two claims are required for this reference implementation:

**Patient Launch Response**
Claim Name: launch_response_patient
Claim Value: user.patient_id
Include with Scope: launch/patient
Include in: Access Token

**FHIR User (OpenID Connect Claim)**
Claim Name: fhirUser
Claim Value: 'Patient/' + user.patient_id
Include with Scope: fhirUser
Include in: ID Token

An example screenshot is shown below:
![Claims Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/claims_example.png "Claims Example")

**Launch Parameter Response Claims**
The SMART launch specification requires any session-level information (such as patient id) to be returned in the /token response alongside the access token.  Okta does not support this out of the box, so the reference implementation includes a /token proxy that will satisfy this requirement.

The token proxy will take any claim in the access token that begins with **_launch_response__** and it will include that claim alongside the access token in the /token response.
For example, a claim in Okta called "launch_response_patient" will cause the token proxy to include a parameter called "patient" in its response alongside the access token.  This works with ANY claim- not only the patient parameter.

### Access Policies
Before any SMART authorizations may take place, an access policy must be created to handle all SMART applications.

**_Note:_**
_There should already be 1 access policy that you created during the prerequisites.  The existing policy should only apply to the custom consent/patient picker application! It is important that this policy you're about to create not apply to the custom consent/patient picker application_

Create a new access policy that will be used for all other OAuth2 clients (except for the custom consent app).
This policy shall have the lowest priority.

Configure the policy/rule as follows and shown:
Applied to Clients: All (for demo purposes)
Allowed Grants: Only Authorization Code
Scopes: All scopes (for demo purposes)
Inline Hook: Select the token hook you created earlier
![Policy Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/access_policy_sample.png "Policy Example")

## SMART Client Registration - Confidential
There are no special considerations for creating confidential SMART clients.  The dynamic client registration protocol may be used, or for demonstration purposes, the Okta admin UI may be used to create a confidential client as shown.

**_Important settings_**:
Application Type: Web
Allowed Grant Types: Authorization Code only
Consent: Not required (it's handled by a custom screen- not Okta default consent screen)
Redirect URI: 2 values shall be put here!
* The smart_proxy_callback URL that you were provided when you deployed the Okta-SMART endpoints.
* The actual redirect_url of the application (this is validated by the /authorize proxy)

![Confidential Client Example](https://github.com/dancinnamon-okta/okta-smartfhir-docs/blob/main/images/confidential_client_example.png "Confidential Client Example")

## SMART Client Registration - Public
Public SMART clients do require some additional configuration over/above what's possible in the Okta admin console.  This is due to the fact that the Okta-SMART /token proxy uses [private_key_jwt](https://developer.okta.com/docs/reference/api/oidc/#jwt-with-private-key) client authentication with Okta.

The easiest way to create one of these clients is to either use dynamic client registration, or use the Okta applications API to create the app.

An example API call is shown below.  An important aspect of the API call is the jwks object.
The JWKS value can be found by visiting the /keys endpoint included in your SMART-Okta endpoints deployed as part of the prerequisites.
```
curl -v -X POST \
-H "Accept: application/json" \
-H "Content-Type: application/json" \
-H "Authorization: SSWS ${api_token}" \
-d '{
		"name": "oidc_client",
		"label": "Sample Public SMART Client",
		"signOnMode": "OPENID_CONNECT",
		"credentials": {
			"oauthClient": {
				"token_endpoint_auth_method": "private_key_jwt"
			}
		},
		"settings": {
			"oauthClient": {
				"client_uri": null,
				"logo_uri": null,
				"redirect_uris": [
					"https://smart-proxy-callback-url",
					"https://actual-app-callback-url"
				],
				"response_types": [
					"code"
				],
				"grant_types": [
					"authorization_code"
				],
				"application_type": "web",
				"consent_method": "TRUSTED",
				"jwks": CONTENT_FROM_KEYS_ENDPOINT_HERE,
		}
}' "https://${yourOktaDomain}/api/v1/apps"
```
