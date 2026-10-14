# MSCXXXX: Attaching media to events

## Introduction

This is revised version of [MSC3911](https://github.com/matrix-org/matrix-spec-proposals/pull/3911) which is an alternative to [MSC3910](https://github.com/matrix-org/matrix-spec-proposals/pull/3910).

Currently, access to media in Matrix has the following problems:

* The only protection for media is the obscurity of the URL, and URLs are easily leaked.
* When a media event is redacted, the media it used remains visible to all. [synapse#1263](https://github.com/element-hq/synapse/issues/1263)
* If a user requests GDPR erasure, their media remains visible to all.
* When all users leave a room, their media is not deleted from the server.

This proposal builds on [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) which adds authentication to media download, to require that the authenticated user is *authorised* to access the requested media. Spec v1.19 already has authenticated download ([`GET /_matrix/client/v1/media/download/{serverName}/{mediaId}`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediadownloadservernamemediaid) and the federation equivalents from MSC3916).

## Proposal

After an item of media is uploaded, it is not viewable, except by the user who uploaded it, until it is attached. A media must be attached to an event via the repeated `attach_media` query parameter on the `/send` API. That event may be a to-device event or an in-room event. To-device events and account data use the same parameter. They are not room events, so their visibility rule is narrower and is fixed below.
User profiles are a separate case: they are not events, though in most cases their contents are copied into membership events. A given piece of media is visible to a user only if that user can see the corresponding event.

### 1. New media upload endpoint

  * A new media upload endpoint is defined. `POST /_matrix/client/v1/media/upload` behaves like [POST /_matrix/media/v3/upload](https://spec.matrix.org/v1.19/client-server-api/#post_matrixmediav3upload), except the new object is restricted. Restricted Media is not initially viewable, except to the user who uploaded it. The same model can be applied to `/_matrix/media/v1/create`, as added by [MSC2246: Asynchronous media uploads](2246-asynchronous-uploads.md).

  * Only the uploader can download a restricted object before it is attached.

  * If the media is created but not attached to either an event or a profile within 24 hours (the same window as asynchronous uploads), the server deletes it.

  * The paste-bin pattern used by IRC and XMPP bridges was already broken by MSC3916: users on the remote network have no Matrix account, so they cannot use authenticated download. This MSC only adds one new constraint, that unattached uploads are deleted after 24 hours. Bridges that forward media or long messages to a remote network host those bytes themselves, at their own HTTP endpoint. Other options are listed under [Alternatives](#alternatives).

  * The existing upload endpoint is deprecated. It keeps working, and media uploaded through it stays unrestricted. Restricted media is never served from the unauthenticated `/_matrix/media/v3` download or thumbnail endpoints. There is no grace period.

### 2. Attaching media

  * To close the gap between creating restricted media and attaching it, a user may attach only media that they created. For a media id returned by the copy API, the creator is the user who called `/copy`, not the creator of the original media id.

  * To attach the same content to another event, it is copied and assigned a new local media id.

  * One event can have several pieces of media attached. Servers define the maximum, and MUST allow at least 10. Each piece of media is attached to only one event. More than the maximum is `400 M_LIMIT_EXCEEDED`.

  * The methods for sending events are extended with a repeated query parameter, `attach_media`.:
  - [PUT /_matrix/client/v3/rooms/{roomId}/send/{eventType}/{txnId}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3roomsroomidsendeventtypetxnid)
  - [PUT /_matrix/client/v3/rooms/{roomId}/state/{eventType}/{stateKey}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3roomsroomidstateeventtypestatekey)
  - [PUT /_matrix/client/v3/sendToDevice/{eventType}/{txnId}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3sendtodeviceeventtypetxnid)
  - [PUT /_matrix/client/v3/user/{userId}/account_data/{type}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3useruseridaccount_datatype)
  - [PUT /_matrix/client/v3/user/{userId}/rooms/{roomId}/account_data/{type}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3useruseridroomsroomidaccount_datatype)
  - [POST /_matrix/client/v3/createRoom](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3createroom)
  - [PUT /_matrix/client/v3/profile/{userId}/{keyName}](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3profileuseridkeyname)

  * [Room upgrade](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3roomsroomidupgrade) does not take `attach_media`. The server copies the media referenced by the state it transfers to the new room, such as the room avatar, and binds each copy to the new room's state event.

  * `attach_media` is value of a local media id, without server name and the `mxc://` prefix. The media being attached is always on the caller's homeserver, so both are redundant, and a media id is only unique within one server. A media id from another server is not valid in `attach_media`; it could collide with a local one. A value that contains /, mxc://, or is empty is malformed, and the whole request is `400 M_INVALID_PARAM`.

  * To reuse media that lives on another server, the client copies it first. The copy API returns a new local media id, and the client passes that local media id in `attach_media`.

  * For example, sending an image and its thumbnail:

    `PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}?attach_media=abc123&attach_media=def456`

  * A repeated transaction with the same media ids returns 200 and the original event id. That check happens before the 404. A new transaction id that presents an already-attached media id is 404.

  * If any well-formed `attach_media` cannot be attached, the server responds with 404 and attached none of them. This is the only response for that failure. It covers an unknown media id, a media id from another server, media the caller did not create, media that is already attached, legacy unrestricted media, and media the caller is not allowed to access. The caller cannot tell them apart.

  * Sending the event binds each media id to that event. A profile write binds the id to that user. Create room binds each media id to the one initial state event that references it. When the server has to put the same bytes on a second event, it copies first, then binds the new id. The client's original id is still bound to only one event.

  * Sending an event in this manner associates the media with the sent event. From then on, the media can be seen by any user who can see the event itself.

  * Alternatively, the same `attach_media` query parameter on [`PUT /_matrix/client/v3/profile/{userId}/{keyName}`](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3profileuseridkeyname) attaches the media to the user's profile instead of to an event. The value is again a local media id. A value that cannot be attached receives the same 404 as on the send endpoints.

  * Profile fields, account data, and unencrypted room events of a type the server understands are compared with `attach_media`. Only the following fields are inspected:

    | Content | Fields |
    | --- | --- |
    | `m.room.message` | `url`, `info.thumbnail_url`, and the `src` of every `<img>` tag in `formatted_body` when `format` is `de.famedly.custom.html` |
    | `m.sticker` | `url`, `info.thumbnail_url` |
    | `m.room.avatar` | `url`, `info.thumbnail_url` |
    | `m.room.member` | `avatar_url` |
    | Profile (`PUT /profile/{userId}/avatar_url`) | `avatar_url` |
    | Account data of any type (global and per-room) | every string value, at any depth of the content, that is an `mxc://` URI |

  * Inline images in `formatted_body` count. A client sending an HTML message with inline images must therefore include each of their media ids in `attach_media`, and the server rejects the event if it does not.

  * Account data has no fixed schema, so the server scans the whole content for MXC URIs instead of named fields. Account data is visible only to its owner, so the scan does not reveal anything to another user.

  * The comparison covers only **restricted local media**: an MXC URI whose server name is this homeserver and whose media id was created through the restricted upload, create, or copy endpoints. MXC URIs for other servers and for legacy unrestricted local media are ignored, so events that reference old uploads, or that forward legacy media, can still be sent without attaching anything.

  * Every restricted local media id found in those fields must appear in `attach_media`, and every `attach_media` value must appear in those fields. A mismatch is `400 M_INVALID_PARAM`, because both sides were supplied by this caller and the error does not reveal another user's media.

  * Legacy unrestricted media cannot be attached. If an `attach_media` value names legacy unrestricted media, the server responds with 404 as for any other media that cannot be attached. Its URL may already have been shared, so restricting it afterwards would protect nothing.

  * `m.room.encrypted` and event types the server does not understand are not parsed. For those events, `attach_media` is the only access list and the client is responsible for the match. Encrypted content stays opaque, which is the point of encryption.

  * Widgets send events through the Widget API rather than calling `/send` directly, so they cannot pass `attach_media` themselves. Widget support requires the Widget API's `send_event` action to accept an `attach_media` list, and its upload action to use the restricted upload endpoint, as in [MSC4039: Access the Content Repository With the Widget API](https://github.com/nordeck/matrix-spec-proposals/blob/nic/feat/widgetapi-upload-files/proposals/4039-widget-api-media.md). Until then, widgets keep using unrestricted media.

### 3. Additional header and checks on `/download` and `/thumbnail` endpoints

  * The authenticated `/download` and `/thumbnail` endpoints from [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) which v1.19 already has, gain an authorization check.

  * The authenticated `/download` and `/thumbnail` endpoints MUST support `HEAD`, and MUST return `Content-Digest: sha-256=:<base64>:` on an authorized response. The value is padded base64, as required by [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530). Clients can check with `HEAD`, or abort a `GET` once the headers arrive.

  * Encrypted media already carries `file.hashes.sha256` (the ciphertext hash) inside the event, so clients can recognise duplicates locally with no server help.

  * Pending media that is not yet attached is allowed only for its creator, and never over federation.

  * If the caller is not allowed to see it, or the media does not exist, or was removed, the server responds with 404. The same status is used for both, to prevent metadata enumeration.

  * Media is visible exactly when the event/profile/etc it is bound to is visible. How a deployment splits the homeserver from its media store is out of scope.
    - A room event (including state created by create room or upgrade): the spec's existing event visibility rules. Federation is allowed when the requesting server has at least one user who would be shown that event.
    - **Exception for invites and knocks:** media referenced by events in `invite_room_state` or `knock_room_state` (the stripped state sent with invites and knocks) is visible to the invited or knocking user. Over federation, the origin allows a requesting server that has a user with a pending invite or knock. Without this exception, invite previews can't show the room avatar.
    - A profile: the same rule the server already uses for profile reads. If profile reads are limited to users who share a room, the media is too.
    - A to-device event: the sender, or a recipient. A remote requester is allowed only when it is a recipient's server, fetching for that user.
    - Account data and URL previews: only the owning user. No other user and no other server.

### 4. Federation API returns a `restrictions` property

  * The [`GET /_matrix/federation/v1/media/download`](https://spec.matrix.org/v1.19/server-server-api/#get_matrixfederationv1mediadownloadmediaid) and `GET /_matrix/federation/v1/media/thumbnail` endpoints specified by [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) are extended to return optional JSON object `restrictions`.

  * If there is no `restrictions` property, the media is legacy unrestricted media. Otherwise, `restrictions` must be a JSON object with one of the following properties:
    * `event_id`: the room event that this media is bound to
    * `profile_user_id`: the user whose profile this id is bound to
    * `user_id`: the single user who may download it (account data and URL previews)
    * `to_device`: an object with `sender` (one user ID) and `recipients` (a list of user IDs), for media bound to a to-device message

    Example of restrictions:
    ```
      { "restrictions": { "event_id": "$Rqnc-F-dvnEYJTyHq_iKxU2bZ1CI92-kuZq3a5lr5Zg" } }
      { "restrictions": { "to_device": { "sender": "@alice:a.org", "recipients": ["@bob:b.org"] } } }
    ```

  * It is invalid for more than one to be set or to have any unknown property. If restriction is invalid, the requesting server must not serve the media to anyone. The requesting server must check the restrictions, and return the requested media only to users allowed by them: users who can view the event or profile, the single `user_id`, or the `to_device` sender and recipients.

  * A federation download of pending, unbound media is 404. The origin does not hand out a restricted file that has no binding.

  * If the requesting server caches the media, it must also cache the restrictions. The requesting server checks the restriction before serving the bytes, including from its cache. It stores the restriction with the bytes.

  * The origin SHOULD set `Cache-Control: max-age` on the federation response. Once it expires, the requesting server MUST revalidate with the origin before serving the media again, and MUST delete its copy if the origin responds with 404.

  * When the requesting server receives a redaction for that `event_id`, it stops serving the media and deletes its copy. If an operator policy retains the bytes, they stay unserved. When the server purges the room, it deletes the room's cached media. It does not wait for the origin to push a delete.

  * The JSON metadata part also contains `sha256`: the SHA-256 of the media bytes, as unpadded base64 (the usual Matrix encoding, unlike the padded `Content-Digest` header). That part arrives before the media part, so the receiving server can stop reading if it already has the bytes. This prevents an avatar traffic storm when a user is in many rooms.

  An example response:
  ```
  Content-Type: multipart/mixed; boundary=gc0p4Jq0M2Yt08jU534c0p
  --gc0p4Jq0M2Yt08jU534c0p
  Content-Type: application/json
  { "restrictions": {
      "event_id": "$Rqnc-F-dvnEYJTyHq_iKxU2bZ1CI92-kuZq3a5lr5Zg"
    },
    "sha256": "47DEQpj8HBSa+/TImW+5JCeuQeRkm5NMpJWZG3hSuFU"
  }
  --gc0p4Jq0M2Yt08jU534c0p
  Content-Type: text/plain
  This media is plain text. Maybe somebody used it as a paste bin.
  --gc0p4Jq0M2Yt08jU534c0p
  ```

### 5. New "media copy" API

  * New endpoint `POST /_matrix/client/v1/media/copy/{serverName}/{mediaId}` is introduced. The body of the request must be a JSON object.

  * Copying media that is not already stored locally is a *download*: the server has to fetch the bytes before it can mint a local media id. So the endpoint takes the same query parameter [`timeout_ms`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediadownloadservernamemediaid) as download (default to 20000). When `serverName` is the local server, `timeout_ms` is ignored.

  * Then this copy endpoint generates new local media id. The response is `{ "content_uri": "mxc://..." }`. The caller is the creator of the new id and may attach it. The original id is unchanged and stays bound to its original event or profile. A `/copy` result is a new restricted object with its own 24 hour window, like a newly uploaded item.

  * If the caller cannot access the source, or it does not exist, the response is 404.

  * Clients use this copy API when forwarding events that have media attachments, including **`m.sticker` events** and **custom emoji**.

  * An edit (`m.replace`) MUST copy each piece of media it references. Pointing the edit at the original id is not allowed: the original id remains bound to the original event, and a user who can see the edit may not be able to see the original. With content-addressed storage, copying local media adds a database row rather than a second copy of the bytes.

  * This copy mechanism, rather than allowing one piece of media to be attached to many events, keeps the list of events for a given piece of media from growing without bound. An ever-growing list would make it hard for servers to cache media reliably and to apply the correct access restrictions.

  * `/copy` does not decrypt. Two copies of the same ciphertext share storage. That correlation is visible to the server operator. It is not visible to another user, because the digest is only returned on a download that user is allowed to make.

  * A batch form, `POST /_matrix/client/v1/media/copy`, copies several items in one request. It takes the same `timeout_ms` query parameter.
    - The request body is `{ "mxc_uris": ["mxc://...", ...] }`, with at most as many entries as the server's attachment maximum.
    - The response is `{ "content_uris": ["mxc://...", ...] }`, in the same order as the request.
    - The request is all-or-nothing. If any source cannot be copied, the response is 404 and no copies are created.

#### Implementation notes

This proposal needs the copy operations to be cheap. Servers SHOULD implement `/copy` without duplicating stored bytes, for example by content-addressed storage. These notes describe the Synapse implementation; other servers and storage providers may differ.

  * Deduplication is internal. It is not a client-visible identifier, and this MSC does not add a feature flag for it. This is totally backward compatible with existing media_id path storage.

  * New bytes are stored by SHA-256. Local and remote objects do not share a file. Every storage provider is given the same relative path the local store uses, which is the upstream Synapse rule, extended with a hash key:
    - Local file: local_content/by-sha/{sha[0:2]}/{sha[2:4]}/{sha[4:]}
    - Local thumbnail: local_thumbnails/by-sha/{sha[0:2]}/{sha[2:4]}/{sha[4:]}/{width}-{height}-{type}-{method}
    - Remote file: remote_content/by-sha/{server}/{sha[0:2]}/{sha[2:4]}/{sha[4:]}
    - Remote thumbnail: remote_thumbnail/by-sha/{server}/{sha[0:2]}/{sha[2:4]}/{sha[4:]}/{width}-{height}-{type}-{method}

  * `by-sha` is a new directory, so it does not collide with a media id or a server name. URL previews stay under `url_cache/` and are not hash keys.

  * Existing deployments keep the `media-id` paths from v1.19 (local_content/{mediaId}, remote_content/{server}/{fileId}, and the thumbnail directories). The server does not rewrite the store when this lands, and there is no configuration switch.

  * Read, replace, and delete all take both paths:
    - Read the hash path, then the media-id path, then the legacy remote thumbnail name that omitted the method. A row with no hash uses only the media-id path. Ask the local disk and each storage provider with those same relative paths, in that order.
    - Write a new file to the hash path. If that path already exists, do not open it and do not upload it again. Publish by writing a temporary file and renaming it into place. If the destination appears during the rename, keep the destination and delete the temporary file. A remote download may land on the media-id path until the hash is known, then move to the hash path if it is empty, or delete the extra media-id file if the hash path is already there. Providers store the relative path that was kept. Storing a hash key is create-if-absent.
    - Delete always removes that media id's media-id path, on the local disk and on every provider. Delete the hash path only when it is the last reference on that side. The local count ignores URL-preview rows. The remote count is per origin server. Then delete that same hash key on every provider.

  * A failed spam check or a failed quota check deletes only the temporary file. It does not delete a hash path that another media id already uses.

  * The same bytes uploaded twice on this server share one local file. The same bytes downloaded twice from the same remote server share one remote file. A local upload and a remote download of the same bytes stay in different directories. The same bytes from two remote servers stay in different directories.

  * Encrypted media needs no special case. The server never knows whether bytes are encrypted; it just hashes whatever it stores. Each Matrix encrypted attachment uses a fresh random key and IV, so two separate uploads of the same photo produce different ciphertext and won't deduplicate. They only deduplicate when the same ciphertext is reused, which is exactly what /copy and forwarding produce. That is the case that matters.

### 6. Profiles and membership events

  * The server generates `m.room.member` events when its user joins a room or changes avatar. Each of those events gets its own copy, created by the server, bound to that member event. The profile object stays bound to the profile. Old member events keep their own copies for as long as those events remain visible. Changing the avatar does not erase history.

  * The extra member events in many rooms are unchanged. Remote servers SHOULD use the `sha256` in the federation metadata to avoid re-downloading bytes they already hold. With that, the traffic storm becomes N small metadata responses instead of N full downloads.

  * `PUT /profile/{userId}/avatar_url?attach_media={localMediaId}` attaches that id to the profile. The `avatar_url` in the JSON body is the `mxc://` URI for that same local id. The server checks that they match. A mismatch is 400. Any other failure to attach is 404. Setting the same avatar again is idempotent and returns 200.

### 7. Removal

* When an event is redacted, the origin removes that media id's authorization before the redaction is federated, and deletes the bytes when no remaining media id references them. After redaction, local download, thumbnail, and copy return 404, the same as for a nonexistent media id.

* Users who left a room can still read its history up to the point they left, so leaving does not trigger removal. The server purges a room's events and the media bound to them when no local user is joined, invited, or knocking, and every local user who was in the room has either forgotten it (`/forget`) or been deactivated. Deactivation includes a GDPR Article 17 erasure and self-service account deactivation.

* Remote servers delete their cached copy when they see the redaction, when they purge the room, or when revalidation returns 404, as described in [Federation API returns a `restrictions` property](#4-federation-api-returns-a-restrictions-property). There is no separate federation purge API.

* Clients SHOULD evict cached media for redacted events.

* Servers MAY retain the bytes for legal or admin reasons, but MUST NOT serve them.

* Redacting the original does not remove media in its edits, so clients that redact an edited message should redact the edits too. See MSC3912 (relation-based redactions).

### 8. URL preview

  * [GET /_matrix/client/v1/media/preview_url](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediapreview_url) returns a different MXC URI for each requesting user. Each users are allowed to only their own URI.

  * That id is restricted with `user_id` set to the requester. Another user previewing the same URL gets another id. The bytes may be stored once. The preview MXC is not unrestricted, and it is not served on `/_matrix/media/v3`.

## Potential issues

* There are two separate places where media is mentioned:
  The event body says what media the client should display:
    ```
    {
      "url": "mxc://example.org/cat"
    }
    ```
  The attach_media query parameter tells the homeserver which media should be accessible:
    ```
    ?attach_media=dog
    ```
  For profile fields, account data, and the unencrypted event types listed in [Attaching media](#2-attaching-media), the server checks that the two match. For encrypted events and custom event types, the server cannot read or interpret the content, so nothing enforces the match. Users may then be unable to download the displayed image while being allowed to download an unrelated one. For those events:
    * `attach_media` is the authoritative access-control list.
    * Clients must ensure that it contains every restricted media id referenced by the event.
    * Incorrect or missing attachments are a client error and may produce broken media.
    * Custom or encrypted formats remain supported because the client explicitly supplies the IDs.

* This MSC couples the media repository fairly tightly to the homeserver, because authorization depends on room membership, history visibility, and profile rules. A separate media store such as matrix-media-repo needs a homeserver-specific API to ask those questions. That API is out of scope.

## Alternatives

* Have the upload endpoint return a nonce, which the send endpoint accepts in place of the MXC URI. The only apparent advantage is that a nonce could be smaller, and so slightly fewer bytes would be sent.

* Use a content token for each piece of media, and require clients to provide it, as in [MSC3910](https://github.com/matrix-org/matrix-spec-proposals/pull/3910).

* Bind member events directly to the profile media instead of copying it. This was rejected because changing an avatar would retroactively alter old member events.

* For bridges that use the content repository as a paste bin, instead of hosting the bytes themselves:
  * using an external service
  * special-casing the bridge's application-service user so that it may upload unrestricted media. This reopens the leak this MSC closes.
  * attaching media to messages after the fact (for example, attaching the paste-bin copy to the original long message)
  * tracking the room and/or user the media came from, so that application services do not have to upload entirely unrestricted media

[MSC4396: Inline linked media](https://github.com/matrix-org/matrix-spec-proposals/blob/39ceda9a358ff4152300da244b3c919ab6752d9a/proposals/4396-inlined-linked-media.md) link media in different way. In this proposal, uploads and sends happens in one `multipart/mixed` request. The server writes an `m.media` array into the event and the client refers to entries with `url_index`, because it does not know the MXC URI until the response. There is no copy API. A reaction or a sticker use points at media already linked to a pack event(redacting the pack disables every use). Federation carries visibility as an event ID, if it is missing, the media is world-readable. With this approach, server is the authority for the link, so custom event shapes stop being a parsing problem, and the gap between upload and attach disappears. The cost is a breaking change for every sender: large files become part of the send request, and existing clients cannot fill in url themselves.

## Security considerations

Letting servers track the relationship between events and media leaks metadata, especially in end-to-end-encrypted rooms.

The homeserver records which event or profile a media id is bound to. In an encrypted room that tells the operator which ciphertext belongs to which event. That is the metadata leak of this design. The Content-Digest does not add to it for other users, because they receive the digest only when they are already allowed to download.

Operators can see that two copies are the same ciphertext if storage is content-addressed. Users cannot query by that hash.

404 on attach, copy, and download is what stops a caller from enumerating ids.

  * A content hash is not an access key. No endpoint accepts a hash. A caller who knows the SHA-256 of a file still cannot download it without a media id they are allowed to use.

## Unstable prefix

* While this MSC is not considered stable, implementations should use the following mapped values.

| Stable | Unstable |
| --- | --- |
| `/_matrix/client/v1/media/upload` | `/_matrix/client/unstable/de.famedly.mscXXXX/media/upload` |
| `/_matrix/client/v1/media/create` | `/_matrix/client/unstable/de.famedly.mscXXXX/media/create` |
| `attach_media` (repeated query parameter; each value is a local media id) | `de.famedly.mscXXXX.attach_media` |
| `restrictions` | `de.famedly.mscXXXX.restrictions` (`event_id`, `profile_user_id`, `user_id`, and `to_device` are not prefixed) |
| `sha256` (federation metadata) | `de.famedly.mscXXXX.sha256` |
| `/_matrix/client/v1/media/copy/:serverName/:mediaId` | `/_matrix/client/unstable/de.famedly.mscXXXX/media/copy/:serverName/:mediaId` |
| `/_matrix/client/v1/media/copy` (batch) | `/_matrix/client/unstable/de.famedly.mscXXXX/media/copy` |

* Servers should provide a feature flag so that clients can enable support.

* [Synapse implementation](https://github.com/famedly/synapse/pull/131) (written for Gematik, for the German TI-Messenger).
  [Client implementation in the Matrix Dart SDK](https://github.com/famedly/matrix-dart-sdk/pull/2134).
  The implementation needs no client-side changes beyond updating the SDK and passing messages through `Event.copyMediaInContent` when forwarding events.

## Dependencies

* [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md)

* Widget support depends on [MSC4039: Access the Content Repository With the Widget API](https://github.com/nordeck/matrix-spec-proposals/blob/nic/feat/widgetapi-upload-files/proposals/4039-widget-api-media.md), extended with `attach_media` on `send_event`.

## Prior art

* Credit: this proposal is based on ideas from @jcgruenhage and @anoadragon453 at https://cryptpad.fr/code/#/2/code/view/oWjZciD9N1aWTr1IL6GRZ0k1i+dm7wJQ7juLf4tJRoo/

* [MSC3796](https://github.com/matrix-org/matrix-spec-proposals/issues/3796):
  a predecessor of this proposal

* [MSC2461](https://github.com/matrix-org/matrix-spec-proposals/pull/2461):
  adds per-user authentication but does not attempt to restrict access to individual items of media.

* [MSC2278](https://github.com/matrix-org/matrix-spec-proposals/pull/2278):
  Deleting attachments for expired and redacted messages

* [MSC1902](https://github.com/matrix-org/matrix-spec-proposals/pull/1902):
  Split the media repo into s2s and c2s parts

* [MSC2846](https://github.com/matrix-org/matrix-spec-proposals/pull/2846):
  Decentralizing media through CIDs

* [MSC3911](https://github.com/matrix-org/matrix-spec-proposals/pull/3911):
  Linking media to events

* [MSC3916](3916-authentication-for-media.md):
  Authentication for media access, and new endpoint names

* [MSC4027](https://github.com/beeper/matrix-spec-proposals/blob/custom-images-in-reactions/proposals/4027-custom-images-in-reactions.md):
  Custom Images in Reactions

* [MSC4396](https://github.com/matrix-org/matrix-spec-proposals/pull/4396):
  Inline linked media

* [Support deleting media from non-local storage providers](https://github.com/element-hq/synapse/pull/19665)
