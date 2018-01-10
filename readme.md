# Key Vault enhanced message security sample using the Azure Python SDK

This sample demonstrates communications to a Key Vault with enhanced message protection enabled.  The sample connects to a pre-existing vault which enables this feature, creates a key in that vault which it will use to wrap and unwrap an AES key created by the client.

## Running the sample:
This sample requires changes to the azure-keyvault Python SDK package which are not currently, published.  These changes are available in a fork of the python sdk repo, https://github.com/schaabs/azure-sdk-for-python/tree/mcrypt.  Also a private development whl package containing these changes is included with the sample 'azure_keyvault-0.3.8+dev-py2.py3-none-any.whl'.

1. pip install 'azure_keyvault-0.3.8+dev-py2.py3-none-any.whl'
2. python mcyprt_