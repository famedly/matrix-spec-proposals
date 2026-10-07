# MSCXXXX: Attaching media to events

This is revised version of [MSC3911](https://github.com/matrix-org/matrix-spec-proposals/pull/3911) which is an alternative to [MSC3910](https://github.com/matrix-org/matrix-spec-proposals/pull/3910).

Currently, access to media in Matrix has the following problems:

* The only protection for media is the obscurity of the URL, and URLs are easily leaked.
* When a media event is redacted, the media it used remains visible to all. [synapse#1263](https://github.com/element-hq/synapse/issues/1263)
* If a user requests GDPR erasure, their media remains visible to all.
* When all users leave a room, their media is not deleted from the server.

This proposal builds on [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) which adds authentication to media download, to require that the authenticated user is *authorised* to access the requested media. Spec v1.19 already has authenticated download ([`GET /_matrix/client/v1/media/download/{serverName}/{mediaId}`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediadownloadservernamemediaid) and the federation equivalents from MSC3916).

**This proposal assumes that media storage has already been deduplicated, so that media is stored by its hash.**

It does not yet stop a download of media whose hash is already known. That can be addressed later.

[MSC4396: Inline linked media](https://github.com/matrix-org/matrix-spec-proposals/blob/39ceda9a358ff4152300da244b3c919ab6752d9a/proposals/4396-inlined-linked-media.md) link media in different way. In this proposal, uploads and sends happens in one `multipart/mixed` request. The server writes an `m.media` array into the event and the client refers to entries with `url_index`, because it does not know the MXC URI until the response. There is no copy API. A reaction or a sticker use points at media already linked to a pack event(redacting the pack disables every use). Federation carries visibility as an event ID, if it is missing, the media is world-readable. With this approach, server is the authority for the link, so custom event shapes stop being a parsing problem, and the gap between upload and attach disappears. The cost is a breaking change for every sender: large files become part of the send request, and existing clients cannot fill in url themselves.

## Proposal

### Overview

After an item of media is uploaded, it must be attached to an event via the repeated `attach_media` query parameter on the `/send` API. That event may be a to-device event or an in-room event.
User profiles are a separate case: they are not events, though in most cases their contents are copied into membership events. A given piece of media is visible to a user only if that user can see the corresponding event.

### Detailed spec changes

#### 1. New media upload endpoint

  * A new media upload endpoint is defined, `POST /_matrix/client/v1/media/upload`. It is based on the existing [`/_matrix/media/v3/upload`](https://spec.matrix.org/v1.19/client-server-api/#post_matrixmediav3upload) endpoint, but media uploaded this way is not initially viewable, except to the user who uploaded it. This is referred to as a restricted media item. The same model can be applied to `/_matrix/media/v1/create`, as added by [MSC2246: Asynchronous media uploads](2246-asynchronous-uploads.md).

  * The existing endpoint is deprecated. Media uploaded via the deprecated endpoint is unrestricted.

#### 2. Attaching media

  * To close the gap between creating restricted media and attaching it, a user may attach only media that they created. For a media ID returned by the copy API, the creator is the user who called `/copy`, not the creator of the original media ID.

  * One event can have several pieces of media attached. Servers define the maximum. (An image plus a thumbnail needs at least two, MSC4396 uses five) Each piece of media is attached to only one event. To attach the same content to another event, it is copied and assigned a new local media ID.

  * The methods for sending events ([`PUT /_matrix/client/v3/rooms/{roomId}/state/{eventType}/{stateKey}`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv3roomsroomidstateeventtypestatekey),
  [`PUT /_matrix/client/v3/rooms/{roomId}/send/{eventType}/{txnId}`](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3roomsroomidsendeventtypetxnid),
  [`PUT /_matrix/client/v3/sendToDevice/{eventType}/{txnId}`](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3sendtodeviceeventtypetxnid),
  [`PUT /_matrix/client/v3/user/{userId}/account_data/{type}`](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3useruseridaccount_datatype),
  [`POST /_matrix/client/v3/createRoom`](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3createroom),
  [`POST /_matrix/client/v3/rooms/{roomId}/upgrade`](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3roomsroomidupgrade), and
  [`POST /_matrix/client/v3/rooms/{roomId}/invite`](https://spec.matrix.org/v1.19/client-server-api/#post_matrixclientv3roomsroomidinvite))
  are extended with a repeated query parameter, `attach_media`. Each value is a local media ID: the same path component used by [this endpoint](https://spec.matrix.org/v1.19/client-server-api/#put_matrixmediav3uploadservernamemediaid). The server name and the `mxc://` prefix are omitted. The media being attached is always on the caller's homeserver, so both are redundant, and a media ID is only unique within one server. A media ID from another server is not valid in `attach_media`; it could collide with a local one.

  * To reuse media that lives on another server, the client copies it first. The copy API returns a new local media ID, and the client passes that local media ID in `attach_media`.

  * For example, sending an image and its thumbnail:

    `PUT /_matrix/client/v3/rooms/{roomId}/send/m.room.message/{txnId}?attach_media=abc123&attach_media=def456`

  * If any `attach_media` value cannot be attached, the server responds with 404. This is the only response for that failure. It covers an unknown media ID, a media ID from another server, media the caller did not create, media that is already attached, and media the caller is not allowed to access.

    Those cases are easy to split into different errors: 400 `M_INVALID_PARAM` when the identifier is unknown or already attached, and 404 when it is already attached and the caller cannot access it. That split tells the caller whether the identifier exists. A 404 would mean it exists and is inaccessible, and a 400 would mean something else. Download, below, already uses 404 for every case in which the caller must not see the media, so that a client cannot enumerate media by probing identifiers. Attach uses that same status, for the same reason.

    A malformed `attach_media` value, such as an empty string, is still 400 `M_INVALID_PARAM`. That error means the request itself is invalid. It does not describe a particular media item.

  * Sending an event in this manner associates the media with the sent event. From then on, the media can be seen by any user who can see the event itself.

  * Servers should ensure that sending an event remains idempotent. In particular, if a client sends an event with a media attachment and then repeats the operation with identical parameters, the server must return a 200 response with the original event ID, even though the media has already been attached. This retry is checked before the 404 above. A different transaction ID, with the same already-attached media ID, cannot be attached again and receives that 404.

  * Alternatively, the same `attach_media` query parameter on [`PUT /_matrix/client/v3/profile/{userId}/{keyName}`](https://spec.matrix.org/v1.19/client-server-api/#put_matrixclientv3profileuseridkeyname) attaches the media to the user's profile instead of to an event. The value is again a local media ID. A value that cannot be attached receives the same 404 as on the send endpoints.

  * If the media is not attached to either an event or a profile within 24 hours (the same window as asynchronous uploads), the server should delete the uploaded media.

#### 3. Additional checks on `/download` and `/thumbnail` endpoints

  * The new `/download` and `/thumbnail` endpoints added in [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) are updated so that the server must check whether the requesting user or server is allowed to see the corresponding event or profile. If the caller is not allowed to see it, or the media does not exist, the server responds with 404. The same status is used for both, to prevent metadata enumeration. A 403 would confirm that the media exists.

  * Access to the event or profile depends on both history visibility and current room membership. It is possible they disagree. Invites need the avatar before the user is a member. Room creation and upgrade create those state events on the server, so the server has to copy the media itself. A workable rule is, the caller may download the media when the server would already reveal that event to them, including current state and stripped invite state. Federation stays "the requesting server has at least one user who would be shown that event".

  * Authorisation depends on room membership and event visibility, which live on the homeserver. A media service that only stores blobs needs a homeserver-specific API to ask that question. That is an implementation split, not a missing endpoint.

#### 4. Federation API returns a `restrictions` property

  * The `/_matrix/federation/v1/media/download` and `/_matrix/federation/v1/media/thumbnail` endpoints specified by [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md) are extended. The returned JSON object may include a `restrictions` property.

  * If there is no `restrictions` property, the media is legacy unrestricted media. Otherwise, `restrictions` must be a JSON object with one of the following properties:
    * `event_id`: the event ID of the event the media is attached to.
    * `profile_user_id`: the user ID of the user whose profile the media is attached to.
  It is invalid for both `event_id` and `profile_user_id` to be set.

  * The requesting server must check the restrictions, and return the requested media only to users who are allowed to view the relevant event or profile. If the requesting server caches the media, it must also cache the restrictions.

  * If neither `event_id` nor `profile_user_id` is present, the requesting homeserver should assume that an unknown restriction is present, and must not allow access to any user.
  An example response:
  ```
  Content-Type: multipart/mixed; boundary=gc0p4Jq0M2Yt08jU534c0p
  --gc0p4Jq0M2Yt08jU534c0p
  Content-Type: application/json
  { "restrictions": {
      "event_id": "$Rqnc-F-dvnEYJTyHq_iKxU2bZ1CI92-kuZq3a5lr5Zg"
  }}
  --gc0p4Jq0M2Yt08jU534c0p
  Content-Type: text/plain
  This media is plain text. Maybe somebody used it as a paste bin.
  --gc0p4Jq0M2Yt08jU534c0p
  ```

  * If the event is redacted, the attached media should be purged from the local server.

#### 5. New "media copy" API

  * Conceptually, the API makes a new copy of a media item. In practice, the server will probably create a new reference to an existing media item, but that is an implementation detail.

  * A new endpoint is defined: `POST /_matrix/client/v1/media/copy/{serverName}/{mediaId}`. The body of the request must be a JSON object.

    The endpoint takes the same [`timeout_ms`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediadownloadservernamemediaid) query parameter as download. Copying media that is not already stored locally is a download: the server has to fetch the bytes before it can mint a local media ID. `timeout_ms` is how long the client is willing to wait for that fetch, in milliseconds. The default is 20000 (20 seconds). When the bytes are already stored locally, the server ignores `timeout_ms`, as download does when the media is already available.

    Treating a local-only copy as a second endpoint, with no `timeout_ms`, would be a different API. A client forwarding a remote file would then have to download it and upload it again, which is the fetch `/download` already performs. This proposal uses that same parameter on `/copy` instead.

  * The response is a JSON object with a required `content_uri` property, whose value is a new MXC URI for the media. The user who called `/copy` is the creator of that new local media ID. The creator of the original media ID is unchanged.

    This is required by the attach rule. A user may attach only media they created, and the new media ID is meant to be attached like a newly uploaded item. If the original uploader stayed the creator of the copy, the user who requested the copy would be forbidden from attaching it, and `/copy` could not be used to forward media.

  * The new media item can be attached to a new event by passing its local media ID as `attach_media`, and otherwise behaves like a newly uploaded item, including the 24 hour window in which it must be attached. A user may copy only media they can access. If the user is not allowed to access the media, or the media does not exist, the server responds with 404. This is the same status download uses. A 403 on copy, next to a 404 on download, would let a caller distinguish "this exists, but you may not see it" from "this does not exist".

  * Clients use this copy API when forwarding events that have media attachments, including `m.sticker` events and custom emoji. This mechanism, rather than allowing one piece of media to be attached to many events, keeps the list of events for a given piece of media from growing without bound. An ever-growing list would make it hard for servers to cache media reliably and to apply the correct access restrictions.

  * Because `/copy` cannot decrypt the media, a homeserver can still associate copies of encrypted media with one another.

  * Open question: should copying media during an edit be forbidden?

#### 6. Autogenerated `m.room.member` events

  * Servers generate `m.room.member` events with an `avatar_url` whenever one of their users joins a room or changes their profile picture.

  * Each such event must use a different copy of the media item, in the same way as the media copy API described above.

#### 7. Backwards compatibility mechanisms

  For backwards compatibility with older clients and requesting servers, servers may, for a short time, allow unauthenticated access via the deprecated `/_matrix/media/v3` endpoints, even for restricted media.

#### 8. URL preview

  * The [`/preview_url`](https://spec.matrix.org/v1.19/client-server-api/#get_matrixclientv1mediapreview_url) endpoint returns an object that references an image for the previewed site. Servers are expected to keep treating such media as unrestricted, at least for local users.

  * It would also be legitimate for a server to return a different `mxc:` URI for each requesting user, and to allow each user access only to their own URI.

### Applications

This section discusses the effect of the proposal on the ecosystem: the changes existing implementations will need, and the later work it makes possible. Encrypted history sharing is a further case, because it uses media blobs that are not tied to a specific event. A separate mechanism for creating media to share with one other user or device might help. See [MSC4268: Sharing room keys for past messages](https://github.com/matrix-org/matrix-spec-proposals/blob/rav/proposal/encrypted_history_sharing/proposals/4268-encrypted-history-sharing.md) or [issue](https://github.com/element-hq/element-meta/issues/39).

#### IRC/XMPP bridges

These bridges were discussed in [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md), but this proposal adds a further problem. These bridges currently use the content repository as a paste bin: large text messages are uploaded as plain-text media, and a link is then sent to the remote network. That becomes a problem, because servers may remove any media that is not attached to an event.

Possible solutions include:

* the bridge hosting its own content repository for this use case
* using an external service
* special-casing the bridge's application-service user so that it may upload unrestricted media
* attaching media to messages after the fact (for example, attaching the paste-bin copy to the original long message)
* tracking the room and/or user the media came from, so that application services do not have to upload entirely unrestricted media

#### Redacting events

* Under this proposal, servers can determine which media an event references when that event is redacted, and add that media to a list to be cleaned up.

* If the server no longer has a local user in the room, it should remove the room, the events, and any attached media.

* This also applies if every user in a room is deactivated, either by a GDPR Article 17 request or by a self-service account deactivation. In that case, every event in the room, and all media referenced by those events, should be removed.

* "Purged from the local server" leaves the status code, client caches, and admin or legal retention unspecified. A remote server is told to cache restrictions with the file and is never told to refresh them. It only learns about a redaction if it is still participating in the room. That stale cache is a consequence of the one-event binding: the list does not grow, and it also does not update. The proposal can require local downloads to fail with 404, allow servers to retain bytes for an operator, and state that remote deletion is best-effort until the caching server sees the redaction or drops the room.

## Potential issues

* Because each `m.room.member` event references the avatar separately, changing an avatar causes an even larger traffic storm when the user is in many rooms.

* Because each copy has its own MXC URI, clients and homeservers cannot tell that two references are the same bytes until after the download. Repeated use of the same content (for example, sticker spam) therefore transfers the same file many times. A content hash advertised on download — via a header such as `X-Matrix-Media-Hash`, `Repr-Digest`, or an `ETag` usable with `If-None-Match` — would let a receiver abort a `GET` after the headers, or issue a `HEAD` first, when that hash is already stored. Deduplicating local storage after download remains an implementation detail. Choosing a hash early could also affect how servers name stored files, and it is unclear how this would work for encrypted media, or whether it would leak more metadata or add cache-poisoning risk beyond this proposal. This can be specified in a follow-up MSC.

* With `m.replace`, copying every attached media item can be expensive. Skipping the copy leaves the new event pointing at media bound to the original event, so a server that only checks the new event will refuse the download, and clients that render the edit fallback as an ordinary message break. Both outcomes follow from rules already in the proposal, which is why the question is still open.

* Custom event formats, particularly those used by widgets, may prevent clients from identifying attached media automatically. Collaborative events that reference several media items also make edits inefficient, because `m.replace` must repeat every unchanged reference. Widgets can keep using unrestricted media, or separate container events, as workarounds; neither is ideal. [MSC4039: Access the Content Repository With the Widget API](https://github.com/nordeck/matrix-spec-proposals/blob/nic/feat/widgetapi-upload-files/proposals/4039-widget-api-media.md)

* For stickers or emoji, a popular room may contain tens of references to the same object, which causes tens of downloads because the client cannot tell them apart. Including a hash of the file in the event would let clients recognise locally that the objects have the same content. See [MSC4027: Custom Images in Reactions](https://github.com/beeper/matrix-spec-proposals/blob/custom-images-in-reactions/proposals/4027-custom-images-in-reactions.md) for more detail.

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
  Nothing currently requires these to match. Users may be unable to download the displayed image while being allowed to download an unrelated one. The homeserver cannot reliably check this. Event formats can be custom, and encrypted event contents are unreadable to the server.

    * attach_media is the authoritative access-control list.
    * Clients must ensure that it contains every restricted media ID referenced by the event.
    * The server does not inspect event content to verify this.
    * Incorrect or missing attachments are a client error and may produce broken media.
    * Custom or encrypted formats remain supported because the client explicitly supplies the IDs.

* Who may download the profile object is unspecified. Each new m.room.member needs its own copy, and the text does not say the homeserver creates those copies. Old member events keep their own copies for as long as those events remain visible, which is consistent with the one-event rule and means a profile change does not erase history.

* Backwards compatibility allows unauthenticated `/_matrix/media/v3` access to restricted media "for a short time", with no bound. Restricted media can simply never be served on the deprecated endpoint, and legacy unrestricted media stays there.

## Alternatives

* Have the upload endpoint return a nonce, which the send endpoint accepts in place of the MXC URI. The only apparent advantage is that a nonce could be smaller, and so slightly fewer bytes would be sent.

* Use a content token for each piece of media, and require clients to provide it, as in [MSC3910](https://github.com/matrix-org/matrix-spec-proposals/pull/3910).

## Security considerations

Letting servers track the relationship between events and media leaks metadata, especially in end-to-end-encrypted rooms.

## Unstable prefix

* While this MSC is not considered stable, implementations should use the following mapped values.

| Stable | Unstable |
| --- | --- |
| `/_matrix/client/v1/media/upload` | `/_matrix/client/unstable/org.matrix.mscXXXX/media/upload` |
| `/_matrix/client/v1/media/create` | `/_matrix/client/unstable/org.matrix.mscXXXX/media/create` |
| `attach_media` (repeated query parameter; each value is a local media ID) | `org.matrix.mscXXXX.attach_media` |
| `restrictions` | `org.matrix.mscXXXX.restrictions` (`event_id` and `profile_user_id` are not prefixed) |
| `/_matrix/client/v1/media/copy/:serverName/:mediaId` | `/_matrix/client/unstable/org.matrix.mscXXXX/media/copy/:serverName/:mediaId` |

* Servers should provide a feature flag so that clients can enable support.

* [Synapse implementation](https://github.com/famedly/synapse/pull/131) (written for Gematik, for the German TI-Messenger).
  [Client implementation in the Matrix Dart SDK](https://github.com/famedly/matrix-dart-sdk/pull/2134).
  The implementation needs no client-side changes beyond updating the SDK and passing messages through `Event.copyMediaInContent` when forwarding events.
  One open question is how to handle media when an event is edited, especially inline media. In theory the media would not have to be copied, because anyone who could see the original event can see the edit.
  One downside of this MSC is that it couples the media repository fairly tightly to the homeserver, so matrix-media-repo may need additional server-specific APIs.

## Dependencies

* [MSC3916: Authentication for media access, and new endpoint names](3916-authentication-for-media.md)

* Deduplication of the media storage, so that media is stored by its hash.

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
