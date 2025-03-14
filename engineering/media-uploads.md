# Media Uploading - March 2025

## The Problem

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

## Encryption

### Facts

- Currently NIP-44 v2 encryption has an upper limit of 64kb of data.
- There are discussions to create a new version that removes this limit. [Issue](https://github.com/nostr-protocol/nips/issues/1712) / [PR](https://github.com/nostr-protocol/nips/pull/1838)
- We currently use NIP-44 v2 in other places (sending welcome messages, encrypting the content of `kind:445` messages). So far we haven't hit the limits on these but it's a known issue with larger groups, for example with the serialized welcome message content would go beyond this limit in groups of ~150 users in my tests.
- There is also an [encrypted file message kind](https://github.com/nostr-protocol/nips/blob/master/17.md#file-message-kind) as part of NIP-17.
- There is also media attachments in [NIP-92](https://github.com/nostr-protocol/nips/blob/master/92.md). We're currently using these to show images as part of the message content.

### Discussed ideas

- Switch to AES-GCM encryption for media uploads (this would likely need to be an extension of or addition to [NIP-EE](https://github.com/nostr-protocol/nips/pull/1427)). What this would look like is a combination Of nip-17 encrypted file messages (minus the inclusion of the decryption key tag) and of nip-92's `imeta` tags.

### Outcomes

- We're going to use `AES-GCM-256` to encrypt the files. This is simpler than using NIP-44 and also has precedent in the NIP-17 encrypted files kind.
- We generate a random nonce to use when encrypting each file and include that in the tags (detailed below).
- We include the `encryption-algorithm` in the tags to make sure it's clear and to leave us room to change the algorithm in the future.

## Blossom uploads

### Facts

- Most blossom servers require authentication, but we don't want to authenticate with a user's Nostr identity keys.
- Blossom servers _may_ require auth for `GET /<sha-256>`, so ideally, we need to check that the server doesn't require auth for `GET` requests. There is an [open PR](https://github.com/hzrd149/blossom/pull/58) on the blossom repo about information docs for blossom servers that would eventually be helpful.
- We need to ensure that we don't include the real `Content-Type` or any other identifying metadata about what the file is or where it comes from.
- Most blossom servers will add a `.bin` file type suffix to files that are uploaded with the `Content-Type: application/octet-stream` header.

### Discussed ideas

- Use a ephemeral nostr identity to create an authentication event required for uploading.

### Outcomes

- We'll hardcode a blossom server that we know doesn't require auth for `GET` requets.
- We'll upload our encrypted files with the `Content-Type: application/octet-stream` header and handle the eventual `.bin` file type suffix

## The Solution

### Uploading & Sending

1. Encrypt the file with `AES-GCM-256` using a random 12-byte nonce.
1. Create a Blossom `kind: 24242` authentication event and sign it using an ephemeral Nostr key pair.
1. Using the auth event, upload the encrypted binary file to the blossom server using a `Content-Type: application/octet-stream` header and a `Content-Length` header with the length of the encrypted file. We don't add any other data to this request on purpose to limit metadata.
1. The response value gives us the blossom URL of the file.
1. We build our `kind: 9` event, appending the file urls to the message content (each on a new line).
1. We create a standard NIP-92 `imeta` tag with the details of the original file (not the encrypted version). This is where we signal the real values for the dimensions, content-type, blurhash, etc.
1. Additionally, we add three fields to the imeta tag that are non-standard in NIP-92 (two of these values exist as their own tags in NIP-17).
   1. `encryption-algorithm` - the encryption algorightm used, in our case it's always `aes-gcm`
   1. `decryption-nonce` - the 12-byte hex-encoded nonce used when encrypting
   1. `filename` - this is the original filename of the file uploaded. We can use this to display the original filename in messages instead of the blossom url.
1. From here, we encrypt and send the message as normal.

Here's an example of a `kind:9` unsigned event with a single encrypted image:

```json
{
  "id": "bdc3158f61427564e36207fe969ec4ed8c320ba5f2dd3646451d536486ba31f9",
  "pubkey": "7a6ac2abc092d404a6a8ca93e97a58dc5082dfb8a744c984cd40c944fb6d6574",
  "created_at": 1741874237,
  "kind": 9,
  "content": "Testing\nhttp://blossom.server.com/309af8602a3a56aded6cdd6756d245a5d0731483f362e976f526c6de2daa7391.bin",
  "tags": [
    [
      "imeta",
      "url http://blossom.server.com/309af8602a3a56aded6cdd6756d245a5d0731483f362e976f526c6de2daa7391.bin",
      "m image/png",
      "dim 1200x1200",
      "blurhash LSRysgof~qRjofofWBRj_3j[9FfQ",
      "x b5bb9d8014a0f9b1d61e21e796d78dccdf1352f23cd32812f4850b878ae4944c", // The sha-256 hash of the original file, not the encyrpted blob
      "descryption-nonce 34661f3eb8541bcb5d4d4f98",
      "encryption-algorithm aes-gcm",
      "filename costa-rica-sunset.png"
    ]
  ]
}
```

### Downloading & Displaying

1. When parsing a `kind:445` message, we decrypt the message content as usual using NIP-44. Then parse and process the resulting serialized MLSMessage object.
1. If we find URLs that have corresponding `imeta` tags, we use the data in the `imeta` tag to replace the blossom URL in the displayed message with the actual image(s). With files and videos we'll need to decide how to show some meaningful preview. If there is more than 1 image, we should show images in a grid (max 4 images).
