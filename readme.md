# Key Vault message integrity and privacy (aka message encryption) sample using the Azure Python SDK 

This sample demonstrates communications to a Key Vault with message encryption enabled.  The sample connects to a pre-existing vault which enables this feature, creates a key in that vault which it will use to wrap and unwrap an AES key created by the client.

## Running the sample:
This sample requires changes to the azure-keyvault Python SDK package which are not currently, published.  These changes are available in a fork of the python sdk repo, https://github.com/schaabs/azure-sdk-for-python/tree/mcrypt.  

Also a private development whl package containing these changes is included with the sample 'azure_keyvault-0.3.8+dev-py2.py3-none-any.whl'.

1. pip install 'azure_keyvault-0.3.8+dev-py2.py3-none-any.whl'
2. set the needed environment variables

        AZURE_VAULT_URL=<EMS enabled vault url>
        AZURE_CLIENT_ID=<client id with access to the vault>
        AZURE_CLIENT_SECRET=<client secret>

3. python mcrypt_sample.py

## Implementation Notes
Communicating with a vault where message encryption is enabled should be fairly transparent to the code which is communicating with the vault.  Messages which need to be protected will be protected by the Key Vault SDK before they are sent to the service, and likewise responses will be unprotected in the SDK before the result is handed back.  Therefore code which calls methods that use message encryption, such as wrap and unwrap is the same regardless of whether message encryption is enabled.

mcrypt_samply.py L95

    # create a random AES key to wrap
    to_wrap = os.urandom(16)
    print('\noriginal value:')
    print(to_wrap)

    # wrap the key
    wrapped = client.wrap_key(vault_base_url=_get_vault_url(),
                              key_name=key_name,
                              key_version='',
                              algorithm='RSA-OAEP',
                              value=to_wrap)
    print('\nwrap_key result:')
    print(wrapped.result)

    unwrapped = client.unwrap_key(vault_base_url=_get_vault_url(),
                                 key_name=key_name,
                                 key_version='',
                                 algorithm='RSA-OAEP',
                                 value=wrapped.result)
    print('\nunwrap result:')
    print(unwrapped.result)
    
However, some changes are needed to ensure your code can authenticate against a vault with enhanced message protected enabled.  This is becuase message encryption require proof of possession tokens for authentication in some instances rather than bearer tokens, which are normally sufficient.  Code to implement the proof of possetion token authentication can be found in this sample in the method _authenticate in mcrypt_sample.py.

    def _authenticate(server, resource, scope, scheme):

        # use the authority, resource, scope and scheme as a key to the token cache
        token_cache_key = '_'.join([server, resource, scope, scheme]).lower()

        # check if we have a cached token response from AAD
        token = _token_cache.get(token_cache_key, None)

        # if no cached response exists or the cached token is expired for the auth request
        # request a token from the specified authority (server)
        if (not token) or (_token_expired(token)):
            # if the scheme is pop we need to generate a proof of possession key
            pop_key = generate_pop_key() if scheme.lower() == 'pop' else None

            # create the body for the AAD auth request
            body = {
                'resource': resource,
                'response_type': 'token',
                'grant_type': 'client_credentials',
                'client_id': _get_client_id(),
                'client_secret': _get_client_secret()
            }
            if pop_key:
                body['pop_jwk'] = json.dumps(pop_key.to_jwk().serialize())

            # url encode the body and post the requst
            data = urlutil.urlencode(body)
            response = requests.request('POST', server + '/oauth2/token', data=data)

            # raise error for non success status codes
            response.raise_for_status()
            token = response.json()

            # if a pop key was used to create the token add it to the cached token so it can be
            # used to sign messages in this session
            if pop_key:
                token['pop_key'] = pop_key

            # cache the response
            _token_cache[token_cache_key] = token

        # return a new AccessToken with the scheme, AAD access_token, and POP key
        return AccessToken(scheme, token['access_token'], token.get('pop_key', None))
        
It should be noted that _autheticate will handle authentication for both methods requiring message encryption, and those which do not, so this code can also be used on vaults which do not support message encryption.

