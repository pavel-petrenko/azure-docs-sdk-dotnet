---
title: Azure Key Vault key client library for .NET
keywords: Azure, dotnet, SDK, API, Azure.Security.KeyVault.Keys, keyvault
ms.date: 04/09/2025
ms.topic: reference
ms.devlang: dotnet
ms.service: keyvault
---
# Azure Key Vault key client library for .NET - version 4.8.0-beta.1 


Azure Key Vault is a cloud service that provides secure storage of keys for encrypting your data.
Multiple keys, and multiple versions of the same key, can be kept in the Azure Key Vault.
Cryptographic keys in Azure Key Vault are represented as [JSON Web Key (JWK)][JWK] objects.

Azure Key Vault Managed HSM is a fully-managed, highly-available, single-tenant, standards-compliant cloud service that enables
you to safeguard cryptographic keys for your cloud applications using FIPS 140-2 Level 3 validated HSMs.

The Azure Key Vault keys library client supports RSA keys and Elliptic Curve (EC) keys, each with corresponding support in hardware security modules (HSM). It offers operations to create, retrieve, update, delete, purge, backup, restore, and list the keys and its versions.

[Source code][key_client_src] | [Package (NuGet)][key_client_nuget_package] | [API reference documentation][API_reference] | [Product documentation][keyvault_docs] | [Samples][key_client_samples] | [Migration guide][migration_guide]

## Getting started

### Install the package

Install the Azure Key Vault keys client library for .NET with [NuGet][nuget]:

```dotnetcli
dotnet add package Azure.Security.KeyVault.Keys
```

### Prerequisites

* An [Azure subscription][azure_sub].
* An existing Azure Key Vault. If you need to create an Azure Key Vault, you can use the Azure Portal or [Azure CLI][azure_cli].
* Authorization to an existing Azure Key Vault using either [RBAC][rbac_guide] (recommended) or [access control][access_policy].

If you are creating a standard Key Vault resource, run the following CLI command replacing `<your-resource-group-name>` and `<your-key-vault-name>` with your own, unique names:

```PowerShell
az keyvault create --resource-group <your-resource-group-name> --name <your-key-vault-name>
```

If you are creating a Managed HSM resource, run the following CLI command:

```PowerShell
az keyvault create --hsm-name <your-key-vault-name> --resource-group <your-resource-group-name> --administrators <your-user-object-id> --location <your-azure-location>
```

To get `<your-user-object-id>` you can run the following CLI command:

```PowerShell
az ad user show --id <your-user-principal> --query id
```

### Authenticate the client

In order to interact with the Azure Key Vault service, you'll need to create an instance of the [KeyClient][key_client_class] class. You need a **vault url**, which you may see as "DNS Name" in the portal,
and credentials to instantiate a client object.

The examples shown below use a [`DefaultAzureCredential`][DefaultAzureCredential], which is appropriate for most scenarios including local development and production environments utilizing managed identity authentication.
Additionally, we recommend using a managed identity for authentication in production environments.
You can find more information on different ways of authenticating and their corresponding credential types in the [Azure Identity][azure_identity] documentation.

To use the `DefaultAzureCredential` provider shown below,
or other credential providers provided with the Azure SDK, you must first install the Azure.Identity package:

```dotnetcli
dotnet add package Azure.Identity
```

#### Activate your managed HSM

This section only applies if you are creating a Managed HSM. All data plane commands are disabled until the HSM is activated. You will not be able to create keys or assign roles.
Only the designated administrators that were assigned during the create command can activate the HSM. To activate the HSM you must download the security domain.

To activate your HSM you need:

* Minimum 3 RSA key-pairs (maximum 10)
* Specify minimum number of keys required to decrypt the security domain (quorum)

To activate the HSM you send at least 3 (maximum 10) RSA public keys to the HSM. The HSM encrypts the security domain with these keys and sends it back.
Once this security domain is successfully downloaded, your HSM is ready to use.
You also need to specify quorum, which is the minimum number of private keys required to decrypt the security domain.

The example below shows how to use openssl to generate 3 self-signed certificate.

```PowerShell
openssl req -newkey rsa:2048 -nodes -keyout cert_0.key -x509 -days 365 -out cert_0.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_1.key -x509 -days 365 -out cert_1.cer
openssl req -newkey rsa:2048 -nodes -keyout cert_2.key -x509 -days 365 -out cert_2.cer
```

Use the `az keyvault security-domain download` command to download the security domain and activate your managed HSM.
The example below uses 3 RSA key pairs (only public keys are needed for this command) and sets the quorum to 2.

```PowerShell
az keyvault security-domain download --hsm-name <your-key-vault-name> --sd-wrapping-keys ./certs/cert_0.cer ./certs/cert_1.cer ./certs/cert_2.cer --sd-quorum 2 --security-domain-file ContosoMHSM-SD.json
```

#### Create KeyClient

Instantiate a `DefaultAzureCredential` to pass to the client.
The same instance of a token credential can be used with multiple clients if they will be authenticating with the same identity.

```C# Snippet:CreateKeyClient
// Create a new key client using the default credential from Azure.Identity using environment variables previously set,
// including AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, and AZURE_TENANT_ID.
var client = new KeyClient(vaultUri: new Uri(vaultUrl), credential: new DefaultAzureCredential());

// Create a new key using the key client.
KeyVaultKey key = client.CreateKey("key-name", KeyType.Rsa);

// Retrieve a key using the key client.
key = client.GetKey("key-name");
```

#### Create CryptographyClient

Once you've created a `KeyVaultKey` in the Azure Key Vault, you can also create the [CryptographyClient][crypto_client_class]:

```C# Snippet:CreateCryptographyClient
// Create a new cryptography client using the same Key Vault or Managed HSM endpoint, service version,
// and options as the KeyClient created earlier.
CryptographyClient cryptoClient = client.GetCryptographyClient(key.Name, key.Properties.Version);
```

## Key concepts

### KeyVaultKey

Azure Key Vault supports multiple key types and algorithms, and enables the use of hardware security modules (HSM) for high value keys.

### KeyClient

A `KeyClient` providing both synchronous and asynchronous operations exists in the SDK allowing for selection of a client based on an application's use case. Once you've initialized a `KeyClient`, you can interact with the primary resource types in Azure Key Vault.

### CryptographyClient

A `CryptographyClient` providing both synchronous and asynchronous operations exists in the SDK allowing for selection of a client based on an application's use case. Once you've initialized a `CryptographyClient`, you can use it to perform cryptographic operations with keys stored in Azure Key Vault.

### Thread safety

We guarantee that all client instance methods are thread-safe and independent of each other ([guideline](https://azure.github.io/azure-sdk/dotnet_introduction.html#dotnet-service-methods-thread-safety)). This ensures that the recommendation of reusing client instances is always safe, even across threads.

### Additional concepts
<!-- CLIENT COMMON BAR -->
[Client options](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/core/Azure.Core/README.md#configuring-service-clients-using-clientoptions) |
[Accessing the response](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/core/Azure.Core/README.md#accessing-http-response-details-using-responset) |
[Long-running operations](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/core/Azure.Core/README.md#consuming-long-running-operations-using-operationt) |
[Handling failures](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/core/Azure.Core/README.md#reporting-errors-requestfailedexception) |
[Diagnostics](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/core/Azure.Core/samples/Diagnostics.md) |
[Mocking](https://learn.microsoft.com/dotnet/azure/sdk/unit-testing-mocking) |
[Client lifetime](https://devblogs.microsoft.com/azure-sdk/lifetime-management-and-thread-safety-guarantees-of-azure-sdk-net-clients/)
<!-- CLIENT COMMON BAR -->

## Examples

The Azure.Security.KeyVault.Keys package supports synchronous and asynchronous APIs.

The following section provides several code snippets using the `client` [created above](#create-keyclient), covering some of the most common Azure Key Vault key service related tasks:

### Sync examples

* [Create a key](#create-a-key)
* [Retrieve a key](#retrieve-a-key)
* [Update an existing key](#update-an-existing-key)
* [Delete a key](#delete-a-key)
* [Delete and purge a key](#delete-and-purge-a-key)
* [List keys](#list-keys)
* [Encrypt and Decrypt](#encrypt-and-decrypt)

### Async examples

* [Create a key asynchronously](#create-a-key-asynchronously)
* [List keys asynchronously](#list-keys-asynchronously)
* [Delete a key asynchronously](#delete-a-key-asynchronously)

### Create a key

Create a key to be stored in the Azure Key Vault. If a key with the same name already exists, then a new version of the key is created.

```C# Snippet:CreateKey
// Create a key. Note that you can specify the type of key
// i.e. Elliptic curve, Hardware Elliptic Curve, RSA
KeyVaultKey key = client.CreateKey("key-name", KeyType.Rsa);

Console.WriteLine(key.Name);
Console.WriteLine(key.KeyType);

// Create a software RSA key
var rsaCreateKey = new CreateRsaKeyOptions("rsa-key-name", hardwareProtected: false);
KeyVaultKey rsaKey = client.CreateRsaKey(rsaCreateKey);

Console.WriteLine(rsaKey.Name);
Console.WriteLine(rsaKey.KeyType);

// Create a hardware Elliptic Curve key
// Because only premium Azure Key Vault supports HSM backed keys , please ensure your Azure Key Vault
// SKU is premium when you set "hardwareProtected" value to true
var echsmkey = new CreateEcKeyOptions("ec-key-name", hardwareProtected: true);
KeyVaultKey ecKey = client.CreateEcKey(echsmkey);

Console.WriteLine(ecKey.Name);
Console.WriteLine(ecKey.KeyType);
```

### Retrieve a key

`GetKey` retrieves a key previously stored in the Azure Key Vault.

```C# Snippet:RetrieveKey
KeyVaultKey key = client.GetKey("key-name");

Console.WriteLine(key.Name);
Console.WriteLine(key.KeyType);
```

### Update an existing key

`UpdateKeyProperties` updates a key previously stored in the Azure Key Vault.

```C# Snippet:UpdateKey
KeyVaultKey key = client.CreateKey("key-name", KeyType.Rsa);

// You can specify additional application-specific metadata in the form of tags.
key.Properties.Tags["foo"] = "updated tag";

KeyVaultKey updatedKey = client.UpdateKeyProperties(key.Properties);

Console.WriteLine(updatedKey.Name);
Console.WriteLine(updatedKey.Properties.Version);
Console.WriteLine(updatedKey.Properties.UpdatedOn);
```

### Delete a key

`StartDeleteKey` starts a long-running operation to delete a key previously stored in the Azure Key Vault.
You can retrieve the key immediately without waiting for the operation to complete.
When [soft-delete][soft_delete] is not enabled for the Azure Key Vault, this operation permanently deletes the key.

```C# Snippet:DeleteKey
DeleteKeyOperation operation = client.StartDeleteKey("key-name");

DeletedKey key = operation.Value;
Console.WriteLine(key.Name);
Console.WriteLine(key.DeletedOn);
```

### Delete and purge a key

You will need to wait for the long-running operation to complete before trying to purge or recover the key.

```C# Snippet:DeleteAndPurgeKey
DeleteKeyOperation operation = client.StartDeleteKey("key-name");

// You only need to wait for completion if you want to purge or recover the key.
while (!operation.HasCompleted)
{
    Thread.Sleep(2000);

    operation.UpdateStatus();
}

DeletedKey key = operation.Value;
client.PurgeDeletedKey(key.Name);
```

### List Keys

This example lists all the keys in the specified Azure Key Vault.

```C# Snippet:ListKeys
Pageable<KeyProperties> allKeys = client.GetPropertiesOfKeys();

foreach (KeyProperties keyProperties in allKeys)
{
    Console.WriteLine(keyProperties.Name);
}
```

### Encrypt and Decrypt

This example creates a `CryptographyClient` and uses it to encrypt and decrypt with a key in Azure Key Vault.

```C# Snippet:EncryptDecrypt
// Create a new cryptography client using the same Key Vault or Managed HSM endpoint, service version,
// and options as the KeyClient created earlier.
var cryptoClient = client.GetCryptographyClient(key.Name, key.Properties.Version);

byte[] plaintext = Encoding.UTF8.GetBytes("A single block of plaintext");

// encrypt the data using the algorithm RSAOAEP
EncryptResult encryptResult = cryptoClient.Encrypt(EncryptionAlgorithm.RsaOaep256, plaintext);

// decrypt the encrypted data.
DecryptResult decryptResult = cryptoClient.Decrypt(EncryptionAlgorithm.RsaOaep256, encryptResult.Ciphertext);
```

### Create a key asynchronously

The asynchronous APIs are identical to their synchronous counterparts, but return with the typical "Async" suffix for asynchronous methods and return a Task.

```C# Snippet:CreateKeyAsync
// Create a key of any type
KeyVaultKey key = await client.CreateKeyAsync("key-name", KeyType.Rsa);

Console.WriteLine(key.Name);
Console.WriteLine(key.KeyType);

// Create a software RSA key
var rsaCreateKey = new CreateRsaKeyOptions("rsa-key-name", hardwareProtected: false);
KeyVaultKey rsaKey = await client.CreateRsaKeyAsync(rsaCreateKey);

Console.WriteLine(rsaKey.Name);
Console.WriteLine(rsaKey.KeyType);

// Create a hardware Elliptic Curve key
// Because only premium Azure Key Vault supports HSM backed keys , please ensure your Azure Key Vault
// SKU is premium when you set "hardwareProtected" value to true
var echsmkey = new CreateEcKeyOptions("ec-key-name", hardwareProtected: true);
KeyVaultKey ecKey = await client.CreateEcKeyAsync(echsmkey);

Console.WriteLine(ecKey.Name);
Console.WriteLine(ecKey.KeyType);
```

### List keys asynchronously

Listing keys does not rely on awaiting the `GetPropertiesOfKeysAsync` method, but returns an `AsyncPageable<KeyProperties>` that you can use with the `await foreach` statement:

```C# Snippet:ListKeysAsync
AsyncPageable<KeyProperties> allKeys = client.GetPropertiesOfKeysAsync();

await foreach (KeyProperties keyProperties in allKeys)
{
    Console.WriteLine(keyProperties.Name);
}
```

### Delete a key asynchronously

When deleting a key asynchronously before you purge it, you can await the `WaitForCompletionAsync` method on the operation.
By default, this loops indefinitely but you can cancel it by passing a `CancellationToken`.

```C# Snippet:DeleteAndPurgeKeyAsync
DeleteKeyOperation operation = await client.StartDeleteKeyAsync("key-name");

// You only need to wait for completion if you want to purge or recover the key.
await operation.WaitForCompletionAsync();

DeletedKey key = operation.Value;
await client.PurgeDeletedKeyAsync(key.Name);
```

## Troubleshooting

See our [troubleshooting guide](https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/TROUBLESHOOTING.md)
for details on how to diagnose various failure scenarios.

### General

When you interact with the Azure Key Vault key client library using the .NET SDK, errors returned by the service correspond to the same HTTP status codes returned for [REST API][keyvault_rest] requests.

For example, if you try to retrieve a key that doesn't exist in your Azure Key Vault, a `404` error is returned, indicating "Not Found".

```C# Snippet:KeyNotFound
try
{
    KeyVaultKey key = client.GetKey("some_key");
}
catch (RequestFailedException ex)
{
    Console.WriteLine(ex.ToString());
}
```

You will notice that additional information is logged, like the client request ID of the operation.

```text
Message:
    Azure.RequestFailedException : Service request failed.
    Status: 404 (Not Found)
Content:
    {"error":{"code":"KeyNotFound","message":"Key not found: some_key"}}

Headers:
    Cache-Control: no-cache
    Pragma: no-cache
    Server: Microsoft-IIS/10.0
    x-ms-keyvault-region: westus
    x-ms-request-id: 625f870e-10ea-41e5-8380-282e5cf768f2
    x-ms-keyvault-service-version: 1.1.0.866
    x-ms-keyvault-network-info: addr=131.107.174.199;act_addr_fam=InterNetwork;
    X-AspNet-Version: 4.0.30319
    X-Powered-By: ASP.NET
    Strict-Transport-Security: max-age=31536000;includeSubDomains
    X-Content-Type-Options: nosniff
    Date: Tue, 18 Jun 2019 16:02:11 GMT
    Content-Length: 75
    Content-Type: application/json; charset=utf-8
    Expires: -1
```

## Next steps

Several Azure Key Vault keys client library samples are available to you in this GitHub repository. These samples provide example code for additional scenarios commonly encountered while working with Azure Key Vault:

* [Sample1_HelloWorld.md][hello_world_sample] - for working with Azure Key Vault, including:
  * Create a key
  * Get an existing key
  * Update an existing key
  * Delete a key

* [Sample2_BackupAndRestore.md][backup_and_restore_sample] - Contains the code snippets working with Azure Key Vault keys, including:
  * Backup and recover a key

* [Sample3_GetKeys.md][get_keys_sample] - Example code for working with Azure Key Vault keys, including:
  * Create keys
  * List all keys in the Key Vault
  * Update keys in the Key Vault
  * List versions of a specified key
  * Delete keys from the Key Vault
  * List deleted keys in the Key Vault

* [Sample4_EncryptDecrypt.md][encrypt_decrypt_sample] - Example code for performing cryptographic operations with Azure Key Vault keys, including:
  * Encrypt and Decrypt data with the CryptographyClient

* [Sample5_SignVerify.md][sign_verify_sample] - Example code for working with Azure Key Vault keys, including:
  * Sign a precalculated digest and verify the signature with Sign and Verify
  * Sign raw data and verify the signature with SignData and VerifyData

* [Sample6_WrapUnwrap.md][wrap_unwrap_sample] - Example code for working with Azure Key Vault keys, including:
  * Wrap and Unwrap a symmetric key

### Additional Documentation

* For more extensive documentation on Azure Key Vault, see the [API reference documentation][keyvault_rest].
* For Secrets client library see [Secrets client library][secrets_client_library].
* For Certificates client library see [Certificates client library][certificates_client_library].

## Contributing

See the [CONTRIBUTING.md][contributing] for details on building, testing, and contributing to these libraries.

This project welcomes contributions and suggestions. Most contributions require you to agree to a Contributor License Agreement (CLA)
declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit <https://cla.microsoft.com>.

When you submit a pull request, a CLA-bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., label, comment).
Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct][code_of_conduct].
For more information see the [Code of Conduct FAQ][coc_faq] or contact <opencode@microsoft.com> with any additional questions or comments.

<!-- LINKS -->
[API_reference]: https://learn.microsoft.com/dotnet/api/azure.security.keyvault.keys
[azure_cli]: https://learn.microsoft.com/cli/azure
[azure_identity]: https://github.com/Azure/azure-sdk-for-net/tree/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/identity/Azure.Identity
[azure_sub]: https://azure.microsoft.com/free/dotnet/
[backup_and_restore_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample2_BackupAndRestore.md
[certificates_client_library]: https://github.com/Azure/azure-sdk-for-net/tree/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Certificates
[code_of_conduct]: https://opensource.microsoft.com/codeofconduct/
[get_keys_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample3_GetKeys.md
[encrypt_decrypt_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample4_EncryptDecrypt.md
[sign_verify_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample5_SignVerify.md
[wrap_unwrap_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample6_WrapUnwrap.md
[hello_world_sample]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples/Sample1_HelloWorld.md
[key_client_class]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/src/KeyClient.cs
[crypto_client_class]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/src/Cryptography/CryptographyClient.cs
[key_client_nuget_package]: https://www.nuget.org/packages/Azure.Security.KeyVault.Keys/
[key_client_samples]: https://github.com/Azure/azure-sdk-for-net/tree/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/samples
[key_client_src]: https://github.com/Azure/azure-sdk-for-net/tree/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/src
[keyvault_docs]: https://learn.microsoft.com/azure/key-vault/
[keyvault_rest]: https://learn.microsoft.com/rest/api/keyvault/
[JWK]: https://tools.ietf.org/html/rfc7517
[nuget]: https://www.nuget.org/
[secrets_client_library]: https://github.com/Azure/azure-sdk-for-net/tree/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Secrets
[soft_delete]: https://learn.microsoft.com/azure/key-vault/general/soft-delete-overview
[DefaultAzureCredential]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/identity/Azure.Identity/README.md#defaultazurecredential
[contributing]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/CONTRIBUTING.md
[coc_faq]: https://opensource.microsoft.com/codeofconduct/faq/
[migration_guide]: https://github.com/Azure/azure-sdk-for-net/blob/Azure.Security.KeyVault.Keys_4.8.0-beta.1/sdk/keyvault/Azure.Security.KeyVault.Keys/MigrationGuide.md
[access_policy]: https://learn.microsoft.com/azure/key-vault/general/assign-access-policy
[rbac_guide]: https://learn.microsoft.com/azure/key-vault/general/rbac-guide

