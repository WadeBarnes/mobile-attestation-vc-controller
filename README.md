# TL;DR

This is a Proof of Concept (PoC) of a ACA-py controller for mobile application attestation. It has the following features:

- [x] Apple App Attestation - WIP
- [x] Android Play Integrity API
- [ ] Apple Fraud Detection API
- [ ] AppStore Receipt Checking (iOS <14.0)

# Development

While this controller can be run as a controller for any ACA-py instance, it was developed using "Traction" as the front end. Any documentation or references should be considered in that context.

## Prerequisites

- VSCode
- [Docker](https://docs.docker.com/get-docker/)
- Traction >= 0.3.2
- Suitable tool for exposing localhost to the internet:
  1. [Cloudflared](https://github.com/cloudflare/cloudflared)
  2. [ngrok](https://ngrok.com/download)
  3. [localtunnel](https://www.npmjs.com/package/localtunnel)

## How it Works

When run, this program will act as a "controller" to an ACA-py agent. It uses Flux to handle DidComm basic messages, and, when prompted, will use Traction to issue a basic demo Attestation Credential.

## Running

### Local Development

<!-- 
 redis-cli --cluster create redis-1:6379 redis-2:6379 redis-3:6379 --cluster-replicas 0 
 
 
 
 /data # redis-cli --cluster create redis-1:6379 redis-2:6379 redis-3:6379 --cluster-replicas 0
>>> Performing hash slots allocation on 3 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
M: db572c8cca958fe96b27f7676db60d633ebb723b redis-1:6379
   slots:[0-5460] (5461 slots) master
M: a2bfe0d0508d54090296045d1a10f67bfec81f55 redis-2:6379
   slots:[5461-10922] (5462 slots) master
M: 3b4d9783e79bfd1e75661ae57c701da5a5042ec0 redis-3:6379
   slots:[10923-16383] (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join

>>> Performing Cluster Check (using node redis-1:6379)
M: db572c8cca958fe96b27f7676db60d633ebb723b redis-1:6379
   slots:[0-5460] (5461 slots) master
M: 3b4d9783e79bfd1e75661ae57c701da5a5042ec0 172.21.0.4:6379
   slots:[10923-16383] (5461 slots) master
M: a2bfe0d0508d54090296045d1a10f67bfec81f55 172.21.0.3:6379
   slots:[5461-10922] (5462 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.


/data # redis-cli cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:3
cluster_size:3
cluster_current_epoch:3
cluster_my_epoch:1
cluster_stats_messages_ping_sent:12
cluster_stats_messages_pong_sent:16
cluster_stats_messages_sent:28
cluster_stats_messages_ping_received:14
cluster_stats_messages_pong_received:12
cluster_stats_messages_meet_received:2
cluster_stats_messages_received:28
total_cluster_links_buffer_limit_exceeded:0
 
 
 
 
 
 
 -->

First, create a `.env` file in the root of your folder by copying env.sample to `.env`, populate the values with your own. For Android Attestation you will need a Google OAuth JSON key in `/src` configured for your app.

```bash
APPLE_ATTESTATION_ROOT_CA_URL="https://www.apple.com/certificateauthority/Apple_App_Attestation_Root_CA.pem"
TRACTION_BASE_URL="https://traction-tenant-proxy-dev.apps.silver.devops.gov.bc.ca"
TRACTION_TENANT_ID="b1d5b628-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
TRACTION_TENANT_API_KEY="286e7818-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
GOOGLE_AUTH_JSON_PATH="path_to_google_oauth_json_key_from_src.json"
MESSAGE_TEMPLATES_PATH="fixtures/"
```

You will also need to create a schema and credential definition id in your Traction instance and then add it to `fixtures/offer.json`, following the format.

For simplicity, this repo comes with a `.devContainer` to allow developers to get up-and-running quickly. Use VSCode to restart or start the container. Once the container is running, you can start the controller with the following command:

```bash
python src/controller.py
```

The output should look something like this:

```bash
vscode ➜ /work (main) $ python src/controller.py
 * Serving Flask app 'controller'
 * Debug mode: on
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 107-923-082
```

Flask will expose port `5000` and this is where you need to point whatever tool you have setup to expose localhost, for example, in the case of `ngrok`:

```bash
npx ngrok http 5000
```

Finally, whatever public endpoint is provided from, in this case `ngrok` needs to be provided to Traction as the controller endpoint. This can be done by going to Settings -> Tenant Profile and entering the URL in the `WebHook URL` field.

### OpenShift Cluster

The general command to deploy this to an OpenShift cluster is:

```bash
helm template <RELEASE> ./devops/charts/controller
--set-string tenant_id=<TENANT_ID> \
--set-string tenant_api_key=<TENANT_API_KEY> \
--set-string redis_url=<REDIS_URL> \
--set-file google_oauth_key.json=<PATH_TO_GOOGLE_OAUTH_KEY>| \
oc apply -n <NAMESPACE> -f -
```

And example command to deploy to the `e79518-dev` namespace is:

```bash
helm template bcwallet ./devops/charts/controller
--set-string tenant_id=123-456-789 \
--set-string tenant_api_key=abc-def-ghi \
--set-string redis_url=redis://redis:6379/0 \
--set-file google_oauth_key.json=./google_oauth_key.json| \
oc apply -n e79518-dev -f -
```

The release name can be anything you want, but it must be unique to the namespace. When deploying to a shared namespace like `e79518-dev`, it is recommended use the a meaningful release name that will help reason about what the controller is doing.

## Notes Below Here

https://developer.android.com/google/play/integrity/verdicts#device-integrity-field. You can then distinguish between MEETS_BASIC_INTEGRITY and MEETS_STRONG_INTEGRITY

## Useful Packages

Here are some interesting packages that may be useful:

https://github.com/invertase/react-native-firebase/tree/main#readme
https://www.npmjs.com/package/@react-native-firebase/app-check
`npm i -S @react-native-firebase/app-check`

https://github.com/kedros-as/react-native-google-play-integrity
https://www.npmjs.com/package/react-native-google-play-integrity
`npm i -S react-native-google-play-integrity`

https://github.com/bpofficial/expo-attestation#readme
https://www.npmjs.com/package/expo-attestation
`npm i -S expo-attestation`

## Handy Test Commands

```bash
jq --arg content "$(cat fixtures/request_issuance.json | base64)" --arg name "jason" '.content |= $content | .name |= $name' fixtures/basic_message.json
```

```bash
jq --arg content "$(cat fixtures/request_issuance.json | base64)" '.content |= $content' fixtures/basic_message.json|curl -v -X POST -H "Content-Type: application/json" -d @- http://localhost:5000/topic/basicmessages/
```

```bash
jq -s '.[0] * .[1]' source.json target.json
```

```bash
jq --arg content "$(jq -s '.[0] * .[1]' fixtures/chalange_response.json attestation.json | base64)" '.content |= $content' fixtures/basic_message.json| curl -v -X POST -H "Content-Type: application/json" -d @- http://localhost:5000/topic/basicmessages/
```

## Apple Verifications Steps

- [x] Use the decoded object, along with the key identifier that your app sends, to perform the following steps:

- [x] Verify that the x5c array contains the intermediate and leaf certificates for App Attest, starting from the credential certificate in the first data buffer in the array (credcert). Verify the validity of the certificates using Apple’s App Attest root certificate.

- [x] Create clientDataHash as the SHA256 hash of the one-time challenge your server sends to your app before performing the attestation, and append that hash to the end of the authenticator data (authData from the decoded object).

- [x] Generate a new SHA256 hash of the composite item to create nonce.

- [x] Obtain the value of the credCert extension with OID 1.2.840.113635.100.8.2, which is a DER-encoded ASN.1 sequence. Decode the sequence and extract the single octet string that it contains. Verify that the string equals nonce.

- [ ] Create the SHA256 hash of the public key in credCert, and verify that it matches the key identifier from your app.

- [ ] Compute the SHA256 hash of your app’s App ID, and verify that it’s the same as the authenticator data’s RP ID hash.

- [ ] Verify that the authenticator data’s counter field equals 0.

- [ ] Verify that the authenticator data’s aaguid field is either appattestdevelop if operating in the development environment, or appattest followed by seven 0x00 bytes if operating in the production environment.

- [ ] Verify that the authenticator data’s credentialId field is the same as the key identifier.

After successfully completing these steps, you can trust the attestation object.

## Android Verification Steps

- [x] Get Integrity Verdict from Google's server via their python client
- [x] Verify the package info matches our app
- [x] Verify the device integrity fields
- [ ] Verify the app integrity fields (left commented out while in development)
- [ ] Verify the nonce in the verdict payload matches the nonce the controller sent to the device
