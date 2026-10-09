# MSCxxxx: Attaching media to events

the client uploads restricted media, then names those local media ids when it sends an event or sets a profile. It is not the [MSC4396] multipart send. The server does not invent the MXC during `/send`. The client already has it.

Uploads and sends stay two requests. Reuse is `POST /copy`, which mints a new local media id for the same bytes. One media id is bound to one event or one profile. Federation learns the binding from a restrictions object on download. A caller who must not see the media gets 404, the same status as a missing id.

Spec v1.19 already authenticates download. It does not define this upload, this query parameter, or /copy. Those are the new pieces.

## Proposal

### Overview

After an item of media is uploaded, it is not viewable, except by the user who uploaded it, until it is attached. Attachment is the repeated `attach_media` query parameter on the send, state, and profile APIs. Each value is a local media id. The client writes the matching mxc:// URI into the event or the profile. From then on, a user can download the media only when they can see that event or that profile.

To-device events and account data use the same parameter. They are not room events, so their visibility rule is narrower and is fixed below.

### 1. Upload

`POST /_matrix/client/v1/media/upload` behaves like [POST /_matrix/media/v3/upload](https://spec.matrix.org/v1.19/client-server-api/#post_matrixmediav3upload), except the new object is restricted. The same applies to `POST /_matrix/media/v1/create` from MSC2246.

The existing upload endpoint remains for legacy unrestricted media. Restricted media is never served from the unauthenticated `/_matrix/media/v3` download or thumbnail endpoints. There is no grace period.

Only the uploader can download a restricted object before it is attached. If it is not attached within 24 hours, the server deletes it. A `/copy` result is a new restricted object with its own 24 hour window. The creator of a copy is the user who called `/copy`.

An application service is not a paste bin. It uploads restricted media and attaches it to the event it sends, then forwards the bytes to the remote network itself. A world-readable URL is hosted outside Matrix. Unattached Matrix media is deleted, so a bare MXC is not a public link.

### 2. Attaching

The client may attach only media it created. For a copied id, that is the user who called `/copy`.

One event can carry several ids. Servers set the maximum, and the maximum is at least 2, so an image and a thumbnail both fit. More than the maximum is `400 M_LIMIT_EXCEEDED`. Each id is attached to only one event or one profile. The same bytes go to another event only after `/copy`.

These methods gain a repeated `attach_media` query parameter:

- [PUT /_matrix/client/v3/rooms/{roomId}/send/{eventType}/{txnId}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3roomsroomidsendeventtypetxnid)
- [PUT /_matrix/client/v3/rooms/{roomId}/state/{eventType}/{stateKey}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3roomsroomidstateeventtypestatekey)
- [PUT /_matrix/client/v3/sendToDevice/{eventType}/{txnId}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3sendtodeviceeventtypetxnid)
- [PUT /_matrix/client/v3/user/{userId}/account_data/{type}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3useruseridaccount_datatype)
- [POST /_matrix/client/v3/createRoom](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3createroom)
- [POST /_matrix/client/v3/rooms/{roomId}/upgrade](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3roomsroomidupgrade)
- [POST /_matrix/client/v3/rooms/{roomId}/invite](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3roomsroomidinvite)
- [PUT /_matrix/client/v3/profile/{userId}/{keyName}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3profileuseridkeyname)

Each value is the local media id only. It is the path component from [PUT /_matrix/media/v3/upload/{serverName}/{mediaId}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixmediav3uploadservernamemediaid), without mxc:// and without a server name. The media is always on the caller's server. A value that contains /, mxc://, or is empty is malformed, and the whole request is `400 M_INVALID_PARAM`.

Example, an image and its thumbnail:

`PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}?attach_media=abc123&attach_media=def456`

If any well-formed id cannot be attached, the server responds with 404 and attaches none of them. 404 covers an unknown id, an id this user did not create, an id that is already attached, and an id this user cannot access. The same status is used for all of those so the caller cannot tell them apart.

Sending the event binds each id to that event. A profile write binds the id to that user. Create room, upgrade, and invite bind each id to the one event the server creates that references it. When the server has to put the same bytes on a second event, it copies first, then binds the new id. The client's original id is still bound to only one event.

A repeated transaction with the same ids returns 200 and the original event id. That check happens before the 404. A new transaction id that presents an already-attached media id is 404.

`attach_media` is the access list. The client must list every restricted media id the event or profile references, and must not list an id the body does not reference.

Where the server can see the content, it enforces that. Profile fields, account data, and unencrypted room events of a type the server understands (`m.room.message`, `m.sticker`, `m.room.avatar`, `m.room.member`) are compared with `attach_media`. Every local MXC media id in that content must appear in `attach_media`, and every `attach_media` value must appear in the content. A mismatch is `400 M_INVALID_PARAM`, because both sides were supplied by this caller and the error does not reveal another user's media.

`m.room.encrypted` and event types the server does not understand are not parsed. `attach_media` is the only access list. Widgets and other custom events stay on that rule. Their clients are responsible for the match. Encrypted content stays opaque, which is the point of encryption.

### 3. Download and thumbnail

The authenticated `/download` and `/thumbnail` endpoints from MSC3916, which v1.19 already has, gain an authorization check.

The server allows the download only when it would already reveal the bound object to the caller:
- A room event, including state created by create room or upgrade: current room membership and history visibility, including stripped state on an invite. Federation is allowed when the requesting server has at least one user who would be shown that event.
- A profile: the same rule the server already uses for profile reads. If profile reads are limited to users who share a room, the media is too.
- A to-device event: the sender, or the target user. A remote requester is allowed only when it is the target user's server, fetching for that user.
- Account data: only the owning user. No other user and no other server.

Pending media that is not yet attached is allowed only for its creator, and never over federation.

If the caller is not allowed to see it, or the media does not exist, or it was removed, the response is 404. It is not 403, and it is not 410. A 403 or a 410 would confirm that the id exists.

A content hash is not an access key. No endpoint accepts a hash. A caller who knows the SHA-256 of a file still cannot download it without a media id they are allowed to use.

On success, the server includes a [Content-Digest](https://www.rfc-editor.org/rfc/rfc9530) of the bytes (sha-256). Clients and servers that already have those bytes may send `If-None-Match` or stop after the headers. The digest is present only on an authorized response. It is how a client recognizes a sticker or an avatar it already has, including identical ciphertext, without a second body download. It is not written into the event, and it is not accepted as `attach_media`.

### 4. Federation `restrictions`

`GET /_matrix/federation/v1/media/download/{mediaId}` and the thumbnail endpoint gain an optional `restrictions` object in the JSON part of the multipart/mixed response.

If `restrictions` is absent, the media is legacy unrestricted media. If it is present, it is a JSON object with exactly one of:

- `event_id`: the room event or to-device event this id is bound to
- `profile_user_id`: the user whose profile this id is bound to
- `user_id`: the single user who may download it (account data)

Any other shape, including two of those fields, is an unknown restriction. The requesting server must not serve the media to anyone.

The requesting server checks the restriction before serving the bytes, including from its cache. It stores the restriction with the bytes. When it later receives a redaction for that `event_id`, it stops serving the media and deletes its copy. If an operator policy retains the bytes, they stay unserved. When the server drops the room, it deletes the room's cached media. It does not wait for the origin to push a delete.

A federation download of pending, unbound media is 404. The origin does not hand out a restricted file that has no binding.

### 5. Copy

`POST /_matrix/client/v1/media/copy/{serverName}/{mediaId}` mints a new local media id for bytes the caller is allowed to download. The body is a JSON object. The query parameter `timeout_ms` matches download. The default is 20000. The server uses it only when it still has to fetch the bytes. A local file ignores it.

The response is `{ "content_uri": "mxc://..." }`. The caller is the creator of the new id and may attach it. The original id is unchanged and stays bound to its original event or profile.

If the caller cannot access the source, or it does not exist, the response is 404.

Clients use `/copy` when forwarding, when sending a sticker or custom emoji into a new event, and when editing. An `m.replace` must copy. The new event then has its own ids, so a server that checks only the new event can still authorize the download, and the visible fallback is a normal message with its own binding. Pointing the edit at the original id is not allowed. The original id remains bound to the original event.

`/copy` does not decrypt. Two copies of the same ciphertext share storage. That correlation is visible to the server operator. It is not visible to another user, because the digest is only returned on a download that user is allowed to make.

### 6. Profiles and membership events

`PUT /profile/{userId}/avatar_url?attach_media={localMediaId}` attaches that id to the profile. The `avatar_url` in the JSON body is the `mxc://` URI for that same local id. The server checks that they match. A mismatch is 400. Any other failure to attach is 404. Setting the same avatar again is idempotent and returns 200.

The server generates `m.room.member` events when its user joins a room or changes avatar. Each of those events gets its own copy, created by the server, bound to that member event. The profile object stays bound to the profile. Old member events keep their own copies for as long as those events remain visible. Changing the avatar does not erase history.

The extra member events in many rooms are unchanged. The bytes are not downloaded again when the `Content-Digest` matches a file the client already has.

Invite, create room, and upgrade follow the same server-copy rule for avatars the server itself writes onto membership. Media the client put in the request is attached from `attach_media`, not copied a second time.

### 7. Removal

When an event is redacted, the origin removes that media id's authorization before the redaction is federated, and deletes the bytes when no remaining media id references them. A later download is 404.

When the server has no local user left in the room, it deletes the room's events and the media bound to them. The same applies when every local user in the room is deactivated, including a GDPR Article 17 erasure.

Remote servers learn this by seeing the redaction, or by dropping the room, as in section 4. There is no separate federation purge API.

### 8. URL preview

[GET /_matrix/client/v1/media/preview_url](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediapreview_url) returns a different MXC for each requesting user. That id is restricted with `user_id` set to the requester. Another user previewing the same URL gets another id. The bytes may be stored once. The preview MXC is not unrestricted, and it is not served on `/_matrix/media/v3`.

### 9. Who answers the visibility question

Authorization uses room membership, history visibility, and profile rules. Those live on the homeserver. Download is a homeserver endpoint. A process that only stores blobs does not expose its own public download API. The homeserver checks access, then reads the bytes. An external store is an internal detail. It does not need a second Matrix endpoint, and it must not answer a client on its own.

### 10. Storage

Deduplication is internal. It is not a client-visible identifier, and this MSC does not add a feature flag for it.

New bytes are stored by SHA-256. Local and remote objects do not share a file. Every storage provider is given the same relative path the local store uses, which is the upstream Synapse rule, extended with a hash key:

- Local file: local_content/by-sha/{sha[0:2]}/{sha[2:4]}/{sha[4:]}
- Local thumbnail: local_thumbnails/by-sha/{sha[0:2]}/{sha[2:4]}/{sha[4:]}/{width}-{height}-{type}-{method}
- Remote file: remote_content/by-sha/{server}/{sha[0:2]}/{sha[2:4]}/{sha[4:]}
- Remote thumbnail: remote_thumbnail/by-sha/{server}/{sha[0:2]}/{sha[2:4]}/{sha[4:]}/{width}-{height}-{type}-{method}

by-sha is a new directory, so it does not collide with a media id or a server name. URL previews stay under `url_cache/` and are not hash keys.

Existing deployments keep the `media-id` paths from v1.19 (local_content/{mediaId}, remote_content/{server}/{fileId}, and the thumbnail directories). The server does not rewrite the store when this lands, and there is no configuration switch.

Read, replace, and delete all take both paths:
- Read the hash path, then the media-id path, then the legacy remote thumbnail name that omitted the method. A row with no hash uses only the media-id path. Ask the local disk and each storage provider with those same relative paths, in that order.
- Write a new file to the hash path. If that path already exists, do not open it and do not upload it again. Publish by writing a temporary file and renaming it into place. If the destination appears during the rename, keep the destination and delete the temporary file. A remote download may land on the media-id path until the hash is known, then move to the hash path if it is empty, or delete the extra media-id file if the hash path is already there. Providers store the relative path that was kept. Storing a hash key is create-if-absent.
- Delete always removes that media id's media-id path, on the local disk and on every provider. Delete the hash path only when it is the last reference on that side. The local count ignores URL-preview rows. The remote count is per origin server. Then delete that same hash key on every provider.

A failed spam check or a failed quota check deletes only the temporary file. It does not delete a hash path that another media id already uses.

The same bytes uploaded twice on this server share one local file. The same bytes downloaded twice from the same remote server share one remote file. A local upload and a remote download of the same bytes stay in different directories. The same bytes from two remote servers stay in different directories.

## Security considerations

The homeserver records which event or profile a media id is bound to. In an encrypted room that tells the operator which ciphertext belongs to which event. That is the metadata leak of this design. The Content-Digest does not add to it for other users, because they receive the digest only when they are already allowed to download.

Operators can see that two copies are the same ciphertext, because storage is by hash. Users cannot query by that hash.

404 on attach, copy, and download is what stops a caller from enumerating ids.

## Unstable prefix

Until this is stable, implementations use `org.matrix.mscxxxx`:

| Stable | Unstable |
| --- | --- |
| `/_matrix/client/v1/media/upload` | `/_matrix/client/unstable/org.matrix.mscxxxx/media/upload` |
| `/_matrix/client/v1/media/create` | `/_matrix/client/unstable/org.matrix.mscxxxx/media/create` |
| `attach_media` | `org.matrix.mscxxxx.attach_media` |
| `restrictions` | `org.matrix.mscxxxx.restrictions (event_id, profile_user_id, and user_id are not prefixed)` |
| `/_matrix/client/v1/media/copy/{serverName}/{mediaId}` | `/_matrix/client/unstable/org.matrix.mscxxxx/media/copy/{serverName}/{mediaId}` |

Servers advertise `org.matrix.mscxxxx` on `/_matrix/client/versions`. The storage layout in section 10 is not part of that flag. It applies to every new file.
