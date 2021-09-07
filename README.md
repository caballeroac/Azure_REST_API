# Azure_REST_API
The goal of this repository is to explain how to make HTTP requests to Azure REST API when we cannot make use of the Azure .NET SDK or Python SKD or others. 

Below we provide two different examples, with curl and python, to get information about the Batch Accounts we have in a subscription. For the HTTP request we will make use to the Azure Batch Management. 

* [Azure Batch Management](https://docs.microsoft.com/en-us/rest/api/batchmanagement/) --> Provides operations for working with the Batch Service
* [Azure Batch Service ](https://docs.microsoft.com/en-us/rest/api/batchservice/) --> To Schedule and run computational workloads

## Pre-requisites

* Azure Subcription


### 1. REST API Requests with Curl

Run `az login`

To authenticate with the API we need the access token. 

The following example will list the Batch accounts in the subcription:

```
subscriptionId='<YOUR_SUBSCRIPTIONID>'

$ curl -sL \
    -H "authorization: bearer $(az account get-access-token --query accessToken -o tsv)" \
    -H "contenttype: application/json" \
    "https://management.azure.com/subscriptions/$subscriptionId/providers/Microsoft.Batch/batchAccounts?api-version=2021-06-01"  |  jq . 
```

The output should be similar to this:

```
{
  "value": [
    {
      "id": "/subscriptions/XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX/resourceGroups/rg-batchdemo/providers/Microsoft.Batch/batchAccounts/XXXXXXXXXXXXX",
      "name": "XXXXXXXXXXXXX,
      "type": "Microsoft.Batch/batchAccounts",
      "location": "westeurope",
      "properties": {
        "accountEndpoint": "XXXXXXXXXXXXXX.westeurope.batch.azure.com",
        "provisioningState": "Succeeded",
        "dedicatedCoreQuota": 828,
        "dedicatedCoreQuotaPerVMFamily": [
          {
            "name": "standardAv2Family",
            "coreQuota": 828
          },
          
          ...
          
          {
            "name": "standardNPSFamily",
            "coreQuota": 0
          },
          {
            "name": "standardFXMDVSFamily",
            "coreQuota": 0
          }
        ],
        "dedicatedCoreQuotaPerVMFamilyEnforced": true,
        "lowPriorityCoreQuota": 500,
        "poolQuota": 100,
        "activeJobAndJobScheduleQuota": 300,
        "autoStorage": {
          "storageAccountId": "/subscriptions/XXXXXXXXXXXXXXXXXXXXXXXXXXXX/resourceGroups/rg-batchdemo/providers/Microsoft.Storage/storageAccounts/XXXXXXXXXXXXXXXXXXXXXXXXX",
          "lastKeySync": "2021-07-15T18:05:54.8476105Z",
          "authenticationMode": "StorageKeys"
        },
        "poolAllocationMode": "BatchService",
        "publicNetworkAccess": "Enabled",
        "encryption": {
          "keySource": "Microsoft.Batch"
        },
        "allowedAuthenticationModes": [
          "SharedKey",
          "AAD",
          "TaskAuthenticationToken"
        ]
      },
      "identity": {
        "type": "None"
      }
    }
  ]
}

```


### 2. REST API Requests with Python (Without Python SDK)

Create a Service Pricipal or use an existing one. 

Run `az login` and create a new Service Principal with `az ad sp create-for-rbac --name YourServicePrincipalName`

For the Python example we will use Adal and requests libraries.
```
pip install adal
pip install requests
```

```
#!/usr/bin/env python3

import adal
import json
import requests

subscriptionId='<YOUR_SUBSCRIPTIONID>'
tenant_id='<YOUR_TENANT_ID>'
client_id='<YOUR_CLIENT_ID>'
client_secret='<YOUR_CLIENT_SECRET>'

def get_auth_header():

    url = 'https://login.microsoftonline.com' + '/' + tenant_id
    context = adal.AuthenticationContext(url)
    token = context.acquire_token_with_client_credentials('https://management.core.windows.net/', client_id, client_secret)

    authHeader = {
        'Authorization': 'Bearer ' + token['accessToken'],
        'Content-Type': 'application/json'
    }

    return authHeader

# Funtion to list Batch accounts in the subscription
def batch_acc_list(_auth_header_token):
   
   uri = 'https://management.azure.com/subscriptions/' + subscriptionId + '/providers/Microsoft.Batch/batchAccounts?api-version=2021-06-01'
   response = requests.get(uri, headers=_auth_header_token)
   jsonData = response.json()
   print(json.dumps(jsonData, indent=4))

# Create Batch Account
def batch_acc_create(_auth_header_token, _accountName, _resourceGroupName):

   body = { "location": "westeurope",
                   "properties": {
                     "autoStorage": {
                               "storageAccountId": "/subscriptions/XXXXXXXXXX/resourceGroups/rg-batchdemo/providers/Microsoft.Storage/storageAccounts/samplestorage"
                     }
                   },
                   "identity": {
                       "type": "SystemAssigned"
                   }
                 }

   json_object = json.dumps(body)

   uri = 'https://management.azure.com/subscriptions/' + subscriptionId + '/resourceGroups/' + _resourceGroupName + '/providers/Microsoft.Batch/batchAccounts/' + _accountName + '?api-version=2021-06-01'


   response = requests.put(uri, data=json_object, headers=_auth_header_token)
   jsonData = response.json()

   print(json.dumps(jsonData, indent=4))


# Delete Batch Account
def batch_acc_delete(_auth_header_token, _accountName, _resourceGroupName):

   uri = 'https://management.azure.com/subscriptions/' + subscriptionId + '/resourceGroups/' + _resourceGroupName + '/providers/Microsoft.Batch/batchAccounts/' + _accountName + '?api-version=2021-06-01'

   response = requests.delete(uri, headers=_auth_header_token)
   print(response)




def main():

   # BATCH ACCOUNT PARAMETERS
   accountName = 'ancabatchdemo'
   resourceGroupName = 'rg-batchdemo'
   poolName = 'demoPool'

   auth_header_token=get_auth_header()

   # LIST BATCH ACCOUNTS
   #account_list = batch_acc_list(auth_header_token)
   #print(account_list)

   # CREATE BATCH ACCOUNT
   new_account =  batch_acc_create(auth_header_token,accountName,resourceGroupName)


   # CREATE A NEW POOL
   # PUT https://management.azure.com/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.Batch/batchAccounts/{accountName}/pools/{poolName}?api-version=2021-06-01
   # new_pool = batch_create_pool(auth_header_token,accountName,resourceGroupName,poolName)

   # DELETE BATCH ACCOUNT
   delete_account =  batch_acc_delete(auth_header_token,accountName,resourceGroupName)

if __name__ == '__main__':

   main()

```

