---

copyright:
  year: 2018, 2021
lastupdated: "2021-06-08"

keywords: security, authentication, access, iam, sso, service credential

subcollection: sql-query

---

{:shortdesc: .shortdesc}
{:new_window: target="_blank"}
{:codeblock: .codeblock}
{:pre: .pre}
{:screen: .screen}
{:tip: .tip}


# Authentication
{: #authentication}

The {{site.data.keyword.sqlquery_full}} service is tightly integrated with {{site.data.keyword.iamlong}} (IAM). 
In order to perform an action by using the {{site.data.keyword.sqlquery_short}} user interface or API, you require an IAM user ID, which is an IBM ID with an IBM Cloud account. {{site.data.keyword.sqlquery_short}} then verifies whether you are authorized to work with the service instance.

## {{site.data.keyword.iamshort}} session support
{: #iam}

The {{site.data.keyword.sqlquery_short}} UI supports {{site.data.keyword.iamshort}} sessions. If you limit the duration of active sessions, the authentication of the UI will stop after the session expires, so you cannot submit new queries or retrieve result data until you reload the page and re-authenticate. Result data that is already cached in the browser remains visible. If you terminate an active IAM session before it expires, the {{site.data.keyword.sqlquery_short}} UI still remains authenticated for the leftover lifetime of the current access token, which can be up to 20 minutes.

Queries submitted from the UI can run at maximum for one hour or the remaining session lifetime, whichever is shorter. Queries submitted via the API can always run for one hour maximum if you retrieve the API token using the method described in the SQL Query API documentation.

## Authenticating access to data resources in SQL Query
{: #accessauthentication}

### SSO through {{site.data.keyword.iamlong}} (IAM)
{: #iamsso}

IAM is also the preferred and the default mechanism for authenticating access to downstream data resources that are referenced by your SQL queries. If you use IAM, ensure that the user ID that you use to submit the SQL query through the web console or the API is authorized to read the input locations (referenced in the `FROM` clauses of the query), and to write the result set to the target location (referenced in the `INTO` clause of the query). 
This mechanism provides seamless single sign-on (SSO) for your query submissions in {{site.data.keyword.sqlquery_short}}. 
You can use the following alternative methods for authenticating access to data resources:

#### IBM Cloud service credentials
{: #servicecredentials}

You can use the unique CRN of a service instance in {{site.data.keyword.Bluemix_notm}} to address a resource location. 
The credentials for accessing this data resource are retrieved from the Credentials object of that service instance. 
The IAM user ID that is used to submit the query must have the operator role for the service instance of the corresponding data resource.

#### Custom user ID and password for each data resource
{: #userpassword}

You can use the USER and PASSWORD keywords in the corresponding FROM or INTO clauses to securely pass user ID and password for each resource location. 
To ensure that the passing of sensitive data is secure, store the password as a custom standard key in an instance of {{site.data.keyword.keymanagementserviceshort}} to which you have access and then use the CRN of this custom standard key instead of the plain text password.
For more information, see [Setting up custom secrets in Key Protect](#kpsetup) about how to store the password securely.

#### Custom API key for each data resource
{: #apikey}

As an alternative to providing user and password combinations, in {{site.data.keyword.Bluemix_notm}} you can also securely pass API keys for each resource location. Using the `APIKEY` keyword inside the according `FROM` or `INTO` clauses, you can pass in this information. 
To ensure that the passing of sensitive data is secure, first store the API key as a custom standard key in a 
{{site.data.keyword.keymanagementserviceshort}} service instance to which you have access, and then use the CRN of this custom standard key instead of the plain text password. For more information, see [Setting up custom secrets in Key Protect](#kpsetup) on how to store the API key securely.

### Currently supported authentication methods per data resource for {{site.data.keyword.sqlquery_short}}
{: #supportedauthentication}

| Authentication method                 | {{site.data.keyword.cos_full}}   | {{site.data.keyword.Db2_on_Cloud_long}}  | {{site.data.keyword.dashdblong}} |
| ---                                   | :---: | :---:        | :---:                  |
| IAM SSO                               | Yes   | Enterprise   | Enterprise             |
| CRN with Service Credentials          | No    | Yes          | Yes                    |
| User and password through Key Protect CRN | No    | Yes          | Yes                    |
| API key through Key Protect CRN           | No    | Enterprise   | Enterprise             |
