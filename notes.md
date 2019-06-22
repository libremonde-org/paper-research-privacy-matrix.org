# Notes on privacy and data collection of Matrix.org

---

**This work is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).** See [Editorial Notes](#editorial-notes) for Attribution details.

---

> **DISCLAIMER:** This research and investigation work is based on several years of experience within the Matrix ecosystem and validation of facts via public and private communication. Reverse engineering was used to ensure some of the statements presented as facts regarding implementations are accurate.
>
> Nonetheless it is possible that a mistake has made its way in these notes. If that is the case, please get in touch with the author which will fix any factual mistakes in good faith. **We always encourage people not to trust statements at face value and always double-check for themselves.**

---

## TL;DR

`matrix.org` and `vector.im` receive a lot of private, personal and identifiable data on a regular basis, or metadata that can be used to precisely identify and/or track users/server, their social graph, usage pattern and potential location. This is possible both by the default configuration values in synapse/Riot that do not promote privacy, and by specific choices made by their developers to not disclose, inform users or resolve in a timely manner several known behaviours of the software.

Data sent on a potential regular basis based on a common web/desktop+smartphone usage **even with a self-hosted client and Homeserver**:

- The Matrix ID of users, usually including their username.
- Email addresses, phone numbers of the user and their contacts.
- Associations of Email, phone numbers with Matrix IDs.
- Usage patterns of the user.
- IP address of the user, which can give more or less precise geographical location information.
- The user's devices and system information.
- The other servers that users talks to.
- Room IDs, potentially identifying the Direct chat ones and the other user/server.

With default settings, they allow unrestricted, non-obfuscated public access to the following potentially personal data/info:

- Every file, image, video, audio that is uploaded to the Homeserver.
- Profile name and avatar of users.

See below for a detailed analysis.

If you have questions, want clarification, have spotted factual errors, or just want to discuss about privacy in Matrix and alternatives, come to our general Matrix room: `#kamax-matrix:kamax.io`

---

## Foreword

On the 12th of June 2019, after 5 years of hard work, Matrix.org released the v1.0 of the protocol, alongside v1.0.0 of the reference Homeserver implementation, synapse, and Riot v1.0.0 earlier in the year.

Having studied Matrix for more than 2 years, creating various implementations with [mxisd]() as our most notable software, we decided to make a review of the protocol on three values that we believe are fundamental to any open protocol:

- **Privacy:** Data/metadata should strictly only be accessible to user's intended recipient(s).
- **Decentralisation:** Network operations can not depend on a "central" set of server(s).
- **Security:** Access to data/metadata is authenticated and authorised by default, with passive or active safeguards (like End-to-End encryption) being on by default.

After the Matrix.org security breach, when an unauthorised person gained access to personal and private data, we believe this review to be critical and necessary on topics not often discussed.

### Purpose and Scope

**This document is a research paper for [The Grid Protocol](https://gitlab.com/thegridprotocol/home#the-grid) project, a fork of the Matrix protocol. Privacy is a core value of the project. This is an effort to document privacy points from our parent project, which can then be used to create guidelines/recommendations on what to do. It will be used for our own governing body/landing website and to evaluate Grid implementations for their priority for privacy.**

In this document, we will attempt to answer the following questions:

- Following the Matrix.org client/server recommendations/guides, can you be sure that your privacy is respected and your data secure?
- Using the default/recommended settings in the recommended clients/servers, where is your data/metadata flowing?
- If you were to create several direct message channels with others, will a 3rd party be aware of it and if yes, which?
- Are the default/recommended client/server explicit about where the data is flowing, which 3rd party is it shared with?
- Given the recent security breach of Matrix.org, what kind of information was accessed?

To do so, we will follow the most common setup and [the ones recommended by Matrix.org itself for self-hosting](https://web.archive.org/web/20190616122919/https://matrix.org/faq/#self-hosting):

- Self-hosting/installing your own client.
- Self-hosting your Homeserver.
- **NOT** self-hosting your own Identity server.

Matrix.org only gives self-hosting recommendations for client, Home and Identity server, a full Matrix stack also include  several other items. We will also cover those unspoken items in this review.

**We believe that privacy and security is only as good as default settings, software and recommendations given to users. This review will therefore be based on the spirit that *Default Settings Matter*, a view shared by core Web actors like [Mozilla](https://blog.mozilla.org/blog/2019/06/04/when-it-comes-to-privacy-default-settings-matter/)**.

**This review is based on the principle that no consent is given to any 3rd party service/Privacy policy/Terms of Services of any kind unless specifically prompted, following the expectations of users that their data and metadata is self-hosted and under their control. This is in line with EU GDPR laws.**

**Many people looking into Matrix are in dire need of privacy and security: activists, journalists, minorities, etc.  It is crucial that they are informed about possible privacy leaks that could later be used to identify them. Such identification usually leads to abuse, harassment, assault, threats, blackmail. It is crucial that users are not mislead. It is equally crucial that they are able to evaluate the real value of the ecosystem when used in a daily, real-world setting.**

### Setup

The following stack will be used as reference, with users connecting via web, desktop and smartphone clients:

- Client: Riot-web [v1.2.1](https://github.com/vector-im/riot-web/releases/tag/v1.2.1), Riot Desktop [v1.2.1](https://github.com/vector-im/riot-web/releases/tag/v1.2.1), Riot Android [v0.9.1](https://github.com/vector-im/riot-android/releases/tag/v0.9.1)
- Server: Synapse [v1.0.0](https://github.com/matrix-org/synapse)

This choice of client and server matches our knowledge and experience that it is representative of the overwhelming majority of the Matrix ecosystem.

#### Client choice

Riot has been chosen following its wide and repeated promotion on the Matrix.org website.

First, under [the predominant "Try now" button](https://web.archive.org/web20190620111226/http://matrix.org/) on Matrix.org on the top right corner. This button leads to [a dedicated landing page](https://web.archive.org/web20190620115310/http://matrix.org/try-now/) with the hereafter quoted first sentence:

> The easiest way to try Matrix is to use the [Riot Web](https://about.riot.im/) client in your browser

The page contains, at the bottom, a less noticeable sentence and link:

> Alternatively, you can find more clients and servers in [Discover](https://web.archive.org/web20190620115310/http://matrix.org/try-now/).

Second, The [Discover page](https://web.archive.org/web20190620121329/http://matrix.org/docs/projects/try-matrix-now/) is made of a filterable list of all known implementations which would allow a user to know about all available clients. The page itself list *Riot* and *Riot for Android and iOS* as first and second options in a manner which is clearly and directly noticeable. The first sentence of the page reads:

> To get started using Matrix, pick a client and join [#matrix:matrix.org](https://web.archive.org/web/20190620122127/https://matrix.to/). You can also check the [Matrix Clients Matrix](https://web.archive.org/web20190620122151/http://matrix.org/docs/projects/clients-matrix/) to see more detail.

Finally, The [Matrix Clients Matrix](https://web.archive.org/web20190620122151/http://matrix.org/docs/projects/clients-matrix/) page also list *Riot Web*, *Riot Android*, *Riot iOS* in first positions in their "compare" tables.

#### Server choice

Synapse has been chosen because it is the first server implementation listed on the [Discover page](), and is the only server feature-complete enough to be used on a day-to-day basis.



## Riot, the reference client implementation

### Overview

Riot is a software written by [New Vector Ltd](https://beta.companieshouse.gov.uk/company/10873661), a UK for-profit created in 2017 to support the people who created Matrix.org after Amdocs, the original founder of Matrix.org, cut them loose. While synapse and nearly all implementations are made under Matrix.org ownership are called "reference implementations". Riot is a "reference implementation" put together by another entity, using SDKs from Matrix.org. Since Matrix 1.0, The Matrix Foundation is officially the owner of anything under Matrix.org, making `matrix.org` and `vector.im` legally distinct.

Riot-web and Riot desktop share the same code base and both ship with [a default config file](https://github.com/vector-im/riot-web/blob/develop/config.sample.json) that contains several URLs/domains that we will explore in the various sections of this review.

This part of the review explores the default settings and behaviours specific to Riot as a Matrix client. Throughout the whole document, we will assume only the Homeserver URL was changed.

### Personal Identifiers

> **NOTE:** This section might contain specifics to Riot Web and Desktop, and overlaps with the Identity Server section.

The first two elements are the Homeserver and the Identity server: `matrix.org` and `vector.im`, respectively. If we are to examine the "Register" screen of Riot, we see that only the Homeserver URL is mentioned and selected, while `vector.im` is not displayed. Only if you click on "Change", you are prompted for both URLs.

On the screen allowing to set custom URLs, one can click on "What does this mean?". The identity server URL explanation reads:

> You can also set a custom identity server, but you won't be able to invite users by email address, or be invited by email address yourself.

So far we see that Identity servers are explained to not be important for the day-to-day usage of Matrix in the FAQ, or even reducing usability in Riot. **This is only true [if sydent is used](https://github.com/matrix-org/sydent/issues/22), the reference identity server.** Riot devs are well aware of the only other Identity server [mxisd]() which federates and [can include data from `vector.im`](https://github.com/kamax-matrix/mxisd/blob/master/docs/faq.md#will-i-loose-access-to-the-central-matrixorgvectorim-identity-data-if-i-use-mxisd). **This is highly misleading and pushes users not to even try to self-host Identity servers, or use another than the default, out of fear.** We will see later the importance of them in terms of privacy.

In a self-hosted scenario, following the recommendation of Matrix.org, only the Homeserver URL would be changed while keeping the Identity Server URL to `vector.im`.

When registering without an email, we are prompted with a **Warning!** sign and the following text:

> If you don't specify an email address, you won't be able to reset your password. Are you sure?

At this point, if we are to cancel and enter an email address in fear to be locked out of our account, we are prompted to validate it using a token/link sent by email. Riot does not give any kind of explanation that the Identity server has been contacted to validate the email, in this case `vector.im`. The Identity server will therefore have the following information upon successful verification of the email:

- The given email address.
- When submitting the token via HTTP request directly to the Identity server:
  - The IP address of the user.
  - Browser/app information via HTTP header `User-Agent`.
  - Any other information sent by browsers by default.
- The Matrix ID of the user, usually including their username, is also made public without any authentication under the [lookup endpoint](https://matrix.org/docs/spec/identity_service/r0.1.0.html#get-matrix-identity-api-v1-lookup) on https://vector.im.

**Example**: If you were to register with the email `dummy@example.org`, you can go to the following URL and see the JSON response including your Matrix ID:

```
https://vector.im/_matrix/identity/api/v1/lookup?medium=email&address=dummy@example.org
```

Change the `address` query parameter to your email to see the mapping which never expires.

At this point, 3 personal identifiable pieces of information are shared with `vector.im`, a 3rd-party for-profit company directly from a `matrix.org` recommendation without any prompt, explicit information or given informed consent from the user as per GDPR requirements. **Two of them can be queried unrestricted and without any credential.**

Vector.im has [a privacy policy](https://vector.im/privacy) which only applies to jobs and related applications, and does not seem to cover the Identity Server usage specifically, while only giving one lawful basis for processing, directly related to recruitment. It is therefore not known how the data submitted to `vector.im` is processed or shared.

From our experience in the Matrix.org community and various discussion with the Matrix.org people, we came to realise that the Identity Server under vector.im is part of a cluster that at least integrates an Identity Server under `matrix.org` and replicates all data from one onto the other. Matrix.org also has a [privacy policy](https://matrix.org/legal/privacy-notice) making New Vector Ltd the Data controller of the service.

> **NOTE:** You may check it for yourself, replacing `vector.im` in the lookup URL above with `matrix.org`

Riot therefore uses, by default, a for-profit service that has no related privacy policy, sharing Personal Identifiable data with a 3rd-party without informing its users, while relying on the trust of a Matrix.org recommendation.

### Devices

Riot is a End-to-End encryption (*E2EE*) capable client. *E2EE* is based on devices, representing the clients from where users connect to Matrix. The term *Device* is used for other specifics in Matrix: This section specifically covers *E2EE* devices.

*E2EE* is a highly-sought feature that allow to encrypt (hide) the content of the events flowing on the network. *E2EE* does not encrypt metadata in Matrix, only what is referred to as `content` . Content is what you say. Metadata are things like the user who sent it (Matrix ID), the time, its origin (Server ID). Matrix.org regularly talks about it [on their blog](https://matrix.org/blog/) and, while it is not enabled by default, rightfully promotes it as a key feature of Matrix.

For *E2EE* to work as intended and make sense, users are prompted to validate other users' devices when they talk with them in an encrypted room. As people login in and out, devices changes and users need to re-validate which devices they trust where encrypted messages should be sent.

A *E2EE* device has one directly visible attribute: its name. This name can be **optionally** set [during login](https://matrix.org/docs/spec/client_server/r0.5.0#post-matrix-client-r0-login) and [manually later on]() by the user. In Riot, if you are to click on a user avatar, or click on a Matrix ID in a message, the right panel will show the user profile including its devices and their names. There is no restriction on getting the devices list, and a user can see the devices list of any other user on the network [if their respective servers can communicate](https://matrix.org/docs/spec/server_server/r0.1.2#get-matrix-federation-v1-user-devices-userid).

By default Riot sets the device name to contain some or all of these attributes:

- If Web, Desktop or Smartphone.
- The URL where it is hosted and the browser used (if applicable).
- The system name.
- The smartphone model (if applicable).

Given the default name set by Riot on login, and the lack of restriction on fetching the *E2EE* device names list, **users who do not change their device names are exposing those attributes for anyone to query, track and process. In the case of the URL, more information could be gathered, like the default configuration of Riot for sensitive services like the Identity Server of the user.**

**Riot does not inform the user or request consent before using sensitive information to set a device name, even if the device name is optional. It also does not recommend changing the device name on first use or after login. Because *E2EE* is currently disabled by default, users do not get to validate devices and see those names either, which could help them being informed on the matter.**

### Welcome Bot

One of the other default settings is a Matrix ID for the Welcome Bot feature. This feature automatically creates a direct chat with an automated program controlling a Matrix user, allowing a user without prior Matrix experience to ask questions and get useful links. The Matrix ID of the user is `@riot-bot:matrix.org`. Upon inviting the user, a request will be made by the user Homeserver to the `matrix.org` Homeserver, allowing the collection of the following information:

- The Matrix ID of the user, built from their username, and which Homeserver/domain they are using.
- The date and time at which the account was created.
- The IP address/hostname of the server connected to the user, which might allow to identify a user in case of a single-user Homeserver.
- From the Homeserver IP address, their potential GeoIP country/city.

  

### Identity Server

> **NOTE:** Some of the described behaviour is specific to Riot Android and possibly iOS

Identity servers are one of the most misunderstood services in Matrix. Contrary to common belief, Identity servers do not deal with accounts or authentication, but with Identifiers labelled `3PID`, a technical term used to describe things like Email and Phone numbers. We'll use *Email* in a generic way in this section.

While Matrix.org does not recommend self-hosting Identity servers, they deal with several key behaviours and personal identifiers:

- Adding/Removing an *Email* to one's profile for discovery by other users.
- Adding an *Email* to allow password reset of an account. Email is the only self-service way to regain control back after forgetting your password.
- Search for other users to connect with by looking up their *Email*.

Control of the `vector.im` and/or `matrix.org` server allows several Denial of Services in terms of blocking 3PID associations and finding other people.

More worryingly, a central server has control over the associations between Email/phone numbers and Matrix IDs and may create them arbitrary, hide or remove them as there are no proof or signature that the 3PID owner allowed such association. This can be used to blacklist/abuse people by abusive administrators relying on an expectation of trustworthiness, but also it allows to target people of interest like activists, people from minorities, etc.

#### Adding an Email

When attempting to add an *Email* to the Settings, a request is made to the Homeserver to validate and add it. This request is proxied to the Identity server, hiding the IP address and any info in the headers from the Identity server.

The Identity server then sends a validation token either in the form of a browser link, or a code to input. In case of email, a link is provided directly pointing to the Identity server instead of the Homeserver. Upon validation, you go back into Riot and click on "Continue" which triggers the final step of actually linking the Matrix ID and the *Email*.

While Matrix sets publishing the association to the Identity server [off by default](https://matrix.org/docs/spec/client_server/r0.5.0.html#post-matrix-client-r0-account-3pid), Riot explicitly requests it. This makes the association public and queryable **without informing the user or prompting for consent.**

The following information is shared with a 3rd-party:

- The IP address of the user.
- Its Matrix ID.

The following information is made queryable without restriction to anyone:

- Association of an *Email* to a Matrix ID.

#### Removing an Email

Removing an *Email* takes on a different approach: while adding an *Email* requires some kind of validation from the owner, removing it does not. It relies on trusting the user's Homeserver to remove the association in a legitimate manner. The user is never prompted to confirm that such removal is wanted or allowed.

#### Searching for other users

Searching for users is divided into two main use cases:

- A [single, specific search](https://matrix.org/docs/spec/identity_service/r0.2.0.html#get-matrix-identity-api-v1-lookup) available in all Riot versions.
- A [bulk search of contacts](https://matrix.org/docs/spec/identity_service/r0.2.0.html#post-matrix-identity-api-v1-bulk-lookup) to find any match, only available in smartphone versions.

Those searches use unauthenticated Identity server endpoints that Riot directly connects to, allowing the user IP address and its device/Riot version to be visible for each request.

While the single specific search behaviour may or may not be understood by users and system administrators, and that potentially identifiable data is shared with `vector.im`, it is recognised that such requests are only made in response to explicit requests from the user. The various FAQs are unambiguous that Identity servers are used for this purpose.

What is not really known by users, and tends to only be obvious to people implementing the Identity Server spec, is Riot's behaviour regarding bulk search.

Once connected to a Homeserver and on first usage of Riot Android, users will be shown a prompt when clicking on the "People" button, requesting permission to access their contact list. After granting permission, **every email and every phone number in the user's phone book will be sent to the Identity server without any kind of obfuscation or masking**.

The undocumented behaviour is that **any time the user switches out then back in the People view, the full contact list is sent again.**

This bulk behaviour allows the Identity server to:

- Know the IP address, client and system of the user.
- Know the potentially complete social graph of the user.
- Receive personal Identifiers (Email and Phone numbers) sent without obfuscation from users unaware of such sharing.
- Receive requests matching pattern usage for the user, specific to certain devices types (smartphones).

### Sharing, Permalinks

Recent versions of Riot have a "Sharing" icon, made of three dots linked together in the shape of a triangle. Riot also has a "Share message" option. Both open a new dialog with a URL starting with https://matrix.to/ and a QR code.

Technically, "sharing" (permalinks) is built around a website https://matrix.to/ instead of a URI scheme. While the website is stateless, [a cookie](https://support.cloudflare.com/hc/en-us/articles/360024915491) is set on each visit by Cloudfare. This cookie uniquely identifies clients even thought the website is supposed to be privacy-oriented and not track users/requests.

If the link is visited instead of intercepted by the client, the following info is shared:

- IP address of the client/user.
- Browsers headers containing device and system information.
- Usage patterns of the "Sharing feature".
- Value of the Cloudflare cookie `_cfduid`.



### Integration server

> **NOTE**: Some of the described behaviours are specific to the Web and Desktop clients.

Riot comes with a proprietary closed-source service (protocol and implementation) called an Integration Server. The Integration server can be used to add services/bots/bridges to a room, like a Jitsi VoIP conference, enhancing the Riot experience. This service is (was?) meant to be the monetisation feature of Riot, remaining closed-source to this date.

Riot comes with the default configuration of using `scalar.vector.im` as its Integration Server. Integration can be triggered using the 4 small squares icon at the top right of a room, connecting to `scalar.vector.im` and displaying the current configuration of the Room/services already integrated.

To do so, the following handshake is done:

1. Riot requests [an OpenID token from the Homeserver](https://matrix.org/docs/spec/client_server/r0.5.0.html#post-matrix-client-r0-register-email-requesttoken). This token can be exchanged for the Matrix ID of the user at the time of writing.

2. Riot connects to the Integration Server to either register a new session with the OpenID token requested earlier, or to validate an existing session.

3. The Integration Server exchanges the OpenID token [via the federation API](https://matrix.org/docs/spec/server_server/latest.html#get-matrix-federation-v1-openid-userinfo) for the user Matrix ID.

4. Riot then calls the Integration Server with the Room ID to get its Integration status.

No information or explanation is given to the user about their Matrix ID, a potential personal identifier, being shared with a 3rd-party service without a privacy policy. No consent is requested either.

What is less known is that step 2 happens **every time a user switches to another room in the UI.** This means that `vector.im` is receiving the following information without the user's knowledge, some without the user even opening the Integration server window:

- A steady stream of requests directly related to user activity and usage pattern of Riot and Matrix.
- Their Matrix ID and their IP address, Riot directly connecting to `scalar.vector.im`.
- The rooms which the user is part of.

---

In terms of Integration usage of the `scalar.vector.im`, several bridges, bots, widgets and sticker packs are provided via the `matrix.org` Homeserver. It means that by using nearly any of them, `matrix.org` will be involved directly or indirectly into the room. In case of bridges and bots, a copy of the room history alongside members' display names and avatars will be known/copied to the `matrix.org` server, further giving a means to directly access data and conversations.

**The tight coupling of `matrix.org` on those servers is never explicitly explained to the users, nor that past chat history could be downloaded in some cases without them being aware, or that any outage to the `matrix.org` server would also affect those services.**

**Users are also not told that the service is proprietary and closed-source, only allowing alternative implementation by reverse engineering. This does not allow privacy/security reviews of the software stack, while being the element that has direct access to users' data.**



### Push Server

> **NOTE:** This section is specific to Riot Android/iOS

Matrix uses a concept of [Push server](https://matrix.org/docs/spec/push_gateway/r0.1.0.html) to send push notifications to smartphones. The push server is meant to be managed by the application developer. In case of Riot, the push server is configured to `matrix.org`.

Riot provides two privacy level for notifications:

- Normal (event metadata only)
- Reduced Privacy (full event data)

While Riot gives the high level differences between the two, it does not mention `matrix.org` involvement or which metadata is shared and visible. Only the Google services are mentioned.

The Push server will have access to the following info in each notification:

- The app ID and The push key, which could uniquely identify a user
- The room ID.
- The event ID.

And overall:

- Pattern usage/activity of the user and the users they connect to.
- ID of rooms joined by the user, potentially identifying direct message rooms.

Control of the `matrix.org` push server allows to perform Denial of Service, blocking notifications that people tend to rely on to further participate in conversations when a reply is sent. See below for a real-word impact during the security breach.

#### Push Rules: Keywords, Highlights and Notifications

In Riot settings, under *Notifications*, you will find "Messages containing keywords" with a link allowing to set arbitrary words/patterns that will specifically trigger highlights, in terms sending Push Notifications to your smartphone with the default configuration of enabled notifications.

These keywords are those personal triggers a user can configure, topics, words that are personal/specific to them on top of what the protocol itself provides. These keywords might contain private topics, name, interests, political choices, etc.

Push notifications contain the Room ID and Event ID which make up the global unique identifier of an event. If the entity running the push server has also access to the room and the event, they are able to know the content of the notification that will be seen by the user, even if the notification itself does not contain it.

**By keeping track of which events triggered a notification for a user, in the case where the entity has access to the event, they have the possibility to extrapolate the keywords given enough notifications. The more rooms a server is joined to, the more effective and accurate this would be.** As one of the biggest servers known in the public federation, `matrix.org` would be a prime location for such data processing.

### VoIP

> **NOTE:** *Call* will be used to refer to audio or video in 1:1 situation and does not cover widgets.

[VoIP calls in Matrix](https://matrix.org/docs/spec/client_server/r0.5.0#voice-over-ip) are based on [WebRTC](https://en.wikipedia.org/wiki/WebRTC) which was designed by the main actors of the web (Mozilla, Google, etc.) to give websites/web-apps a mean to do VoIP in a native way and directly in the browser without a need for plugins/extensions.

If you are already familiar with VoIP, STUN and TURN, [you can skip to the Matrix bit](#in-matrix).

#### Background

VoIP in general, WebRTC included, make call happen using two principles:

- Signalling channel: This is how the devices/clients will send messages about the call, like "I am placing a call", "These are my media channels and my codecs" or "I'm hanging up". This is also where SDP data is exchanged. We will explain SDP in a second.
- Media channel(s): This is where you'll find the actual voice/video data that you will hear/see.

VoIP is very sensitive to slow and unstable network connections or delays for media, but at the same time not **all** data is needed for rendering. Because VoIP is real-time, a missing/out of order sound/video frame cannot be rendered as the next ones have been already. It is then dropped. For sound and video to work and be useful to humans, there cannot be a lot of delay either due to their real-time nature.

The best way to accomplish this is to have both clients connect to each other. This ensures the shortest and fastest path (no server in the middle to use and depend on) is used to transmit data. To do so they need to find out their IP addresses and have an incoming port open. Because of NAT, firewall restrictions, network access on smartphone on Data network or any number of reasons, clients are usually not able to ensure the port is open and directed to their devices, or even know their public IP addresses in the first place. Regardless, clients following the protocol will attempt to collect as much information as possible to be sent to the other client.

Once a client has collected all the data it can (system IP addresses, available codecs, etc.), it will place it into a well-understood message format. That format is called [SDP](https://en.wikipedia.org/wiki/Session_Description_Protocol) and is used in various other protocols, like [SIP](https://en.wikipedia.org/wiki/Session_Initiation_Protocol). The other client will do the same, usually in the "I answer the call" message. Both clients will then try to connect to each other. If they can connect, great! Your call is now working. If not, clients usually fail the call after X seconds - *could not connect media* is what you would see in Riot.

In the real world, relying on values only visible by the clients tend to very low successful connected calls. To fix that, two other protocols/tool-sets were designed: [STUN](https://en.wikipedia.org/wiki/STUN) and [TURN](https://en.wikipedia.org/wiki/TURN).

STUN is a way for clients to get more info about themselves and their network connection, like their public IP address and usable port(s) for media channels. It is based on a simple, fast, light request/response mechanism. It is used just before sending the "I am placing a call" signal. The client can then include all that info in their SDP message of placing a call. While this help clients gather better info, it might have no effect.

Finally, TURN allows to use a relay, a proxy of sorts, that will act like a middle man, receive data from one client and sending it back to the other. Being a relay, clients do not need to have ports open; they only need to connect to the server, which grants them a duplex channel to send and receive. TURN is the big gun of VoIP and allows a very high success rate in placing calls.

#### In Matrix

STUN and TURN are quite sensitive privacy-wise: As a server in both, you get to know that a client is trying to attempt a call and their IP address. As TURN, you get to know which clients are connected to each other, and you receive the encrypted data of the call.

All Riot versions are currently hardcoded with [a default STUN server pointing to Google](https://web.archive.org/web/20190621153620/https://github.com/vector-im/riot-meta/issues/150):

- [Web/Desktop](https://github.com/matrix-org/matrix-js-sdk/blob/a73dabcb6735f71ece4455cf56926c5692ad4146/src/webrtc/call.js#L96)
- [Android](https://github.com/matrix-org/matrix-android-sdk/blob/aa7b11958ea78520d8bc5fb32ce7bddabdbdb13d/matrix-sdk/src/main/java/org/matrix/androidsdk/call/MXWebRtcCall.java#L602)
- [iOS](https://github.com/matrix-org/matrix-ios-sdk/blob/3d3a0984b1c3d4fa0042f183a68c8e3f8ab782a6/MatrixSDK/VoIP/MXCallManager.m#L34)

Synapse is configured by default with [an empty set of TURN servers](https://github.com/matrix-org/synapse/blob/v1.0.0/synapse/config/voip.py#L21). Knowing that TURN is tried before STUN, **if synapse has no TURN servers configured, STUN will be attempted for each call.**

The synapse's documentation [has a specific entry for it](https://github.com/matrix-org/synapse/blob/v1.0.0/INSTALL.md#setting-up-a-turn-server), but does not mention the privacy aspect if no TURN server is configured and the client has a default STUN server. Our experience with the ecosystem is that people installing synapse will postpone TURN to later, or just ignore it as they just want to try synapse and just send text messages.

When they eventually do try VoIP and that it works, they will ignore or have forgotten the TURN setup. If placing voice/video calls do not work, some give up installing/using a TURN server. Our experience with users is that the [dedicated document](https://github.com/matrix-org/synapse/blob/v1.0.0/docs/turn-howto.rst) for it is quite obscure and hard to understand, asking the user to consider several things most sysadmins aren't familiar with at all:

- Building `coturn` from source
- To consider security settings like a black and white list of peer IPs

**It also doesn't give any specific about several crucial points like network configuration or TURN TLS.**

**Network configuration:** the TURN server **MUST** know about its public IP address and ports MUST be opened and relayed to the TURN server. The guide does not document the public IP address configuration options nor which ports to open and assume all those are easy to figure out.

**TURN TLS:** In WebRTC, media streams are encrypted by default, protecting privacy. The TURN client/server exchange is not by default, requiring extra configuration in `coturn` and specific TURN URIs (starting with `turns:` ) to be set first in the `turn_uris` array. **The synapse's documentation does not recommend the use TURN TLS, only mentioning it as optional in step 7 of the `coturn` setup. The documentation also does not give example of TURN TLS URIs, leaving it up to the reader to figure out.**

This is based on regular questions/feedback received from the community over several years, which also triggered one of the authors of this document to write [a "straight to the point" document](https://gist.github.com/maxidorius/2b0acc2e707ae9a2d6d0267026a1024f) to help users who are lost with the various options usually alien to them.

Whichever STUN server is used, they would have access to:

- The IP address of the client and derived Geo location
- Call creation patterns

**When all put together, it is easy for system administrators to skip TURN which will trigger STUN requests to a 3rd-party STUN server for each call placed by a user.**

**If the STUN server and the Push server are the same, VoIP metadata could be directly correlated using all the other data received. Because VoIP [must only used in rooms with two participants](https://matrix.org/docs/spec/client_server/r0.5.0#voice-over-ip), correlating such data with received notification data as a Push server would directly identify Direct Messages rooms.**



### Basic network calls

#### Headers

Throughout this document, we refer to concepts such as *HTTP headers*, *Browser information* and *system version*. These terms are usually obscure to most people but are one of the most common privacy leaks. They do not really provide usable information by themselves. Instead, **they are key elements to process when correlating various sources if the intent of a malicious 3rd party is to identify a person or deanonymize submitted analytics.**

Some of those headers can be redacted/removed in the HTTP libraries used in software, but cannot in web-apps.
Given the complex and technical nature of HTTP headers and the ability of the various software to control them, this document will limit the scope of this section to their value.

**Except for the Original header, none of these headers matter for the Matrix protocol. Some might allow to improve the user experience (e.g. Accept-Language).**

##### Origin

> **NOTE:** This header is specific to Riot Web and Riot Desktop

[`Origin`](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields#origin-request-header) is a standard mandatory header used in [CORS](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) to request the various `Access-Control-*` headers.

`Origin` is set to the protocol, hostname/IP address and optional port where the content is hosted. In case of Riot Web, it would be something like `https://riot.example.org`. In case of Riot Desktop, it is `vector://vector`.

**This mandatory header will therefore inform the server about the location where the client is currently hosted, or that it is Desktop version. This can also contain non-public hostnames, IP addresses, etc.**

Per example, this information could be used to:

- Identify weaker setup that use HTTP.
- Trace back a request to a Homeserver domain, inferred from where Riot-Web is hosted.
- Identify the device type of a user: Web or Desktop.
- Perform discovery of a domain Matrix setup, by reading the host `config.json`  if it is available at the root of the domain: Homeserver URL, Identity Server URL, Integration server, etc.

##### Referer

> **NOTE:** This header is specific to Riot Web

[`Referer`](https://en.wikipedia.org/wiki/HTTP_referer) is an optional HTTP header. It is very similar to `Origin` except it contains the URL of the page from where the request was made. This header [is typically used when tracking users](https://en.wikipedia.org/wiki/HTTP_referer#Details).

**This header gives a more precise value than `Origin`, and would give more accurate results in tracing back to the original client and related configuration.**

##### Accept-Language

[`Accept-Language`](https://en.wikipedia.org/wiki/Content_negotiation#Server-driven) is a header that informs the server about the preferred language that should be used for the content: English, French, etc. By default, this header is set to the installation language/system language.

**This header gives [demographic data](https://en.wikipedia.org/wiki/Demography).**

##### User-Agent

> **NOTE:** Examples are taken from the GitHub issue [vector-im/riot-meta#295](https://github.com/vector-im/riot-meta/issues/295).

[`User-Agent`](https://en.wikipedia.org/wiki/User_agent#Use_in_HTTP) is set to the software and system version/type, allowing a server to give the most appropriate content for that software. One typical use is to detect graphical browser from command line browser, and output the best format for a specific content. Per example, the graphical browser would receive an image while a command line browser its alternate text.

These are examples of the value in each Riot version:

---

Web:

```
Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:67.0) Gecko/20100101 Firefox/67.0
```

Desktop:

```
Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Riot/1.2.1 Chrome/69.0.3497.128 Electron/4.2.2 Safari/537.36
```

Android - Google Play:

```
Riot.im/0.9.1 (Linux; U; Android 5.1; Phone Model Build/precise build number incl. security patch level; Flavour GooglePlay; MatrixAndroidSDK 0.9.23)
```

Android - F-Droid:

```
Riot.im/0.8.28a (Linux; U; Android 9; Phone Model and ROM Build/precise build number incl. security patch level; Flavour FDroid; MatrixAndroidSDK 0.9.19)
```

iOS:

```
Riot/0.8.6 (iPhone; iOS 12.2; Scale/3.00)
```

---

The value of this header tends to be very specific and more easily tied to users being of high value if correlation is wanted.

**By default, some HTTP servers log this value, including the client IP address, allowing a direct match. These can be directly accessible in case of a security breach.**

## Synapse, the reference server implementation

### Basic network calls

When interacting with users from other servers or rooms containing them, synapse uses the S2S protocol, a specific set of endpoints that usually require [authentication/authorization](https://matrix.org/docs/spec/server_server/r0.1.2.html#authentication).

Up until recent synapse versions, self-signed certificates were accepted as signing keys and certificate fingerprints were checked via a [validation approach](https://matrix.org/docs/spec/server_server/r0.1.2.html#retrieving-server-keys) borrowed from the perspective project. Synapse would check that the keys received are the same if requested by another server, called a Notary server.

Prior to v0.99.2, synapse contained a `perspective` key in its configuration which [was uncommented (enabled) by default pointing to `matrix.org`](https://github.com/matrix-org/synapse/blame/282c97327f150a37d53f90ab6207bc1f98e70da3/synapse/config/key.py#L124). The server would be queried on a regular basis to fetch the keys of every other server synapse was talking to.

From v0.99.3, the configuration was [commented out](https://github.com/matrix-org/synapse/commit/fd463b4f5db3fb164505bc2b2300c6a5f73c73e4#diff-8b49bb218a1d7b96778287934cc311eaR797) and instead [hardcoded into the source code](https://github.com/matrix-org/synapse/blame/v0.99.5.2/synapse/config/key.py#L55), making it so that **even if the configuration was manually commented out, or removed, synapse would still talk to `matrix.org` by default and reference all other servers that the Homeserver is in contact with.**

One of the big change for synapse v1.0 and Matrix was the switch from a self-signed, perspective approach to regular CA TLS certificates via [MSC 1711](https://github.com/matrix-org/matrix-doc/blob/master/proposals/1711-x509-for-federation.md). This proposal [recognised](https://github.com/matrix-org/matrix-doc/pull/1711/files#diff-14fe96e0952d0411db3e9ecbbddce789R40) the centralisation problem and the attack surface of how synapse used `matrix.org` as a single notary server. The proposal was meant to move away from the perspective model, validate TLS certificates directly and not require notary servers anymore (or so understood), [decreasing centralization](https://github.com/matrix-org/matrix-doc/pull/1711/files#diff-14fe96e0952d0411db3e9ecbbddce789R222).

As of synapse v1.0.0, we see that the perspective key has been switched for a new key called `trusted_key_servers` which is commented out in the default generated configuration. But `matrix.org` is still hard-coded [in the source code](https://github.com/matrix-org/synapse/blob/v1.0.0/synapse/config/key.py#L85).

**We have confirmed that synapse v1.0.0 still connects to `matrix.org` to fetch keys, even if no longer necessary, and does so for every single server your Homeserver talks to.** We also confirmed that despite the key lookup endpoint [not requiring authentication](https://matrix.org/docs/spec/server_server/r0.1.2.html#querying-keys-through-another-server), synapse does send cryptographically signed requests to `matrix.org` which ensures the requester can be identified.

`matrix.org` has therefore the ability to:

- Know which servers exchange data with each other.
- Build a social server graph of many, if not nearly all, federation servers.
- Build usage patterns from the regular re-validation requests.
- Block servers from talking to each other by returning invalid key data.
- Still be a Man-in-the-middle source of attack for anyone who would have access to the `matrix.org` servers.

Synapse developers on Matrix.org do not give the details and impact information to system administrators about how potentially private information (in case of single-user Homeserver for example) is shared with a 3rd party without consent. They also take on a non-intuitive approach in regards of configuration, relying on hard-coded configuration in case it was commented out / removed from the configuration file.

**We have confirmed that removing the hard-coded values from the source code and all possible configuration options does not prevent synapse from exchanging data with other servers in a secure manner to the best of our knowledge.** We have been running such a setup on some of our Homeservers for several months without any issue.

### Media repository

Users can exchange files, images, video and audio using the Media Repository feature. Upon successful upload, a URI is returned that can be embedded in messages or used to build a classic URL to access the media. The media repository is primarily used to store users' avatar for their profile, but can and will contain sensitive and highly private data, like pictures of one's family, PDFs of private scanned papers, proprietary and closed source project files/documents, etc.

In terms of privacy and access security, it has two major issues:

- It is not possible to delete/remove a file from the repository using the [regular Client API](https://matrix.org/docs/spec/client_server/r0.5.0.html#id111).
- Files are directly accessible via a [public, unauthenticated endpoint/URL](https://matrix.org/docs/spec/client_server/r0.5.0.html#get-matrix-media-r0-download-servername-mediaid).

Files are referenced using a dedicated link which never expires. A user-friendly version is also accessible via Riot and can be copied to the clipboard, and then used in any other application. While easy on the user, it gives a permanent link to a file, which can also be shared/seen by unintended entities. Given the lack of delete/remove action and lack of access control, it puts an unexpected and usually misunderstood requirement to be extremely careful if pointing to private/sensitive data.

Riot does not inform the user of such lack of basic access control and privacy when uploading files to rooms. From our experience, users believe that access to files are controlled in the same way that access to those rooms is, keeping files private and inaccessible to anyone outside of a non-public room.

It is interesting to note that the undocumented server version of this API uses authentication in synapse while the client version does not.

### User profiles

User profiles hold the information about the display name and avatar of a user. Those are set in Riot in the Settings view. Access is unrestricted and unauthenticated using [a specific endpoint](https://matrix.org/docs/spec/client_server/r0.5.0.html#get-matrix-client-r0-profile-userid),

This means that the following information, if configured, is directly available publicly, in an unrestricted manner without the user informed and explicit consent:

- Their display name, which can include their real first/last/middle name.
- Their avatar, which may be a picture of themselves.



### Identity Servers

By default, synapse only allows two Identity servers to be used for the various 3PID interactions:

- `matrix.org`
- `vector.im`

The initial idea and concepts behind Identity servers was to be [independent of Home servers](https://matrix.org/docs/spec/identity_service/r0.2.0#general-principles) and only hold association data. Homeservers would [hold Administrative data](https://matrix.org/docs/spec/client_server/r0.5.0#adding-account-administrative-contact-information) for their use to interact with the user directly.

In practice, execution of this idea has lead to only trusting central servers and disallowing clients and users from picking Identity servers they trust: they must all be manually set in synapse default configuration. Such a change may or may not be possible, depending on the level of control the user has over the Homeserver configuration, or ability to reach/communicate with the system administrator.

Due to the difficulty of adding new ones, system administrators tend to either leave the default configuration or add new ones **without ever removing the default ones.**



## Usage of Matrix.org and Vector.im

All services (hosted under `matrix.org`, `vector.im` and `scalar.vector.im`) are going through [Cloudflare](https://en.wikipedia.org/wiki/Cloudflare), a US-based CDN. TLS termination is done at the Cloudflare level, allowing them to decrypt and see in clear all the traffic coming in and out.

It is important to put this information in perspective of all the data/metadata shared given all the points above, allowing a foreign 3rd-party to directly have access to plain text traffic, private identifier, data and metadata without ever being mentioned anywhere.

## Matrix.org security breach on Apr 11 2019

### Timeline and events

For those unaware, Matrix.org was breached by an attacker for several days which triggered service downtime and a full rebuild of the Matrix.org infrastructure. Some people were amazed to see how this did not impact their Homeserver and they could continue to talk to others without interruptions and their data was safe on their own servers. That being true for some people, the reality was not so straight-forward.

On their [initial communication](https://matrix.org/blog/2019/04/11/we-have-discovered-and-addressed-a-security-breach-updated-2019-04-12), they say:

> ### The security breach is not a Matrix issue.
>
> The hacker exploited a vulnerability in our production infrastructure (specifically a slightly outdated version of Jenkins). **Homeservers other than matrix.org are unaffected.**

While the security breach was not in of the Matrix protocol, other Homeservers **were** affected by it. As per our analysis above, we know that people hosting a typical stack would have the following services not available to them:

- No key signature verification via notary, without visible impact to users.
- [Identity services based on `vector.im`](https://git.feneas.org/feneas/association/issues/51)
- No push service, with direct impact to users (we were affected) for 24h+ reported to us.
- No bridges/bots/widgets hosted on the `matrix.org` Homeserver.

The announcement does not mention anything about collected data from Homeservers as part of the natural behaviour of the network, even though, "*the attacker did have access to the production database*".

In terms of personal identifiers like emails and phone numbers, you can read:

> **What has not been affected?**
>
> Identity server data does not appear to have been compromised

While technically correct, Identity data as most commonly understood is also present in the Homeserver database which **was** accessed by the attacker. They eventually posted a screenshot of various commands ran on a DB extracts: how much the attacker actually accessed is unclear given the Matrix.org communication.

Finally, on the 12th of April, the attacker used collected credentials (before being locked out) to take control of Cloudflare and pointing matrix.org to another website. The communication is not clear if the defacement affected the `/_matrix` API endpoints and its data coming from others servers.

### Privacy and Security Impact

Taking into account all the data and metadata flowing to `matrix.org`, the security breach is a concerning event as an attacker had means to collect and process those data mostly found in system/application logs, database and reverse proxy logs.

The attacker could also have directly disrupted the federation in a significant manner via Denial of Service and cryptographic  poisoning for the Notary and Push services. The attacker had access to hypothetical private room messages in which Integration services are used like bots or bridges.

## Closing words

In a world where Privacy and Security are extremely hard to come by, protocols that give the means for decentralised, secure and private communications are highly sought, sometimes to the point where users will turn a blind eye to *minor* issues and *inconveniences* that might be solved down the line. Several of these shortcomings, leaks and issues have been brought up to the Matrix.org team and have witnessed first hand disregard for such reports, and purposeful de-prioritisation of issues while working on [mxisd](https://github.com/kamax-matrix/mxisd), our Federated Identity server focusing on privacy.

Privacy destruction is never about a single HTTP call, or a specific piece of data being leaked. It's always about putting together data from various sources, the amount and regularity of receiving such data. Privacy protection is a mindset, where one understand the cumulative effect of small, isolated pieces of data when put together.

By releasing v1.0, Matrix.org makes a promise of a secure and self-contained protocol while promoting privacy. But at the same time, has a near-monopol in the whole ecosystem in terms of client and server use: Riot and synapse, also labelled "reference implementations". We believe that reference implementations should reflect the core values of the protocol. They currently fail to do so and instead produce a near-centralised network which fails to protect people's privacy.

Security breaches in Matrix.org are an important reminder that we also are at the mercy of 3rd party entities with which we share our personal information unknowingly. They might leak private data unintentionally/unknowingly but still with a strong impact on the user, like it has happened many times in the past with security breaches across the Internet.

While users on the matrix.org Homeserver have to explicitly agree to the Terms of Use and the Privacy policy, no agreement is ever sought from users on self-hosted servers that also use `matrix.org` and `vector.im`. How is their data handled? Are they processed in some way? Which method of lawful processing under GDPR allows for this constant sharing of (meta)data? We hope such questions will be answered to ensure users' privacy is handled appropriately.

---

We do not claim we have made a full investigation or review, but we hope these notes will be useful for you to better understand:

- How Matrix works.
- The entities behind Matrix.org and how they relate to each other.
- What happens when you use Riot and synapse with only changing your Homeserver URL.
- How your private data and metadata are sent to those entities most likely without your knowledge or consent.

That you'll be able to make an educated and informed decision when choosing which Client, Homeserver, Identity server and Integration server you wish to run in the future. That you'll know which questions to ask when looking for the next best thing.

To discuss further, come to our Matrix room: `#kamax-matrix:kamax.io`.



## Editorial notes

**CC Attribution:** [The Grid protocol](https://gitlab.com/thegridprotocol/home/blob/master/README.md#the-grid) community  
**Authors:** The [Kamax.io](https://www.kamax.io/) Team - [Maximus](https://github.com/maxidorius) and [mkatee](https://github.com/mkatee)  
**Reviewers:** [Slavi](https://github.com/spantaleev), [mujx](https://github.com/mujx), Juuso "Linda" Lapinlampi

Special thanks to the following people who provided significant feedback pre and post-publication:

- [Tulir Asokan](https://maunium.net/)
- [Jason Volk](https://github.com/jevolk)

---

**This work is licensed under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).** See [Editorial notes](#editorial-notes) for Attribution details.

---
