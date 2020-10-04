# Protocol documentation

## Summary
 - [Basics](#basics)
 - [New device side](#new-device-side)
 - [Trusted device side](#trusted-device-side)
 - [Notes](#notes)

## Basics
Remote authentication works with 2 sides. We'll refer to the device you are authenticating the "New device" and the
authenticated device "Trusted device".

The New device will interact with a WebSocket, while the Trusted device will interact with a REST API. This is a
nice balance between the amount of data the new device will receive and to get realtime events from the Trusted device,
and not using too much power and resources from the Trusted device which can be a phone with limited battery and
processing power.

## New device side
To prevent overload, each IP can only open 3 different connections (to also account for multiple people under the same
roof). If you try to open more connections, the oldest session will be force closed. In addition, each IP may only
create 10 sessions per minute.

### Packet format
The payload structure is flat, with relevant data just as additional keys on the packet. Only `op` key is consistent.
```json
{
  "op": 0,
  "relevant": "data"
}
```

### Opcodes
| Opcode | Name          | Sender | Description                                                |
|--------|---------------|--------|------------------------------------------------------------|
| 0      | HELLO         | Server | Sent on connecton open.                                    |
| 1      | KEY           | Client | Shares public key to server.                               |
| 2      | NONCE         | Both   | Nonce used to test the encryption.                         |
| 3      | TOKEN         | Server | Sent once encryption is negotiated.                        |
| 4      | SESSION_INIT  | Server | Sent once a trusted device initiated an authentication.    |
| 5      | SESSION_TOKEN | Server | Sent once the trusted device confirmed the authentication. |
| 6      | HEARTBEAT     | Client | Ping. Must be sent every <n> ms specified in `HELLO`.      |
| 7      | HEARTBEAT_ACK | Server | Pong.                                                      |

#### HELLO
Sent as soon as the connection opens. Contains the following data:
 - `heartbeat_interval`: Interval (in ms) between each heartbeat expected by the server. See [HEARTBEAT](#heartbeat).
 - `session_lifetime`: Time (in ms) before the session will be invalidated and the connection closed.

#### KEY
At this stage, the client should generate a RSA keypair (RSA-OAEP with SHA-256) and share the public key to the server
to enable encryption. The payload should contain the following data:
 - `public_key`: Base64-encoded public key, in SPKI format

##### Why a RSA key exchange? This is unnecessary!
No. This is, in fact, the most important part.

Although the whole thing should use a SSL connection, a clever attacker can run a MITM attack and steal user data and
an authentication token. This prevents it. See [how](#MITM-proof).

#### NONCE
Sent to test the encryption. Contains the following data:
 - `nonce`: Random bytes, base64 encoded.

The server will send this payload with encrypted data, and expects to get the same payload with the decrypted nonce
back. If the nonce returned to the server is incorrect, the connection will be closed, as the server will consider
the encryption initialization failed.

#### TOKEN
The token may consists of multiple parts, separated by a point. The client should only expect 1 part, as any additional
parts are internal (or for custom clients using a superset of this protocol). The first part will always be the SHA-256
hex digest of the public key fingerprint. **ALL CLIENTS <u>MUST</u> CHECK THIS EQUALITY**; see [why](#MITM-proof).

Payload contains the following data:
 - `token`: The session token. You need to pass this data to the trusted device (using a QR Code for example).

#### SESSION_INIT
Sent as soon as a trusted device initiated a remote authentication. Contains the following data:
 - `user`: Basic user details. Passed as an encrypted JSON string. Data will depend on the configuration of the server.

User details should be shown to the user to let them visually confirm the right device is being authenticated.

#### SESSION_TOKEN
Sent once the trusted device confirmed the authentication. The connection will be closed immediately after this payload
has been dispatched. Contains the following data:
 - `token`: Encrypted authentication token.

#### HEARTBEAT
The client must sent heartbeats every <n> ms specified by the server, and must confirm they receive the corresponding
acknowledge from the server. If the client doesn't send heartbeats, the server will consider the connection dead and
close it. If the server doesn't send the acknowledgement after a heartbeat, the server is probably dead and the client
should reconnect.

There is no payload attached, and the server should immediately respond with a `HEARTBEAT_ACK`, without payload either.

## Trusted device side
The trusted device only has 2 API endpoints to call to perform the authentication. The process has been designed to be
extremely light and consume the least amount of resources as possible since it's primarily designed to be run on
mobile devices.

The fingerprint we expect the device to use is the one sent by the server to the new device. It can be shared in
any way (QRCode, black magic, ...). **The device must implement reasonable enough security to protect the user**.
The flow is designed to require two interactions from the user to let it scan the QR code, see **visual feedback** on
the device they're authenticating, and then confirm the authentication with a warning message.

### Authenticating to the API
All API calls must be authenticated. To authenticate, the microservice must be able to define the validity of a token
and get a user object from it, by itself. This means your tokens have to either be stateless ([JWT](https://jwt.io),
[Tokenize token](https://github.com/Bowser65/Tokenize), ...), or have the possibility to be mapped to a user ID/object,
for example using a database.

How the authentication token must be passed depends on how it has been configured. Read CONFIG.md for more details.

### Initialize
Once the device received the session token, it can start initializing the remote authentication. This is a simple
HTTP POST request:

```
POST /initialize
Content-Type: application/json

< { "token": "[session token]" }

# Success response:
HTTP 200 OK
Content-Type: application/json
> { "ticket": "[ra ticket]", "features": [ "array of strings" ] }

# Error response (failed auth):
HTTP 401 Unauthorized

# Error response (invalid token):
HTTP 400 Bad Request
```
Once the ticket has been generated, the new device will receive the `SESSION_INIT` payload with encrypted user data.
The ticket is only valid for 60 seconds.

### Confirm the authentication
To confirm the authentication (and grant a token to the user), you need to do another POST request. You can also
describe some token properties, which will be used during generation. The available features are served in the
initialization payload.

**THIS ACTION MUST NOT BE AUTOMATED**. The user **must** be prompted to confirm the authentication, so they can
confirm the correct device is being authenticated (thanks to visual feedback for example).

```
POST /confirm
Content-Type: application/json

< { "ticket": "[session ticket]", "features": [ "features to enable, opt-in" ] }

# Success response:
HTTP 204 No Content

# Error response (failed auth):
HTTP 401 Unauthorized

# Error response (invalid ticket or invalid features):
HTTP 400 Bad Request
```
The new device will receive the `SESSION_TOKEN` and will be disconnected from the RA server; the flow is now complete.

### Cancel the authentication
If you wish to cancel the authentication, you need to send a simple DELETE request to invalidate the ticket.
```
DELETE /cancel
Content-Type: application/json

< { "ticket": "[session ticket]" }
# Success response:
HTTP 204 No Content

# Error response (failed auth):
HTTP 401 Unauthorized

# Error response (invalid ticket or invalid features):
HTTP 400 Bad Request
```
The new device will be disconnected from the RA server.

## Notes
### MITM-proof
By requiring a RSA key exchange and encrypting data, we are voiding any chances for an attacker to run a successful
MITM attack. But it's not only because we use encryption.

See, in [FINGERPRINT](#FINGERPRINT), we **strongly insist** on the fact that you **must** check the first part of the
fingerprint is your public key' fingerprint. That's the nail in the coffin of the attacker:

Since the data is encrypted, the attacker would have to decrypt it to get your precious user info and token. But since
the attacker doesn't know the private key, they must generate a new pair to pass to our servers and have a way to
decrypt the data. And here the attacker has two solutions
 - Either the attacker passes the real fingerprint from the server which has *their* fingerprint
 - Replace the fingerprint and pass a tampered one to the client

In the 1st scenario, the client will check the fingerprint, reject it and immediately close the connection;<br>
In the 2nd scenario, the client will not reject the fingerprint, but the server will not know this one and reject it

So, thanks to this encryption + fingerprint validation, we prevent any MITM attack. Have a coffee ☕

#### But what's the point of mitigating a MITM attack?
Well, it depends. This microservice has been designed to be as generic as possible, so it can easily be integrated
in any application and be used as-is. Which is why security is not taken lightly as this may be in some cases the only
entry point for an attacker, due to the *remote* nature of this thing.

It also is useful even when using a secured HTTPS connection. It isn't rare for HTTPS connections to be
*intentionally* MITM'd, and unfortunately with in a lot of cases old and vulnerable software. The key exchange on
an application-level accounts for this potential flaw and further secures the flow. There are for sure many other attack
vectors, but this is not an excuse to lower the security level of other services.
