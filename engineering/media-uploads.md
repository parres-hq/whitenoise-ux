# Media Uploading - March 2025

## What's the problem we're solving

Effective Communication goes beyond just text. We want users of White Noise to be able to richly communicate with images, videos, audio messages & audio files, and documents as well as text.

However, given Noster's open and decentralized relay network, we need to think carefully about how and where these files are stored so that we don't unintentionally expose the content of the files or any sensitive metadata.

This document explores several different aspects of the problem space and details the solutions/trade-offs that we have decided to go with in order to add media uploads to White Noise.

## Basic requirements

- Users should be able to upload one or more media files in chats.
- Uploaded files should show in the UI before being sent as a message to the group.
- Users should be able to send messages containing media files with or without text content.
- Users should be able to remove uploaded media files before sending them to the group.
- Media files should be encrypted using a group secret (exporter secret), ideally using a standard nostr method of encryption.
- Media files should be uploaded to a blossom server chosen by the user.

## Encryption scheme

### Facts

- Currently NIP-44 v2 encryption has an upper limit of 64kb of data.
- There are discussions to create a new version that removes this limit. [Issue](https://github.com/nostr-protocol/nips/issues/1712) / [PR](https://github.com/nostr-protocol/nips/pull/1838)
- We currently use NIP-44 v2 in other places (sending welcome messages, encrypting the content of `kind:445` messages).

## Blossom uploads

### Facts

- Most blossom servers require authentication
