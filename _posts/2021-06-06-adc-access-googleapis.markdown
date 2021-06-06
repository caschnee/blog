---
layout: post
title:  "Access Google APIs with Compute resources' attached service account"
date:   2021-06-06 -0400
categories: gcp security iam
---

## TL;DR
Using GCP resources' [attached service accounts](https://cloud.google.com/iam/docs/impersonating-service-accounts#attaching-to-resources) to authenticate your application running in GCP frees you from managing and storing secrets. However authenticating to certain googleapis such as the [Admin Directory API](https://developers.google.com/admin-sdk/directory) or the [Gmail API](https://developers.google.com/gmail/api) is not enabled by default, even if the service account has permission to access those APIs.

GCP limits the [OAuth scopes](https://oauth.net/2/scope/) when using service accounts attached to Compute Engine or Cloud Functions. The Cloud Functions' default scopes are broader than Compute Engine but lack customization. In Compute Engine, however, adding custom OAuth scopes; e.g. to access the Admin Directory API; is possible.

## Why use service accounts?
The traditional way of authenticating an application is through the distribution and management of "long-lived" secrets. This approach raises numerous challenges, including distributing, storing, securing, and rotating those secrets.

In the cloud, however, the [recommended way](https://cloud.google.com/docs/authentication#strategies) to authenticate your workloads is through their attached service accounts. Every cloud workload; from Compute Instances to Cloud Functions; runs under a service account identity. GCP takes care of access token rotation and frees you from managing secrets. This method is called [**ADC (Application Default Credential)**](https://cloud.google.com/docs/authentication/production).

Cloud service accounts have their limit though, they do not support authenticating to a third-party API, or any APIs not supported by the cloud provider. Reaching those limits is very common in hybrid or multi-cloud environments for example. 

Note that default service accounts have broad permissions in GCP. As a best practice, consider creating custom service accounts for resources and limit their permissions on a least privilege basis. Also, consider organization policy to [limit the default service accounts permissions](https://cloud.google.com/resource-manager/docs/organization-policy/restricting-service-accounts#disable_service_account_default_grants). 

Solutions to overcome those limits include using a secret manager to store third-party secrets and provide the resources' attached service account access to the specific secret manager's secret. However, rotating, distributing, and managing access to those third-party secrets still need to be taken care of.

So your resource has a service account, great! Google libraries and SDK provide seamless support for [using the attached service account](https://cloud.google.com/docs/authentication/production#automatically) to authenticate your application. But how does the authentication mechanism work under the hood?

## The metadata server 
Compute Instance metadata can be [stored in a metadata server](https://cloud.google.com/compute/docs/storing-retrieving-metadata) provided by GCP. "Metadata" also includes service account information. In addition, the metadata server provides an endpoint to request a short-lived OAuth bearer token for the attached service account. The metadata server can be accessed with a simple get request from the Compute Instance SSH shell as shown below. Note that the `metadata.google.internal` endpoint is only accessible from inside GCP resources, including Cloud Shell and Cloud Functions.

```
curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/<SERVICE_ACCOUNT_EMAIL>/token" -H "Metadata-Flavor: Google"
{"access_token":"REDACTED","expires_in":3599,"token_type":"Bearer"}
```

The bearer token is only valid for a limited time and for a specific set of OAuth scopes. Another metadata server endpoint lists the OAuth scopes given to this bearer token:

```
curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/<SERVICE_ACCOUNT_EMAIL>/scopes" -H "Metadata-Flavor: Google"
https://www.googleapis.com/auth/cloud-platform
```

With this token, it is then possible to call a Google API; on the following conditions:

* The service account attached with the resource has appropriate permissions to access this API. For example, see [here](https://cloud.google.com/identity/docs/how-to/setup#auth-no-dwd
) on how to provide service account permissions to access Google Workspace API, including the Admin Directory API.
* The requested API scope is included in the metadata server's scopes. Refer to the list of [all OAuth scopes for Google APIs](https://developers.google.com/identity/protocols/oauth2/scopes).


## OAuth scopes in Compute Engine
With a custom service account, the default scope is `https://www.googleapis.com/auth/cloud-platform` and cannot be changed from the UI. With this configuration, the Admin Directory API and the Gmail API cannot be accessed for example. The [gcloud command to set service account and scopes for a Compute Engine instance](https://cloud.google.com/sdk/gcloud/reference/compute/instances/set-service-account), however, enables us to customize the `scopes` flag.

Using the gcloud command, we can set a custom scope for our service account:

```
gcloud compute instances set-service-account <COMPUTE_INSTANCE_NAME> \
    --service-account <SERVICE_ACCOUNT_EMAIL> \
    --zone=us-central1-a \
    --scopes "https://www.googleapis.com/auth/admin.directory.group.readonly"
```

Querying the metadata server returns the expected scope.

```
curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/<SERVICE_ACCOUNT_EMAIL>/scopes" -H "Metadata-Flavor: Google"
https://www.googleapis.com/auth/admin.directory.group.readonly
```

We can then successfully use the service account bearer token returned by the metadata server to query the Admin Directory API:

```
curl "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/<SERVICE_ACCOUNT_EMAIL>/token" -H "Metadata-Flavor: Google"

curl \
  'https://admin.googleapis.com/admin/directory/v1/groups?domain=<DOMAIN>' \
  --header 'Authorization: Bearer <BEARER_TOKEN>' \
  --header 'Accept: application/json' \
  --compressed
```

## OAuth scope in Cloud Functions

Cloud Functions are serverless workloads, hence typically provide less infrastructure customizations. This is also true for the OAuth scopes, as changing the Cloud Functions scopes does not seem to be possible. We'll have to do with the default ones (listed below), which are though broader than the default Compute Engine scopes.

```
https://mail.google.com/
https://www.googleapis.com/auth/analytics
https://www.googleapis.com/auth/calendar
https://www.googleapis.com/auth/cloud-platform
https://www.googleapis.com/auth/contacts
https://www.googleapis.com/auth/drive
https://www.googleapis.com/auth/presentations
https://www.googleapis.com/auth/spreadsheets
https://www.googleapis.com/auth/streetviewpublish
https://www.googleapis.com/auth/urlshortener
https://www.googleapis.com/auth/userinfo.email
https://www.googleapis.com/auth/youtube
```

The list above has been extracted with the below script (run in a Cloud Function).

{% highlight python %}
import requests 

def hello_world(request):
 
    headers = {
        "Metadata-Flavor": "Google"
    }
 
    r = requests.get("http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/scopes",headers=headers)
    print(r.text)

{% endhighlight %}

## Conclusion
Using Compute resources' attached Service accounts with ADC libraries is a nice way to avoid managing secrets in a cloud environment but have some limits, especially for serverless offerings. 

## Related links

* [Authenticating as a service account](https://cloud.google.com/docs/authentication/production)
* [The 2 limits of Google Cloud IAM service](https://medium.com/google-cloud/the-2-limits-of-iam-service-on-google-cloud-7db213277d9c)
* [OAuth 2.0 Scopes for Google APIs](https://developers.google.com/identity/protocols/oauth2/scopes)
* [Best practices for securing service accounts](https://cloud.google.com/iam/docs/best-practices-for-securing-service-accounts)
* [Google OAuth credential: going deeper, the hard way](https://medium.com/google-cloud/google-oauth-credential-going-deeper-the-hard-way-f403cf3edf9d)