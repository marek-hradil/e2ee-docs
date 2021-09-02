# End-to-end Encryption Protocol Documentation

> Author: Marek Hradil
> Last Update: 22.8.2021

This document describes how the FaceUp encryption protocol works,
what were the main design goals, and the choices we have made to achieve them.
The main focus is to explain parts of the encryption system in a way that is understandable for newcomers or reviewers.

## Outline

- [Platform specification](#platform-specification)
  - [Platforms](#platforms)
  - [Users](#users)
  - [Institutions](#institutions)
  - [Reports](#reports)
- [Main goals of encryption system](#main-goals-of-encryption-system)
- [Solution overview](#solution-overview)
  - [Environments and tools used](#environments-and-tools-used)
  - [Report encryption](#report-encryption)
  - [Report key exchange](#report-key-exchange)
  - [Creating report](#creating-report)
  - [User login](#user-login)
  - [Whistleblower login](#whistleblower-login)
  - [User registration and recovery](#user-registration-and-recovery)
  - [Attachment encryption](#attachment-encryption)
- [Unfinished parts](#unfinished-parts)
  - [Integration with SSO](#integration-with-sso)
  - [Key rotation](#key-rotation)
  - [Better chat message and internal comment encryption](#better-chat-message-and-internal-comment-encryption)
- [Future upgrades](#future-upgrades)
  - [WebCrypto keys](#webcrypto-keys)

### How to read

If you're familiar with the platform and you just need the encryption info, read the solution overview.
Otherwise, try to take a look into the platform specification.
In unfinished parts, problems that weren't solved ideally are listed.
The future upgrades chapter is for ideas that aren't implemented yet but seem like a good idea for further consideration.

## Platform specification

The FaceUp platform can be shortly described as whistleblowing software - it gives people a secure communication channel, through which they can report anything, that is not easy to say in person.

> Also, if you want more of the bussiness/overall info to get more insight - [nntb.cz(czech)]("https://www.nntb.cz") or [faceup.com(other)]("https://www.faceup.com").

### Platforms

- **Reporting app**
  - Made for whistleblowers. They can create a new report or open an already existing one.
- **Administration app**
  - Made for institution administrators. They can view reports, chat with whistleblowers or with other administrators and manage reports (assign people, change statuses, view and export graphs, ...). In the settings section, they can manage administrators, manage the organizational structure and customize institution reporting app - changing colors, texts, logos. Administrators cannot delete reports and can edit only reports created by themselves.
- **FaceUp administration**
  - Made for FaceUp employees. They can register, edit or delete institutions and manage their payments. Also, reports sent to unregistered schools can be viewed here and resent to school email.

All of the platforms are web applications, served by a static file server and communicating with the backend server via HTTPS.
More of the technical details in [tools used](#environments-and-tools-used)

### Users

There are three main types of users:

- **Whistleblowers** (also Victims)
  - They want to anonymously create a report and then be able to come back to it and chat with institution administrators. Also, they don't have to be authorized before they submit the report.
- **Administrators** (also Teachers or Managers)
  - They want to be able to list all reports send to their institution, read them and chat between themselves and with the whistleblower. They have to be registered by other administrators or by FU admin.
- **FaceUp Administrators** (also FuAdmins)
  - They want only the ability to list all institutions and preview their basic info (name, address, ...).

### Institutions

Currently, the platform has two main institution types:

- **Schools**
- **Companies**

Both of them share around 80% of the functionality however, some details may differ - students have to report their class, employees choose the organizational unit, etc. None of the differences should be important for this protocol, except one - students in the Czech republic can send a report to an unregistered school. An unregistered school has no registered users - in this case, we cannot encrypt data end-to-end (one end is missing), and so the protocol should be able to opt-out of the E2EE easily.

_(This separation may not be permanent and could be merged in the future, given the small differences)_

### Reports

Piece of data, sent by a whistleblower to point out a problem, that is sent to their institution.
Here differences between institution versions come to play.
For both institutions:

- more info
- attachments
- organizational unit id
- chat messages, can have their attachments
- internal comments messages, can have their attachments

Schools have also:

- victim name
- class

And companies differ:

- sender name (optional)
- category of problem

Different users then can interact with the report in different ways:

- **Whistleblowers** - can chat with administrators and come back to the report with a generated pin
- **Administrators** - can chat with whistleblower or append internal comments to the report
- **FaceUp Administrators** - usually cannot access reports, except scenario, where no recipient is registered. If so, they can access the report and send it to the institution by email.

## Main goals of encryption system

Encrypt reports in the way that the reports are kept safe and can be accessed:

- only by:
  - Whistleblowers
  - Institution administrators
- but not by:
  - Faceup administrators
  - System administrators (meaning whoever develops or manages the platform)
  - Others

## Solution overview

Designing an encryption protocol, that is as close as possible to end-to-end encryption (E2EE) while keeping platform simplicity (not having a mobile app and sending reports through it).
However, this restriction (not developing a mobile application) means that an ideal result is not possible.
On the client-side (the browser), the code is not downloaded to the device and thus we cannot guarantee that the user gets the same code every time.
A malicious system administrator could then change the static server functionality or stored code and send code, that leaks information, doesn't encrypt data properly, etc. True E2EE is currently **not possible**.

But implementing this type of encryption still brings parts of the desired results:

- users still can sense malicious activity on their data
- transfers over the network are secure (if the TLS would be broken into)
- attack vector on these data (possible leak outside of the platform) is cut down only to client-side
  - when considering protecting static file server and client-side code easier than the whole infrastructure
- reports are kept private
  - not leaked to logs, metrics, ...

For these reasons, we decided to implement the E2EE protocol.

### Environments and tools used

Client-side:

- **React** for UI render
- **libsodium.js** for encryption and randomness
- **Apollo client** for GQL communication with BE

Server-side:

- **Node.js** as runtime
- **Apollo server** for GQL API
- **libsodium.js** for randomness

Encryption:

- secret key encryption - [libsodium secretbox](https://libsodium.gitbook.io/doc/secret-key_cryptography/secretbox#algorithm-details)
- public key encryption - [libsodium sealed box](https://libsodium.gitbook.io/doc/public-key_cryptography/sealed_boxes#algorithm-details)
- key derivation and hashing
  - [libsodium pwhash(argon 2)](https://libsodium.gitbook.io/doc/password_hashing/default_phf#algorithm-details) for short input
  - [libsodium crypto_kdf_derive_from_key](https://libsodium.gitbook.io/doc/key_derivation)
- random data generation - [randombytes\_\*](https://libsodium.gitbook.io/doc/generating_random_data)
- file encryption [libsodium secretstream](https://libsodium.gitbook.io/doc/secret-key_cryptography/secretstream)

### Report encryption

First of all, the report key is generated of `crypto_secretbox_KEYBYTES` length (256 bit) along with nonce of length `crypto_secretbox_NONCEBYTES` (64 bit).
Victim/Sender name and more info fields are converted to serialized into JSON format and encrypted symmetrically.
Chat messages and comments are also encrypted with the report key, but one by one, not in serialized JSON format.

For attachments, `secretstream` is used, acting as a kind of ratcheting algorithm, so the parts of the file cannot be switched.

Reading the report after encrypting it is shown on this diagram:

![Read report process](/images/read-report.png)

### Report key exchange

Transferring keys between users is done simply through user key pairs. Every user has its private key and public key (these are generated via `crypto_box_keypair`).
After encrypting the report, users public keys are fetched from the server. For every recipient (a user that should have access to the report), the report key is encrypted with the public key. The encrypted report keys are then sent to the backend and distributed between users.

### Creating report

For further clarity, we will include this diagram of the full create report process.

One other thing could be new here - generated pin for accessing the report. It allows the victim to come back to the report, and from the technical perspective, it is nothing but identity and secret added and generated together.

The two parts, in more detail:

- identity part(6 numbers) - generated on the BE, unique and scoped to one institution
- secret part(10 numbers) - generated on the FE, acts like a password for normal users

The question is if the 10^16 size is sufficient however, the needed key derivation takes about 1-2s for one guess, so the time should be sufficient to shut down the user account. Also extracting the encrypted private key from BE, to break it locally should not be possible.

![Create report process](/images/create-report.png)

### User login

Common process based on email and password is used, however with two major differences:

- we need to derive key from password (later reffered to as password key)
- we want to send a password that is pre-hashed, so we don't allow the backend to derive password key
  - that also requires user salt on client-side

With that in mind, the following process is implemented:

1. User sends a request to the server, to the pre-login endpoint. Supplies identity (email/identity pin part) and version parameters.
2. Server responds with his salt, or if the identity was invalid, with deterministic salt generated from the identity. This is done in constant time.
3. User continues with deriving the password key from the password (password/secret pin part) and the salt. For that `pwhash` is used. This key is then stored in memory.
4. User derives password pre-hash from password key and sends it along with identity to login endpoint.
5. Login endpoint completes the hashing with the last hash, where it applies app salt and checks it with the database records (constant time again). Encrypted user private key is sent back along with its nonce, JWT, and public key.
6. User private key is decrypted with password key and nonce.
7. Public key and private key are saved to user storage and password key is deleted from memory.
8. User is logged in.

![User login process](/images/user-login.png)

### Whistleblower login

Logging in is for the whistleblower is very similar to the normal user process however, the obvious difference is that the whistleblower doesn't have an email address and password, as it is not an anonymous way to log in.

These credentials are therefore substituted for one PIN (generated in create report process), made of two parts - identity and secret ([further specification of the parts](#creating-report)).

Otherwise, the process is the same as the user login process, but email is substituted for the identity part and password for the secret.

**Pre-login endpoint** should be specified in more detail. It has two main purposes:

- getting user salt - necessary for the pre-hash of the user password
- getting user encryption version - for encryption migration
  Also, it is important to note that the only parameter that can be supplied is the email of the user or the identity part of the PIN of the whistleblower.

Making this endpoint secure is however quite hard in reality because it has to be:

- constant time
- deterministic on the input

The constant time is the simpler of the two requisites. On each call of the endpoint, salt and version are generated, then DB is asked for the useand if it is not found, fake salt and version are returned.

If the salt and version would be random, running the endpoint twice on the same input would leak pieces of information about registered and not registered emails.

The implemented solution thus uses random data generation with seed (binary sequence composed of a few static bytes and bytes from the email). That should result in salt that is the same between the calls but not easily identifiable given the static pieces of data.

_(The process of using seed in the data generation is genuinely the same as using HMAC. Should be upgraded in the future, but on the first search of this in libsodium documentation, I did not find it.)_

### User registration and recovery

Encrypting data with the user password is great, but users tend to lose passwords frequently, and not letting them recover somehow their account would be very strict. A recovery mechanism is implemented.

The information user wants to recover is either the password key (so he can decrypt his private key) or the private key itself.
But getting the private key itself results in a better user experience because it doesn't rotate along with the password - in the password key recovery, the recovery key would change in every password case. This is not the case with the private key, where it rotates independently.

In registration, the recovery key is created:

- **Recovery key** (128 bit) - used to encrypt user private key, then serialized into a string and given to the user

The recovery key itself is also encrypted with the current password key, to be showable in administration.

![User registration process](/images/user-registration.png)

The recovery process is then following:

1. User supplies email token, new password, and recovery key
2. System fetches private key recovery and decrypts it with the recovery key
3. New password key, password pre-hash, salt, and a nonce is created
4. Private key is re-encrypted with the new password key
5. Recovery key is re-encrypted with the new password key
6. New params are sent to the user, and public key, private key, and JWT are saved into storage

![User recovery process](/images/user-recovery.png)

### Attachment encryption

Finally, more detail into how attachments are encrypted could be useful.
When encrypting attachment, the report key is used, but not directly, because of a different key length.
This transition is done by key derivation function (`crypto_kdf_derive_from_key`), to which report key, context, and subkey id are given.
Context is simply for the distinction of different keys and is static in the platform code.
Subkey id is randomly selected to fit into one byte (aka values 0-255) and is different for each attachment.
The whole process is then as follows:

1. Generate subkey id and libsodium `secretstream` header
2. Convert these two to buffers and concatenate them
3. Create chunks from the source file (size of chunk - 40960) and run them through `secretstream`.
4. Concatinate encrypted chunks and prepend header with subkey id.
5. Send to file storage and save URL to backend.

## Unfinished parts

### Integration with SSO

One big problem that came along during the implementation was the Single Sign On (SSO).
SSO is a way for the user to have one account shared between different applications. The account info is usually stored in some user storage (AWS Cognito, Auth0, Microsoft AD, ...) and accessed through the provider. This is considered a good and secure practice however, for us it came with a problem.

Given its design, the providers don't have a way how to access any key that could be derived from the user password. Also storing keys in user additional data doesn't seem very secure, because it is essentially the same as storing a plain unhashed password.

> We have tried to contact AWS to help us with our problem, but after some email exchanges, they told us that they're currently unable to deliver a solution to this problem. Stackoverflow question is still unanswered and the Auth0 forum guided us to store the key in additional data.

The current solution doesn't leverage E2EE advantages as it uses a server key. User private keys are then encrypted with the server key.

### Key rotation

Not having a key rotation strategy is surely not a best practice, but it is planned to be implemented soon for user key pair (public, private key).

### Better chat message and internal comment encryption

Chat messages and internal comments are currently encrypted one by one. This leaves space for attacks such as changing the order of messages that could be dangerous.
A ratcheting system should be implemented instead, as it is in files encryption, but the problem is internal comments can be deleted. This isn't supported by libsodium `secretstream` so a simpler solution (one-by-one message encryption) was designed.

## Future upgrades

### WebCrypto keys

Very interesting should be the WebCrypto CryptoKeys API - it offers the ability to import a key and then save it as an unextractable object. That should result in improved security, where if a malicious script is not in place at the time of the user logging in and importing the key to the object, it should not be able to extract the key afterward.

The problem is the libsodium client-side part has to be then switched for WebCrypto encryption functions that have to be configured by the programmer and could then leave bigger rooms for errors. Also in the public key encryption, it allows only RSA-OAEP encryption but no alternative (libsodium uses ECC).
