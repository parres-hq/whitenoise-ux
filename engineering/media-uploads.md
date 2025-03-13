# Media Uploading - March 2025

## What's the problem we're solving

Effective Communication goes beyond just text. We want users of White Noise to be able to richly communicate with images, videos, audio messages & audio files, and documents as well as text.

However, given Noster's open and decentralized relay network (which doesn't support storing binary data), we need to think carefully about how and where these files are stored so that we don't unintentionally expose the content of the files or any sensitive metadata.

This document explores several different aspects of the problem space and details the solutions/trade-offs that we have decided to go with in order to add media uploads to White Noise.

## Basic requirements

- Users should be able to upload one or more media files in chats.
- Uploaded files should show in the UI before being sent as a message to the group.
- Users should be able to send messages containing media files with or without text content.
- Users should be able to remove uploaded media files before sending them to the group.
- Media files should be encrypted using a group secret (exporter secret), ideally using a standard nostr method of encryption.
- Media files should be uploaded to a [blossom](https://github.com/hzrd149/blossom) server chosen by the user.
- Media files should show in the chat transcript alongside any text content.
- The app should download and locally cache images from chats. These images should respect the disappearing message rules we'll inplement in the future

## Encryption scheme

### Facts

- Currently NIP-44 v2 encryption has an upper limit of 64kb of data.
- There are discussions to create a new version that removes this limit. [Issue](https://github.com/nostr-protocol/nips/issues/1712) / [PR](https://github.com/nostr-protocol/nips/pull/1838)
- We currently use NIP-44 v2 in other places (sending welcome messages, encrypting the content of `kind:445` messages). So far we haven't hit the limits on these but it's a known issue with larger groups, for example with the serialized welcome message content would go beyond this limit in groups of ~150 users in my tests.
- There is also an [encrypted file message kind](https://github.com/nostr-protocol/nips/blob/master/17.md#file-message-kind) as part of NIP-17.
- There is also media attachments in [NIP-92](https://github.com/nostr-protocol/nips/blob/master/92.md). We're currently using these to show images as part of the message content.

### Discussed ideas

- Switch to AES-GCM encryption for media uploads (this would likely need to be an extension of or addition to [NIP-EE](https://github.com/nostr-protocol/nips/pull/1427)). What this would look like is a combination Of nip-17 encrypted file messages (minus the inclusion of the decryption key tag) and of nip-92's `imeta` tags.

## Blossom uploads

### Facts

- Most blossom servers require authentication

### Discussed ideas

- Use a ephemeral nostr identity to create an authentication event required for uploading. I believe it's true that the GET /<sha-256> endpoint is usually not guarded by auth on Blossom servers. If, this is the default behaviour then we can use this method to upload under throw away identities and only the user's who know where those files are will be able to access them, and then decrypt them.
