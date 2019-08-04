# Annex E - GDPR Data Extract - Room events

This annex is part of the [Notes on privacy and data collection of Matrix.org, Part 2](../README.md) research document.

**If you would like to link to this document, you may only do so using the header link of the Annexes section title found on the [main document](../README.md) to ensure vital context is seen by any reader.**

This document is provided for educational purposes only, in the public interest of individuals who want to understand and see what data is given in response to a GDPR Request, in the scope of the Matrix protocol.  
Use of this document, for any other purpose, is **NOT** allowed without the express approval of the authors of the research document.

---

This document contains a set of tables, extracts and correlated data to validate the claims done on the main research document.

All Room and Event IDs are partial and the last characters have been redacted. To match in your database, use the following approach, replacing `<VALUE>:

- For Room IDs: `WHERE room_id LIKE '<VALUE>%'`
- For Event IDs: `WHERE event_id LIKE '<VALUE>%'`

Bridging is a major feature of Matrix and GDPR data requests directly involves considering the lawful basis and privacy expectations of individuals on networks that are not in scope of Matrix.org privacy notice as they do not use their services, and may not even be aware of the bridging in the first place. The biggest bridge from Matrix.org is the one linked to Freenode IRC, and is present in most of the biggest rooms run by of the Matrix.org server users. We will also include numbers for the users of Freenode IRC as an example of the impact of a leak.

---

## Overview

Layout of the ZIP file:

```
export-max.zip
\--- rooms
     +--- !AA-RoomID:example.org
     |    +--- events
     |    \--- state
     |         +--- $EventIdOne:example.org
     |         +--- $EventIdTwo:example.org
     |         \--- $EventIdThree:example.org
     |
     |--- !AB-RoomID:example.org
     |     \--- events
     |
     +--- ...
     |
     \--- ...
```

---

### Size of the extract

Total bytes size:

```bash
$ du -bd 1 | grep rooms
8687851718	./rooms
```
### Total number of lines

One line is equal to an event, except for a few exceptions that were malformed and split over several lines.

```bash
$ cd rooms
$ find . -type f -print0 | wc -l --files0-from - | grep total
7660193 total
```

### File format

The intended format was one event per line, but did not take into account events containing new line characters. Such characters were not escaped and prevented a simple load directly into a database.  
Other special characters were not escaped also, like the null character (`\0`) leading to invalid JSON depending on the database/system used to load the extract.
Files also contained multiple instances of the same event, usually spread across the `events` and `state` files, but sometimes in a single file also.

Those issues required to write a custom application (or script) to:

- Attempt re-composing multi-lines events using the [heuristic](https://en.wikipedia.org/wiki/Heuristic) "[Trial and Error](https://en.wikipedia.org/wiki/Trial_and_error)" method.
- Escape invalid characters.
- De-duplicate events to be accurately used.

Afterwards, events were injected into a table called `gdpr` in the same database as a synapse instance.

## Sanity checks

We compared the numbers from two types of servers:

- A corporate server, labelled "*Corp*", that:
  - Only has a handful of individuals using it on a daily basis
  - Is for users who have advanced technical knowledge about IT and Matrix
  - Is joined to all big rooms of the federation (Matrix HQ, Riot, etc.)
- A Friends & Family server, labelled "*F&F*", that:
  - Has about 20 individuals using it on a daily or weekly basis
- Is for users without any significant IT knowledge that use it as an alternative to Facebook Messenger, WhatsApp, etc.
  - Is not joined to the open federation, and only connects with two other servers of similar size and purpose.

Over the last 30 months, we see the following numbers:

|          | People | Event Count | % of Extract | Event Size (bytes) | % of Extract |
| -------- | ------ | ----------- | ------------ | ------------------ | ------------ |
| **Corp** | `2`    | `5102055`   | `66.604 %`   | `11811160064`      | `135.95 %`   |
| **F&F**  | `17`   | `158767`    | `2.07 %`     | `191889408`        | `2.21 %`     |

------

Number of events generated per user on each using query (will also list bots that are counted as users, but ignore appservice users). This includes all events, not just IM messages.

```sql
SELECT u.name, COUNT(e.*) AS total
FROM users u
  LEFT JOIN events e ON e.sender = u.name
WHERE appservice_id IS NULL
GROUP BY u.name
ORDER BY total DESC;
```

*Corp*, only individuals:

```sql
     name    | total  
-------------+--------
 @[REDACTED] | 180648
 @[REDACTED] |  18538

```

*F&F* Top 5, only individuals:

```sql
     name    | total 
-------------+-------
 @[REDACTED] | 24912
 @[REDACTED] | 21547
 @[REDACTED] | 14766
 @[REDACTED] |  9469
 @[REDACTED] |  6268
```

And as a sanity check, over the same period, count of events visible from the Corp server, sent by one member of the `matrix.org` core team: `41430`

Highest numbers are all in the same order of magnitude and in line with the known activity of those users, if compared to each other.

## Tables

The data was loaded into the same database as the related Homeserver's database, to allow matching and comparing with local data. All provided events were de-duplicated using their hash, timestamp, depth, sender, and room ID. Event ID is not present on all events due to recent room versions and cannot be used.

Several other tables were created to hold various computed states, like room membership of the individual doing the GDPR request, which are used in other queries.

### Main table

All fields, except for `hash` and `personal`, are taken from the raw event data. `raw` represent the JSON provided in the GDPR extract for that event. `hash` is the Hexa-encoded SHA-512 value of the raw JSON.

```sql
synapse=# \d gdpr;
                Table "public.gdpr"
  Column  |            Type             | Modifiers 
----------+-----------------------------+-----------
 room_id  | text                        | 
 event_id | text                        | 
 sender   | text                        | 
 origin   | text                        | 
 type     | text                        | 
 raw      | jsonb                       | 
 hash     | text                        | 
 depth    | bigint                      | 
 personal | boolean                     | 
Indexes:
    "evuniq" UNIQUE CONSTRAINT, btree (hash, room_id, sender, type)
    "gdpr_depth_idx" btree (depth)
    "gdpr_evid_idx" btree (event_id)
    "gdpr_personal_idx" btree (personal)
    "gdpr_roomid_idx" btree (room_id)
    "gdpr_sender_idx" btree (sender)
    "gdpr_type_idx" btree (type)
```

The `personal` column was populated using the following query, matching all events where the Matrix ID or Email of the individual are found in the raw JSON, and all other rows set to `false`:

```sql
UPDATE gdpr
SET personal = true
WHERE
  raw::text LIKE '%[Matrix ID]%'
  OR raw::text LIKE '%[Email address]%';

UPDATE gdpr
SET personal = false
WHERE personal IS null;
```

### Rooms

A dedicated table is created to hold the list of Room IDs from the extract, and will be used in various queries.

```sql
CREATE TABLE gdpr_rooms AS
SELECT DISTINCT room_id
FROM gdpr
ORDER BY room_id ASC;
```

### Rooms membership

To be able to know if an event was accessible to the individual, the membership state needs to be computed. Not all states are relevant for this research: only the most recent is and specifically the ones that would not allow events to be seen by the individual.

This computation at the database level is a 3-steps process:

- Prepare membership events of the individual to be mapped to GDPR extract data, using the individual's Homeserver database, which is ultimately responsible for access control.
- Find the least and most recent membership event that can also be found in the GDPR request, to understand what membership state looked like from the remote server Point of View.
- Compute the membership state for each room involved in the GDPR extract and store it in dedicated tables.

Preparation of the membership events:

```sql
CREATE TABLE gdpr_room_memberships AS
SELECT e.depth,e.origin_server_ts,rm.*
FROM room_memberships rm
  JOIN events e ON e.room_id = rm.room_id AND e.event_id = rm.event_id
  JOIN gdpr_rooms gr ON gr.room_id = rm.room_id
WHERE user_id = '[Matrix ID]';
```

Find the least recent join membership events, for the very first join to a room:

```sql
CREATE TABLE gdpr_room_membership_earliest_selector AS
SELECT DISTINCT ON (room_id) room_id,depth,origin_server_ts
FROM gdpr_room_memberships
WHERE membership = 'join'
ORDER BY room_id, depth ASC, origin_server_ts ASC;
```

We specifically exclude `invite` as `join` membership is the one that would trigger other servers sending events to another, and virtually no room use the "since they were invited" history visibility setting.

Compute and store earliest membership for all rooms in the GDPR extract:

```sql
CREATE TABLE gdpr_room_membership_earliest AS
SELECT m.*
FROM gdpr_room_memberships m, gdpr_room_membership_earliest_selector s
WHERE
  s.room_id = m.room_id
  AND s.depth = m.depth
  AND s.origin_server_ts = m.origin_server_ts
ORDER BY m.room_id ASC;
```

Find the most recent membership events:

```sql
CREATE TABLE gdpr_room_membership_latest_selector AS
SELECT DISTINCT ON (room_id) room_id,depth,origin_server_ts
FROM gdpr_room_memberships
ORDER BY room_id, depth DESC, origin_server_ts DESC;
```

Compute and store latest membership for all rooms in the GDPR extract:

```sql
CREATE TABLE gdpr_room_membership_latest AS
SELECT m.*
FROM gdpr_room_memberships m, gdpr_room_membership_latest_selector s
WHERE
  s.room_id = m.room_id
  AND s.depth = m.depth
  AND s.origin_server_ts = m.origin_server_ts
ORDER BY m.room_id ASC;
```

### Leaked events

#### Soft leak

Soft leaked events are those that are not directly related to the individual: they do not contain any identifier of the individual.

Those events are those flagged with `personal = false`.

#### Medium leak

Medium leaked events are those that would not be seen by the user via their regular usage of their Matrix account and rooms. So anything before the individual first joined the room or after leaving/being banned.

Technical definition: all events that have either happened:

- *before* the first membership state change that is `join`.
- *after* the last membership state change that is not `join`.

"*before*" is defined as DAG-based (`depth` is equal or lesser to the join event) and time-based (`origin_server_ts` is lesser or equal to the join event).  
"*after*" is defined as DAG-based (`depth` is equal or greater to the leave event) and time-based (`origin_server_ts` is equal or greater to the leave event).

This simplistic approach does not take into account some events that might have occurred at the same time as the membership change, not being aware of it yet, that would change their visibility if processed. That visibility could go either way (allow or deny) depending on the membership change itself. We also believe it could not have impacted Matrix.org extract, as they modified their Homeserver implementation to use the actual computed state and would have given them the actual value, preventing any event in this grey area to be included.

Given our experience with Matrix servers, we believe that this amount is statistically negligible (< 1%) and does not impact the outcome of this research.

We create a dedicated table to store those events:

```sql
CREATE TABLE gdpr_leak_medium AS
SELECT gm.depth AS state_depth, gm.origin_server_ts AS state_ts, g.*
FROM gdpr_room_membership_earliest gm
  JOIN gdpr g ON g.room_id = gm.room_id
WHERE
      g.depth <= gm.depth
  AND (g.raw->>'origin_server_ts')::bigint <= gm.origin_server_ts
  AND g.event_id != gm.event_id
UNION ALL
SELECT gm.depth AS state_depth, gm.origin_server_ts AS state_ts, g.*
FROM gdpr_room_membership_latest gm
  JOIN gdpr g ON g.room_id = gm.room_id
WHERE
      gm.membership != 'join'
  AND g.depth >= gm.depth
  AND (g.raw->>'origin_server_ts')::bigint >= gm.origin_server_ts
  AND g.event_id != gm.event_id;
```

#### Hard leak

Hard leaked events are those that simply cannot be seen by the individual, because they are banned from a room. This restriction would apply on all servers, including their own.

Formally, there are all events that have happened *after* the last ban membership event.

```sql
CREATE TABLE gdpr_leak_hard AS
SELECT gm.depth AS state_depth, gm.origin_server_ts AS state_ts, g.*
FROM gdpr_room_membership_latest gm
  JOIN gdpr g ON g.room_id = gm.room_id
WHERE
      gm.membership = 'ban'
  AND g.depth >= gm.depth
  AND (g.raw->>'origin_server_ts')::bigint >= gm.origin_server_ts
  AND g.event_id != gm.event_id;
```

## Extract analysis

This section contains a set of queries to better understand the extract given to us, and its scope/size.

### Total number of distinct events

```sql
SELECT COUNT(*) FROM gdpr;

  count  
---------
 3473062
(1 row)
```

### Total number of "orphan" events

Some events are part of rooms where we do not have enough membership data given received state events.  
We label those events as "orphans".

```sql
SELECT COUNT(*) FROM gdpr WHERE room_id NOT IN (
    SELECT DISTINCT room_id FROM gdpr_room_membership_earliest
    UNION
    SELECT DISTINCT room_id FROM gdpr_room_membership_latest
);

 count 
-------
    55
(1 row)
```

Given their small numbers, we will treat those as not significant for this research.

### Earliest event

We find the first room that was created in the extract, excluding malformed event that do not have a valid timestamp.

```sql
SELECT MIN((raw->>'origin_server_ts')::bigint)
FROM gdpr
WHERE type = 'm.room.create' AND (raw->>'origin_server_ts')::bigint > 0;

      min      
---------------
 1409924389016
(1 row)
```

UTC time and date for the timestamp: Fri Sep 05 2014 13:39:49

### Latest event

We find the latest generated event taking the timestamp at the time of writing this sentence to exclude fake timestamps.

```sql
SELECT MAX((raw->>'origin_server_ts')::bigint)
FROM gdpr
WHERE type = 'm.room.create' AND (raw->>'origin_server_ts')::bigint > 0;

      max      
---------------
 1563359715478
(1 row)
```

UTC time and date for the timestamp: Wed Jul 17 2019 10:35:15

This is in line with the timestamp find on the extract files themselves.

### Events without Matrix domain in their IDs

This tracks the number of events in newer room versions that do not have a Matrix domain in them, not giving the possibility to an individual to "track down" their data across the federation.

```sql
SELECT COUNT(*) FROM gdpr WHERE event_id IS NULL;

 count 
-------
  6692
(1 row)
```

Sanity check using the count of events having a domain:

```sql
SELECT COUNT(*) FROM gdpr WHERE event_id LIKE '%:%';

  count  
---------
 3466370
(1 row)
```

We match the total amount of events: `3466370 + 6692 = 3473062`

This is ~0.2% of all events.

### Total number of distinct rooms

```sql
SELECT COUNT(*) FROM gdpr_rooms;

 count 
-------
   470
(1 row)
```

### Number of events about the individual

```sql
SELECT COUNT(*) FROM gdpr WHERE personal = true;

 count 
-------
 77621
(1 row)
```
This is ~2% of the the distinct events.

As a sanity check, we compare to the amount of messages from the individual sent in rooms that had the `matrix.org` Homeserver involved. This simplistic query will yield a higher number than what Matrix.org actually saw, but the amount should be in the same magnitude when compared to the amount of events in the extract:

```sql
SELECT COUNT(*)
FROM events
WHERE
  sender = '[REDACTED]'
  AND room_id IN (
      SELECT DISTINCT room_id
      FROM room_memberships
      WHERE user_id LIKE '%:matrix.org'
  );
 
 count 
-------
 78135
(1 row)
```

Given the difference being less than 1%, we believe that the numbers are accurate and show that our approach is sane so far.

### Number of events related to the individual not sent by them

```sql
SELECT COUNT(*) FROM gdpr WHERE personal = true AND sender != '[Matrix ID]';

 count 
-------
  6583
(1 row)
```

### Room membership states

Overall view of the membership state for the individual at the "end" of the extract timeline:

```sql
SELECT membership,COUNT(room_id) FROM gdpr_room_membership_latest GROUP BY membership;

 membership | count 
------------+-------
 leave      |   343
 ban        |    14
 invite     |     2
 join       |   104
(4 rows)
```

### Amount of messages per type

We check what kind of events are present in the extract which gives us a first idea of what kind of data we are looking at. Some events tend to contain more personal data than other, per example `m.room.message` contain IM messages, files, pictures, videos, etc. Another example are [`m.call` events](https://matrix.org/docs/spec/client_server/r0.5.0#id89) which contain the IP(s) addresses of users which might need to be forgotten after placing a call, due to the highly personal nature.

We also look out for anomalies that would show signs of a broken processing (buggy code) that would lead to events (not) being included that should (not) be.

Per example, we see that only 467 `m.room.create` events are present, even tho we have data about 470 of them: 3 events seem to be missing from the extract. This is the first sign that the extract algorithm was not working correctly and relied on flawed access control logic.

```sql
SELECT substring(type,0,57),COUNT(*) FROM gdpr GROUP BY type ORDER BY count DESC;

                        substring                         |  count  
----------------------------------------------------------+---------
 m.room.message                                           | 2933289
 m.room.member                                            |  479557
 m.room.encrypted                                         |   21482
 m.room.redaction                                         |   18773
 xyz.maubot.bounce                                        |    4863
 m.room.power_levels                                      |    2566
 m.room.topic                                             |    2136
 m.room.aliases                                           |    1674
 m.room.history_visibility                                |     727
 m.room.join_rules                                        |     726
 m.room.name                                              |     692
 m.room.guest_access                                      |     584
 m.room.third_party_invite                                |     584
 m.room.related_groups                                    |     541
 m.room.avatar                                            |     533
 m.room.create                                            |     467
 m.reaction                                               |     421
 m.sticker                                                |     396
 opsdroid.database                                        |     395
 m.room.canonical_alias                                   |     390
 org.matrix.room.preview_urls                             |     386
 m.room._ext.enter                                        |     297
 im.vector.modular.widgets                                |     231
 m.call.candidates                                        |     161
 m.call.hangup                                            |     158
 m.room.bot.options                                       |     108
 m.room.server_acl                                        |      97
 m.room.bridging                                          |      87
 io.t2l.fixroom                                           |      68
 m.room.pinned_events                                     |      67
 m.call.invite                                            |      54
 m.call.answer                                            |      50
 m.room.plumbing                                          |      37
 org.matrix.neb.plugin.github.projects.tracking           |      31
 org.matrix.neb.plugin.jenkins.projects.tracking          |      30
 heal                                                     |      29
 m.room.encryption                                        |      29
 test                                                     |      25
 de.msg-net.sync                                          |      22
 org.matrix.neb.plugin.jira.issues.expanding              |      21
 ircd.room.revelation                                     |      18
 m.text                                                   |      15
 m.ping                                                   |      15
 py-resync                                                |      14
 ircd.test                                                |      12
 m.presence                                               |      10
 im.vector.web.settings                                   |      10
 tk.msgs.sync                                             |      10
 null                                                     |       9
 org.matrix.neb.plugin.jira.issues.display                |       8
 xyz.maubot.switch.friendcodes                            |       7
 org.matrix.neb.plugin.jira.issues.tracking               |       7
 m.room.tombstone                                         |       7
 m.typing                                                 |       6
 bs.room.host_data                                        |       6
 audiusMedia                                              |       6
 m.notice                                                 |       6
 tk.msgs.[REDATED]                                        |       6
 typing                                                   |       5
 ircd.reset                                               |       5
 m.room.calendar.event                                    |       5
 re.[REDATED].testing                                     |       5
 event_forward_extremities                                |       4
 m.topic                                                  |       4
 org.matrix.dummy_event                                   |       3
 !ping                                                    |       3
 org.matrix.test                                          |       3
 org.[REDATED].test                                       |       3
 org.sw1v.test                                            |       3
                                                          |       3
 resynctest                                               |       3
 ovh.riot.test                                            |       2
 neb.plugin.github.projects.tracking                      |       2
 m.dummy                                                  |       2
 m.room.notice                                            |       2
 m.room.social.chaos                                      |       2
 re.[REDATED].test                                        |       2
 resync                                                   |       2
 m.massage                                                |       2
 im.vector.user_status                                    |       2
 m.room.lukeb                                             |       2
 uk.org.57north.fixstate                                  |       2
 testresync                                               |       1
 "m.room.message"                                         |       1
 [REDATED]                                                |       1
 m.room.empty                                             |       1
 m.rge                                                    |       1
 m.room.encryted                                          |       1
 m.[REDATED]                                              |       1
 m.room.member [REDATED]                                  |       1
 m.but.you.are.talking.to.yourself.arent.you.questionmark |       1
 m.room.message`                                          |       1
 m.room.nam                                               |       1
 l.[REDATED].event                                        |       1
 [REDATED]                                                |       1
 ircd.bump                                                |       1
 ircd.block                                               |       1
 foo                                                      |       1
 de.mst-net.sync                                          |       1
 m.room.toys                                              |       1
 m.room_message                                           |       1
 custom-event                                             |       1
 m.sure.you.do                                            |       1
 m.syrup.snoopdog                                         |       1
 com.room.message                                         |       1
 m.unknown                                                |       1
 m.vector.web.settings                                    |       1
 m.youtubelink                                            |       1
 neb.plugin.jira.issues.display                           |       1
 com.aviraldg.commenthero                                 |       1
 m.room.config                                            |       1
 clear.extreme.[REDATED]                                  |       1
 ping?                                                    |       1
 [REDATED].fake.m.room.message                            |       1
 t2l.custom2.room.message                                 |       1
 t2l.customm.room.message                                 |       1
 a                                                        |       1
(117 rows)
```

### Sub-types of message events

The most interesting event type is `m.room.message` which is used to hold messages from users. Those messages are organised in different [sub-types](https://matrix.org/docs/spec/client_server/r0.5.0#m-room-message-msgtypes) representing the type of content the message holds.

```sql
SELECT raw->'content'->>'msgtype' AS type, COUNT(*)
FROM gdpr
WHERE type = 'm.room.message'
GROUP BY raw->'content'->>'msgtype'
ORDER BY count DESC;

                   type                    |  count  
-------------------------------------------+---------
 m.text                                    | 2655735
 m.notice                                  |  156427
 m.emote                                   |   57440
                                           |   32295
 m.image                                   |   29268
 m.file                                    |    1004
 m.video                                   |     870
 m.audio                                   |     196
 m.location                                |      12
 text                                      |       8
                                           |       7
 m.room.message                            |       6
 m.fluffychat.whisper                      |       3
 test                                      |       3
 com.aviraldg.commenthero                  |       2
 m.audius.media                            |       2
 m.fluffychat.roar                         |       2
 m.bad.encrypted                           |       2
 com.gitlab.[REDATED].fluffychat.call      |       1
 m.made-up-event                           |       1
 namespace? what namespace?                |       1
 m.fake                                    |       1
 m.calendar                                |       1
 Ｕｎｉｃｏｄｅ                            |       1
 m.room.topic                              |       1
(25 rows)
```

Empty sub-type values are from redacted events, or actually empty values.

### Avatars

Avatar URLs not fetched previously:

```sql
select count(*) from gdpr_avatars a left join remote_media_cache rmc on a.media_url = concat('mxc://',media_origin,'/',media_id) where rmc.media_id is null;
 count 
-------
 20517
(1 row)
```

### Users involved in the extract

These numbers only account for senders, and more users might be involved in the content of the messages themselves. These numbers are ballpark figures as they also involve bots, bridges, and any other kind of users which do not qualify as data subject under GDPR.

#### Total

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr;

 count 
-------
 92253
(1 row)
```

#### With non-related events

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr
WHERE sender != '[Individuals Matrix ID]' AND personal = false;

 count 
-------
 92252
(1 row)
```

#### From Matrix.org

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr
WHERE sender LIKE '%:matrix.org';

 count 
-------
 77658
(1 row)
```

This means 14595 users involved are not from Matrix.org.

#### From Freenode

Amount of users:

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr WHERE sender LIKE '%freenode%';

 count 
-------
 28404
(1 row)
```

Amount of different bridges:

```sql
SELECT COUNT(DISTINCT origin) FROM gdpr WHERE sender LIKE '%freenode%';

 count 
-------
     7
(1 row)
```

#### From other networks

Only from identified or documented bridges:

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr
WHERE
     sender LIKE '@telegram_%:%'
  OR sender LIKE '@_discord_%:%'
  OR sender LIKE '@_xmpp_%:%'
  OR sender LIKE '@_slack_matrixdotorg_%:%'
  OR sender LIKE '@gitter_%:matrix.org';

 count 
-------
  2992
(1 row)
```



## Soft leak: Events not related to the individual

### Total

```sql
SELECT COUNT(*) FROM gdpr WHERE personal != true;

  count  
---------
 3395441
(1 row)
```

About ~98% of the distinct events are not related to the individual that made the GDPR request.  
This amounts to ~3.4M events.

### Rooms

Rooms that contained events not related to the individual:

```sql
SELECT COUNT(DISTINCT room_id) FROM gdpr WHERE personal != true;

 count 
-------
   461
(1 row)
```

### Display names

Amount of display names that were never seen by the individual's server:

```sql
SELECT COUNT(*) FROM (
SELECT n.user_id, n.displayname, COUNT(rm.*) AS seen
FROM (
  SELECT DISTINCT
    raw->>'state_key' AS user_id,
    raw->'content'->>'displayname' AS displayname
  FROM gdpr
  WHERE personal = false
    AND type = 'm.room.member'
    AND raw->'content'->>'displayname' IS NOT null
) AS n
  LEFT JOIN room_memberships rm
    ON rm.user_id = n.user_id AND rm.display_name = n.displayname
GROUP BY n.user_id, n.displayname
HAVING COUNT(rm.*) = 0
) n;

 count 
-------
  3126
(1 row)
```

### Media: Avatars

```sql
create table gdpr_leak_soft_avatars as select distinct raw->'content'->>'avatar_url' as media_url from gdpr where personal = false and type = 'm.room.member' and raw->'content'->>'avatar_url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_soft_avatars;

 count 
-------
 30224
(1 row)
```

### Media: Files

```sql
create table gdpr_leak_soft_media as select distinct raw->'content'->>'msgtype' as type, raw->'content'->>'url' as media_url from gdpr where personal = false and type = 'm.room.message' and raw->'content'->>'url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_soft_media;

 count 
-------
 30556
(1 row)
```

### New media

```sql
create table gdpr_leak_soft_media_new as select m.media_url from (select media_url from gdpr_leak_soft_media union select media_url from gdpr_leak_soft_avatars) m left join remote_media_cache rm on concat('mxc://',rm.media_origin,'/',rm.media_id) = m.media_url group by m.media_url having count(rm.*) = 0;
```

```sql
select count(*) from gdpr_leak_soft_media_new;

 count 
-------
 48334
(1 row)
```



### Users impacted by the leak

These numbers only account for senders, and more users might be involved in the content of the messages themselves. These numbers are ballpark figures as they also involve bots, bridges, and any other kind of users which do not qualify as data subject under GDPR.

#### Total

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr
WHERE personal = false AND sender != '[Individuals Matrix ID]';

 count 
-------
 92252
(1 row)
```

#### From Matrix.org

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr
WHERE personal = false AND sender LIKE '%:matrix.org';

 count 
-------
 77659
(1 row)
```

This means 14593 users involved are not from Matrix.org.

#### From Freenode

Amount of users:

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr
WHERE personal = false AND sender LIKE '%freenode%';

 count 
-------
 28404
(1 row)
```

#### From other networks

Only from identified or documented bridges

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr
WHERE
  personal = false AND (
         sender LIKE '@telegram_%:%'
      OR sender LIKE '@_discord_%:%'
      OR sender LIKE '@_xmpp_%:%'
      OR sender LIKE '@_slack_matrixdotorg_%:%'
      OR sender LIKE '@gitter_%:matrix.org'
  );

 count 
-------
  2992
(1 row)
```

## Medium leak: Events not received by the individual's server

This section is related to the events that are usually not seen by the individual since the individual is not joined (yet or anymore) to the rooms involved. These events represent data that has no reasonable justification for being shared with the individual that made the request, except for a possible subset of `414` events.

### Total

```sql
SELECT COUNT(*) FROM gdpr_leak_medium;

  count  
---------
 1495974
(1 row)
```

### That have a reference to the individual

This represent the amount of events that related to the individual that they would not have been aware of, given the day-to-day usage of Matrix. These events were in scope of the GDPR Data request as they contained the identifiers (Matrix ID and/or Email) for that individual that were given in the request.

```sql
SELECT COUNT(*) FROM gdpr_leak_medium WHERE personal = true;

 count 
-------
   414
(1 row)
```

### Per room

Breakdown of events count for each room involved in the GDPR extract. Reminder that the Room ID is partial for privacy reason, but still unique enough to be found in the databases of the servers present in them.

```sql
SELECT substring(room_id,0,15),COUNT(*)
FROM gdpr_leak_medium
GROUP BY room_id ORDER BY count DESC, room_id ASC;

   substring    | count  
----------------+--------
 !UcYsUzyxTGDxL | 343140
 !cURbafjkfsMDV | 260793
 !BbUcoMPQhtUNJ | 158387
 !XqBunHwQIXUiq |  95674
 !DgvjtOljKujDB |  72069
 !tYmNuaXMRnHEZ |  66852
 !DdJkzRliezrwp |  62571
 !DwmKuvGvRKciq |  30049
 !uDQoIebqsjEEt |  29629
 !YOdvwVKQSEZlj |  28767
 !GnEEPYXUhoaHb |  27707
 !vfFxDRtZSSdsp |  26834
 !HCXfdvrfksxuY |  26059
 !SudviOJlimDvr |  19040
 !OIwzuTNQegQIA |  15374
 !tDRGDwZwQnlko |  13773
 !wzHrsErnsyaqb |  12991
 !gRrCzddQslIHo |  12310
 !PCzUtxtOjUySx |  11824
 !rhpXsXjZAQRWH |  10710
 !SrHINiUkedNXD |  10662
 !boLskYiwabbCQ |   9419
 !fLXbhWnqiSoyB |   9300
 !MbRaSiMIRhhxD |   8023
 !BPvgRcBVHzyFS |   7714
 !QtykxKocfZaZO |   7324
 !RJFCFtixHgPhz |   6350
 !TwyvcuLABIpuE |   5862
 !KzHjwnhdwaLkj |   5061
 !zySiFNhIBsCmR |   4424
 !zBMUYyApaSEJL |   4199
 !yZHTGeDKZUeKa |   3927
 !OYyXUbcTKVsDB |   3566
 !WZbLERcNJxrvk |   3365
 !hwiGbsdSTZIwS |   3322
 !HsxjoYRFsDtWB |   3048
 !oVdyFBNJbncvg |   3046
 !fGpPGxhShCvzX |   3044
 !yMPZjINQYAUeJ |   2992
 !FyQrGcOoVamcL |   2967
 !SrBFnGbFvAfkL |   2724
 !svJUttHBtRMdX |   2323
 !qYuPEjvrchmJn |   2029
 !qPNOnUsNsZrTo |   2018
 !PeOkQLnTUKVtu |   1992
 !ChuQQIVJvwyJu |   1967
 !MkOEdAuxWAGsv |   1876
 !FVLTSqqprVFuS |   1787
 !cnYmJsupkkKzT |   1784
 !bjmvDTtwYdtMc |   1734
 !IadFDmzRTSlgz |   1684
 !qBFNwucQebGPQ |   1477
 !VDSHqoZBTefnX |   1420
 !gkpqsseYVzWQk |   1358
 !blHeuKmYNNfIq |   1345
 !iNmaIQExDMeqd |   1333
 !zTIXGmDjyRcAq |   1287
 !HwocBmCtBcHQh |   1253
 !rUpoBIkqaoOXq |   1186
 !uDPWSkzohqjVw |   1185
 !jBgeKJHcBGVKh |   1088
 !AAAANTUiY1fBZ |   1069
 !tPaEUwcgnyarD |   1051
 !pCujylGSQwfVb |   1029
 !LgffRGCXiGOZg |    982
 !KWFFgTfdrmuCu |    951
 !etgMkbhwIvWhR |    949
 !eUGMvloIjhBoA |    859
 !orhvUCVfMDCHo |    788
 !ZclcEpFTORTjm |    786
 !oAzXMbPSeRDsN |    781
 !WDguMmdNdRrqE |    738
 !cApDlsILLZkmA |    727
 !NnpdIGrSqllLs |    640
 !YHhmBTmGBHGQO |    612
 !NXrvsnxCRgMdL |    599
 !FXXkFzPnpVCmw |    591
 !oxGQbqwErOmUq |    580
 !NZMOxbULGynZS |    569
 !ZjJkQNQAlmRlG |    562
 !colPXFbzasyys |    554
 !VcucjFBPGgOCx |    539
 !wFOSHHabtfRcR |    493
 !vSyFKxtfVhfbi |    408
 !FPUfgzXYWTKgI |    385
 !gZaRdadhwPaPQ |    312
 !htOanVjArJyYU |    276
 !HRNVeGThDQRMY |    272
 !IgoVCuQKBuAtE |    267
 !PGaxNzvxuqcBX |    241
 !evAqkPHZuPHIG |    221
 !BTjVdFSwvEkTD |    220
 !qKpaYkJrMmWJk |    207
 !zCLWAbYiBPnmF |    207
 !gHhrLEfqoCgmR |    199
 !YDwlnxbKxEYzj |    198
 !efedUfUxxyRTa |    198
 !JJHiJkTxxJiGA |    196
 !VExuChwLUaNnI |    196
 !NwLDclCUYcfkZ |    195
 !PxMWbvomtRqGO |    188
 !LnIURtMENeGIP |    184
 !MiaZvYagNIApU |    180
 !XIydWLOEJvcDB |    176
 !KmtCqtUMyzWKT |    175
 !ZkDFDHHVxVwJr |    168
 !pJwzQQUCraoWt |    166
 !SEKsMhpBOKQFh |    157
 !pLNftUcxUgPyS |    144
 !hJGrZZlxBAzps |    143
 !TInBDrtHjZBjU |    137
 !WGRPyrAYebnQH |    133
 !QuvwCZDOuPQzG |    129
 !dXgJygZcurLmm |    123
 !YkZelGRiqijtz |    122
 !tdrTPBnFTAUPh |    119
 !etUivBFgFPNCq |    118
 !iLajGEwovCOwp |    115
 !DYgXKezaHgMbi |    112
 !iDlXcgpWeKYoX |    104
 !GEweEzjaaSgao |    101
 !rPrXjIlQNLAjO |     97
 !bPGsIuotKiRcJ |     92
 !CszCrGEsVJERG |     91
 !UCxzezTXzvCIh |     84
 !rBgKOuqmjhJzQ |     84
 !wwGQryJUcMWcj |     82
 !AMRItmdwTPmtI |     81
 !XZYWDXBgmeMVp |     81
 !boGLIDBIkrNod |     80
 !uewiilduiDRfP |     79
 !nxOySBqziItJd |     78
 !PuCCCAWGBCkbR |     77
 !PukFFdIcHgtaa |     76
 !MRDYgvSSrLVHF |     75
 !WElxiGqlWmrdY |     75
 !MhJgddNhalZCK |     72
 !XlzpnlSIOSkbn |     70
 !uSHOhUcNIMioM |     70
 !mCUETrwJCsmUR |     69
 !pzzTxdMzTgaZN |     66
 !ZAXhfRqRdNtty |     63
 !yGzSFCGzQySqD |     62
 !XCLLzPDuMPDTw |     61
 !hNdeyGkXjZXtN |     61
 !iuOFqcgBzrpFM |     61
 !nlyPjQTGCOOwo |     60
 !ElbSWdqiIEqix |     59
 !lDjTNkSxAWNzn |     59
 !BdLmuYxxJLTcS |     58
 !ZmSZzxujGNoCy |     58
 !lnjwXgUmVSYdx |     56
 !PtKxxSgyjbNPr |     55
 !NgPZEHUZKexBT |     52
 !uFhErnfpYhhla |     51
 !IsiXQMluQPunv |     50
 !CwOrUlHbsjJZs |     47
 !nEvnOkbKZMEqi |     47
 !mPYrVomQESEYW |     45
 !CdWhIoBWTEpbX |     41
 !EtKiyhLuMHgBx |     41
 !lBbDUDayNiOPv |     41
 !VATZpnpoHylMI |     39
 !lOlhHqkXrBEVo |     39
 !LagvxzDzVtchB |     38
 !QhQlEwIKnYlBR |     38
 !pJWcqvNgRnTcj |     38
 !eoqUnZuNQOycY |     37
 !rVyyCEILieSgP |     37
 !UBYOHQZrjZwap |     36
 !VpacCXZFLTrQd |     36
 !ivlIxGBUieUFf |     36
 !BfcuvYZbkaJDY |     35
 !QZfijrbditXOT |     35
 !RcUaHZNbRmKiM |     35
 !YEMrjmUnyWDXx |     35
 !dMDtGbujSAcxo |     34
 !gNJKLKBQQqLMX |     34
 !NvNzoExKIggXP |     33
 !rrBOGOnXFEwBI |     33
 !umCdWUiIaJrNO |     33
 !GwSiDzLhevmUH |     32
 !FDfHoQvKPWchO |     31
 !RbbuPYoqQuKqm |     31
 !rGnsidGhpRsBP |     31
 !JEkIjEJVcXwkH |     30
 !vSaoPgXctLrBw |     30
 !LppXGlMuWgaYN |     28
 !nVPEpbhLGLXoD |     28
 !LmMZjOMkFYxMD |     26
 !fteHUSpasKmIL |     26
 !hHRqGOTnPebSV |     26
 !mzOQREsifihQo |     24
 !dFbWzOlivjbJV |     23
 !nfPiwyCnpzSHA |     23
 !ybKYZRhSUcVVX |     23
 !RxbgExYqTqbzP |     22
 !eYssLbfpyZXhU |     21
 !OGmvpSCWTghkd |     20
 !bJbQfMQgbpNXx |     20
 !wLDnqCZwHlErX |     20
 !VaENXDoUVoCPE |     19
 !DGzyCNYwKufHp |     18
 !SFoAyyrssKVmz |     18
 !tqjmRtiOTvQNY |     18
 !TjpQZyMfonHCO |     17
 !lEmzNCalwGjKv |     17
 !oUTlLrwlFzJTH |     17
 !BfeiywfnndUqt |     16
 !gaSksXUwCVFzP |     16
 !HrRbUivowOaiM |     15
 !OluOzqbTJgecd |     15
 !fyzEKXRlOnbNT |     15
 !njlAtOfBXEoOo |     15
 !qvCRFpZsGETXt |     15
 !fILrmqIZQfJcr |     14
 !kpudCQcQiXBGy |     14
 !mkIKdwhXkOQeM |     14
 !FArEnfvrFQccP |     13
 !GdUnrYTVDddhg |     13
 !avWcCZExgSbRE |     13
 !nBkVPUdQZoLOI |     13
 !vFqbnsHvBiklY |     13
 !DhKMQbtOKDEUA |     12
 !PtsuqXwfpyCtC |     12
 !UTxvuVFUFoNXn |     12
 !jRlMMIqJDVavI |     12
 !kUSLRbkbZQBnJ |     12
 !ruaviCwHdJSWf |     12
 !CRfrAqMWDFIWV |     11
 !gUbnAcpdLfguU |     11
 !FSIklbEejzuft |     10
 !GanozlTDQJvCq |     10
 !IdFpTbyBVJGGt |     10
 !VSIAAfGoVlYVj |     10
 !cBhajBZpQHHSm |     10
 !olAEUPfceghKn |     10
 !oqITxomYWbosR |     10
 !rAucCyhfijClI |     10
 !vntvfhJElvfkz |     10
 !zZxEBPZIsvPlN |     10
 !CXcPbImEZwbtg |      9
 !IRJFHmOPozXME |      9
 !KCuqUoXIvtMmg |      9
 !PUVWAjdgrMkiN |      9
 !QnVITTFaszHHV |      9
 !QvorQCuKfyJSS |      9
 !SjpjUdpQbCnma |      9
 !XFTqhogTmStUv |      9
 !YLUonUPZxltZV |      9
 !jCybpfJZlDzeu |      9
 !qKoCkpMpjkgtn |      9
 !rAvPpowSRFosM |      9
 !umvUDthkSyIFX |      9
 !wzTQlJDCNTnpH |      9
 !xRWcCIOVueTCU |      9
 !zYqMWFUVGFeeL |      9
 !AdNymCYPKJCVp |      8
 !AqyfBZHtrZUVq |      8
 !ENmhzuITtbaIg |      8
 !FaLYCIoNwsHXs |      8
 !HTuPUFgdxBIKy |      8
 !HUeDbmFUsWAhx |      8
 !HyCsIIwtkCbJe |      8
 !JACYZNEjPXxkf |      8
 !KHPHxqIovKchy |      8
 !KtUIjQTYxdsUQ |      8
 !NAExjCAsmqwgP |      8
 !NQuglAWZrBxrJ |      8
 !NcRYgcDrzIBlz |      8
 !OPgEkNPLLvIoE |      8
 !PHBVNXduzUpUL |      8
 !PYqMetBMXFBQc |      8
 !ScSQPFHrlAYTe |      8
 !ThRvqZqVbNWHA |      8
 !ZDCRKKXDgLFUm |      8
 !aNXzKbDwRZRhm |      8
 !aeGDwkhcSVorU |      8
 !avnYIBBmXUwrf |      8
 !bsNswiCGSFHkd |      8
 !hNVccTgJpNTgm |      8
 !jKvymBitRTfDm |      8
 !jeUfTgxdIvhLL |      8
 !nLJYxDZtzYJBE |      8
 !rhoyisVXVeXes |      8
 !sIEDEnwPStTlP |      8
 !sVmyXiNGjSZim |      8
 !uLICUxuLgtRhF |      8
 !uWZBrVvrUUhBD |      8
 !vpeuPWQKgfqfU |      8
 !wmsrVdTqfjyax |      8
 !AMNVffNvSVMJr |      7
 !AQkPdGhhLgdPt |      7
 !DeNUdQIzHgdqI |      7
 !DiKjsbrAUpsYU |      7
 !DlQdqCqbyuEhV |      7
 !EkokgLyRCTwFS |      7
 !FNRUqBcOmvzFO |      7
 !FZPwbwLoaefYc |      7
 !GqSiFARwpgGso |      7
 !HCjaPBIqGQkaB |      7
 !KOWviGCtQHnSM |      7
 !LHqJKzHdygjws |      7
 !LPAujxIZzSCxV |      7
 !McUkmbGfiwwJe |      7
 !NotYJfZdMyUNf |      7
 !PnvGrlynHpyrB |      7
 !QmasvhaKJeSaw |      7
 !RaDPtWIFlRLCW |      7
 !RduKEIaWPqHJf |      7
 !RpOjogffMRkjM |      7
 !YzOxyHUIBwmUd |      7
 !aPdKSMLYKmZSN |      7
 !aQcQJnCkQhqvD |      7
 !bKKsmxbIfxKDy |      7
 !eKveJTnascxvr |      7
 !eizbIipopDNGH |      7
 !fQnlkVruxgWRi |      7
 !fyiJEfqeZxAPD |      7
 !jKWdJhpLUgqim |      7
 !jjOzqLiIhgXoI |      7
 !kIIfAkMTUESYk |      7
 !kluBlUgXejRUg |      7
 !mGKjYgbcAbEUx |      7
 !nTVcZncFwSAQP |      7
 !nbeOMzUtUfjgJ |      7
 !oBNMiqQZmpanx |      7
 !phELVaSWcbIiL |      7
 !qCILpUsiinQMg |      7
 !qZASJfQaHYMEp |      7
 !rPwAlRgqZjnTk |      7
 !svdGHMOkISdkN |      7
 !tbSfNVMmUQMjO |      7
 !uUwivGwOrAxlo |      7
 !vNCcHSbbHXLYC |      7
 !xcupIpVuIRHRB |      7
 !zgyBHpvHQroCM |      7
 !zxPuDSPSNzZpg |      7
 !LZDPbokFjsCQx |      6
 !xRbDubdoPMugQ |      6
 !CmGmFdjwSxrPp |      5
 !rGqBfowRYlDtr |      5
 !VQnUtrwLRKRQc |      4
 !BDdMvwWbCbvaj |      1
 !BqJmNHhDBZJOU |      1
 !BzqvWHPaLKaeu |      1
 !CCEZyzOrThevS |      1
 !CNegDiRTUackK |      1
 !CqHkcDvPHsynX |      1
 !DasXYiCUXHRde |      1
 !DcMnActIMSMcJ |      1
 !DicQyMJzILZyd |      1
 !EicfUlejkfPxW |      1
 !EjzGdVITybOAQ |      1
 !EnvwdnXVniUDP |      1
 !FqjSxPMWIsbhX |      1
 !GEOldHgnmkmQk |      1
 !HcwsdqEHceuCH |      1
 !JBHDybgEvxuYU |      1
 !JJdNixvVBlspt |      1
 !JTgmxLvWAtlvR |      1
 !JjnrOQcTVYQVo |      1
 !KCgLjpNDdrwmy |      1
 !KDFidFyVsBFSQ |      1
 !KcCbplxvKaYSE |      1
 !KpQZNtijOJKzF |      1
 !KygQjCPxwsMhB |      1
 !LEaSuVyJfLqHC |      1
 !LOejfckOcMdTS |      1
 !LoofsapunTpgs |      1
 !LtFjpWghCdHXK |      1
 !MDGUnxWASkbvk |      1
 !NEsuEfLBVEaVf |      1
 !NPRUEisLjcaMt |      1
 !NVkDBMfJIeHtD |      1
 !OvbhAHGYMCSRo |      1
 !PGymJuwWJImUF |      1
 !PgXSOaEHorTEj |      1
 !RAkmJhUsvzfbr |      1
 !RJbvbSOFnuKmA |      1
 !ROByHNfgGtUna |      1
 !RlmAHyIuKhyVy |      1
 !SSkhqpwWmbDPT |      1
 !SumQwLrWFeLSw |      1
 !TMXgDpfyTnhVj |      1
 !TifqosozuzVES |      1
 !TwkMTAYEQrXKe |      1
 !VHCTHMiSzcIvx |      1
 !VeHsXcugnlowL |      1
 !VoxICeNvdbUVk |      1
 !VvMuMiIEmFYjp |      1
 !XrpEkljFjPdML |      1
 !XvqAhWZgiCiFS |      1
 !XwLpFeZdUhiMH |      1
 !YpdrgmeBZqaXE |      1
 !ZMOtrMFnCcMic |      1
 !ZSCxwXvUCrHbX |      1
 !aWGizUZfDonDT |      1
 !bAyKnwGMhbOqu |      1
 !bWsecdPuYSzMh |      1
 !cDuIBnFDyveJt |      1
 !cNndrveRGjoZQ |      1
 !cdRjuElFFzwrr |      1
 !cfIoweTSSMRvt |      1
 !czMYVoFDvHYBw |      1
 !dVodwlzkwtbDv |      1
 !dcVHfBjQRAwbf |      1
 !dxoxKujQBEIOX |      1
 !ekfbWgYXoTJjz |      1
 !fDhTpwkXEHDcx |      1
 !fGwwGqSpYYQav |      1
 !fQxAyfvcUDMiv |      1
 !fyRzhvFivwLiz |      1
 !gRIgUrCbgBpbI |      1
 !gXIztTlEeDruz |      1
 !gZBacciyUfqcx |      1
 !jKMAUualYoxUw |      1
 !jXEBfGLgWyhPP |      1
 !jYdWXZQCXtCvO |      1
 !jrSziltwFWaIr |      1
 !juOoLAgqAOiyI |      1
 !kQoWiTzWrbAYS |      1
 !kYSlDiWbSkRBD |      1
 !kZCAtHxTnhczl |      1
 !lHayZrusjZJXy |      1
 !lSpgqeFaUqejO |      1
 !lZtaCFMAWvjoS |      1
 !lcOSsUiAgOwXc |      1
 !lgRlvbIVNMBXe |      1
 !lnClIVCKeunkc |      1
 !mrHnfvxriYMXq |      1
 !nTzmyGSxcPhBC |      1
 !pDFuXrtXnmxJA |      1
 !pJttsHoGvpxDR |      1
 !pzDzXARMkYuWH |      1
 !qqyBQslOrNRrw |      1
 !rElSjRdVBYASa |      1
 !rKDtLwIOcGYQa |      1
 !rYTglJxrXFSjJ |      1
 !rnPIlwjyFjkXn |      1
 !sNNrqjLEhDzmm |      1
 !tIcIdyrNZQiEq |      1
 !tOsTVRQhJNAwu |      1
 !tQlMVJcGErWHQ |      1
 !urnYiQEeCOcxS |      1
 !uwkxdtLtEdLSf |      1
 !uwunOYtZqoVJN |      1
 !vacOHsmWmLYbC |      1
 !voSXuGebTnVLh |      1
 !yPgVCClQmpsrB |      1
 !yTLlgttSCmoIN |      1
 !zWatgcJLRnkJp |      1
(452 rows)
```

We also see that 452 rooms are involved.

### Type breakdown

```sql
SELECT substring(type,0,57), COUNT(*)
FROM gdpr_leak_medium
GROUP BY type ORDER BY count DESC;

                        substring                         |  count  
----------------------------------------------------------+---------
 m.room.message                                           | 1354528
 m.room.member                                            |  126279
 m.room.redaction                                         |    6742
 m.room.power_levels                                      |    1297
 m.room.topic                                             |    1246
 m.room.aliases                                           |    1021
 m.room.join_rules                                        |     546
 m.room.name                                              |     545
 m.room.history_visibility                                |     539
 m.room.third_party_invite                                |     522
 m.room.create                                            |     452
 m.room.guest_access                                      |     412
 m.room.avatar                                            |     361
 m.room.canonical_alias                                   |     280
 m.room.related_groups                                    |     205
 org.matrix.room.preview_urls                             |     160
 im.vector.modular.widgets                                |     129
 m.reaction                                               |      89
 m.room.bot.options                                       |      79
 m.sticker                                                |      75
 m.call.hangup                                            |      74
 m.room.bridging                                          |      52
 m.room._ext.enter                                        |      46
 org.matrix.neb.plugin.github.projects.tracking           |      31
 m.call.candidates                                        |      30
 org.matrix.neb.plugin.jenkins.projects.tracking          |      30
 m.room.plumbing                                          |      27
 org.matrix.neb.plugin.jira.issues.expanding              |      21
 io.t2l.fixroom                                           |      21
 m.call.answer                                            |      16
 m.ping                                                   |      14
 m.room.server_acl                                        |      11
 m.room.encrypted                                         |      11
 m.call.invite                                            |      10
 m.room.pinned_events                                     |       9
 org.matrix.neb.plugin.jira.issues.display                |       8
 org.matrix.neb.plugin.jira.issues.tracking               |       7
                                                          |       3
 bs.room.host_data                                        |       3
 m.room.encryption                                        |       3
 m.notice                                                 |       3
 m.room.calendar.event                                    |       2
 m.topic                                                  |       2
 opsdroid.database                                        |       2
 ircd.test                                                |       2
 m.room.lukeb                                             |       2
 org.matrix.test                                          |       2
 tk.msgs.sync                                             |       2
 neb.plugin.github.projects.tracking                      |       2
 m.massage                                                |       2
 neb.plugin.jira.issues.display                           |       1
 com.aviraldg.commenthero                                 |       1
 m.syrup.snoopdog                                         |       1
 m.but.you.are.talking.to.yourself.arent.you.questionmark |       1
 test                                                     |       1
 m.room.nam                                               |       1
 m.youtubelink                                            |       1
 m.room.tombstone                                         |       1
 l.lukeb.event                                            |       1
 m.unknown                                                |       1
 m.room.config                                            |       1
 m.nepugia                                                |       1
 m.text                                                   |       1
 m.dummy                                                  |       1
 com.room.message                                         |       1
 m.room.member [REDACTED]                                 |       1
 t2l.custom2.room.message                                 |       1
 m.sure.you.do                                            |       1
 t2l.customm.room.message                                 |       1
(69 rows)
```

### Message type breakdown

```sql
SELECT raw->'content'->>'msgtype' AS type, COUNT(*)
FROM gdpr_leak_medium
WHERE type = 'm.room.message'
GROUP BY raw->'content'->>'msgtype'
ORDER BY count DESC;

           type           |  count  
--------------------------+---------
 m.text                   | 1215737
 m.notice                 |   70677
 m.emote                  |   32098
                          |   21161
 m.image                  |   13857
 m.video                  |     470
 m.file                   |     415
 m.audio                  |      85
 m.location               |      10
 m.room.message           |       5
 m.fluffychat.whisper     |       3
 com.aviraldg.commenthero |       2
 text                     |       2
 m.fluffychat.roar        |       2
 test                     |       2
 m.calendar               |       1
 m.fake                   |       1
(17 rows)
```

Empty type are redacted events.

### Display names

Amount of display names that were never seen by the individual's server:

```sql
SELECT COUNT(*) FROM (
SELECT n.user_id, n.displayname, COUNT(rm.*) AS seen
FROM (
  SELECT DISTINCT
    raw->>'state_key' AS user_id,
    raw->'content'->>'displayname' AS displayname
  FROM gdpr_leak_medium
  WHERE type = 'm.room.member' AND raw->'content'->>'displayname' IS NOT null
) AS n
  LEFT JOIN room_memberships rm
    ON rm.user_id = n.user_id AND rm.display_name = n.displayname
GROUP BY n.user_id, n.displayname
HAVING COUNT(rm.*) = 0
) n;

 count 
-------
  2931
(1 row)
```

### Media: Avatars

```sql
create table gdpr_leak_medium_avatars as select distinct raw->'content'->>'avatar_url' as media_url from gdpr_leak_medium where type = 'm.room.member' and raw->'content'->>'avatar_url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_medium_avatars;

 count 
-------
 12736
(1 row)
```

### Media: Files

```sql
create table gdpr_leak_medium_media as select distinct raw->'content'->>'msgtype' as type, raw->'content'->>'url' as media_url from gdpr_leak_medium where type = 'm.room.message' and raw->'content'->>'url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_medium_media;

 count 
-------
 14705
(1 row)
```

### New media

```sql
create table gdpr_leak_medium_media_new as select m.media_url from (select media_url from gdpr_leak_medium_media union select media_url from gdpr_leak_medium_avatars) m left join remote_media_cache rm on concat('mxc://',rm.media_origin,'/',rm.media_id) = m.media_url group by m.media_url having count(rm.*) = 0;
```



```sql
select count(*) from gdpr_leak_medium_media_new;

 count 
-------
 23914
(1 row)
```



### Users impacted by the leak

These numbers only account for senders, and more users might be involved in the content of the messages themselves. These numbers are ballpark figures as they also involve bots, bridges, and any other kind of users which do not qualify as data subject under GDPR.

#### Total

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_medium
WHERE sender != '[Individuals Matrix ID]';

 count 
-------
 34945
(1 row)
```

#### With non-related events

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_medium
WHERE sender != '[Individuals Matrix ID]' AND personal = false;

 count 
-------
 34945
(1 row)
```

#### From Matrix.org

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_medium
WHERE sender LIKE '%:matrix.org';

 count 
-------
 29011
(1 row)
```

This means 5934 users involved are not from Matrix.org.
#### From Freenode

Amount of users:

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr_leak_medium WHERE sender LIKE '%freenode%';

 count 
-------
  9812
(1 row)
```

#### From other networks

Only from identified or documented bridges

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_medium
WHERE
     sender LIKE '@telegram_%:%'
  OR sender LIKE '@_discord_%:%'
  OR sender LIKE '@_xmpp_%:%'
  OR sender LIKE '@_slack_matrixdotorg_%:%'
  OR sender LIKE '@gitter_%:matrix.org';

 count 
-------
   774
(1 row)
```

## Hard leak: Events bypassing access control

### Total

```sql
SELECT COUNT(*) FROM gdpr_leak_hard;

 count 
-------
  9055
(1 row)
```

### That have a reference to the individual

This represent the amount of events that related to the individual that they could not have access to using Matrix access control. These events were in scope of the GDPR Data request as they contained the identifiers (Matrix ID and/or Email) for that individual that were given in the request.

```sql
SELECT COUNT(*) FROM gdpr_leak_hard WHERE personal = true;

 count 
-------
     0
(1 row)
```

There was no event in the extract that matched.

### Per room

Breakdown of events count for each room involved in the GDPR extract. Reminder that the Room ID is partial for privacy reason, but still unique enough to be found in the databases of the servers present in them.

```sql
SELECT substring(room_id,0,15),COUNT(*)
FROM gdpr_leak_hard
GROUP BY room_id ORDER BY count DESC, room_id ASC;

   substring    | count 
----------------+-------
 !QtykxKocfZaZO |  7300
 !iNmaIQExDMeqd |   693
 !FyQrGcOoVamcL |   572
 !FPUfgzXYWTKgI |   342
 !svJUttHBtRMdX |   148
(5 rows)
```

We see that 5 rooms are involved.

### Type breakdown

```sql
SELECT substring(type,0,57), COUNT(*)
FROM gdpr_leak_hard
GROUP BY type ORDER BY count DESC;

         substring         | count 
---------------------------+-------
 m.room.member             |  7898
 m.room.message            |  1070
 m.room.aliases            |    42
 m.reaction                |    22
 m.room.power_levels       |     7
 m.room.server_acl         |     4
 m.room.redaction          |     3
 m.room.topic              |     3
 opsdroid.database         |     2
 im.vector.modular.widgets |     1
 m.room.canonical_alias    |     1
 m.room.third_party_invite |     1
 m.room.tombstone          |     1
(13 rows)
```

### Message type breakdown

```sql
SELECT raw->'content'->>'msgtype' AS type, COUNT(*)
FROM gdpr_leak_hard
WHERE type = 'm.room.message'
GROUP BY raw->'content'->>'msgtype'
ORDER BY count DESC;

   type   | count 
----------+-------
 m.text   |   583
 m.notice |   473
 m.emote  |     7
 m.image  |     4
          |     3
(5 rows)
```

Empty type are redacted events.

### Media files

Potential leaks, given that some may already be known by the server from previous events.

```sql
SELECT COUNT(*) FROM gdpr_leak_hard WHERE raw::text LIKE '%mxc://%';

 count 
-------
  1913
(1 row)
```

### Display names

Amount of display names that were never seen by the individual's server:

```sql
SELECT COUNT(*) FROM (
SELECT n.user_id, n.displayname, COUNT(rm.*) AS seen
FROM (
  SELECT DISTINCT
    raw->>'state_key' AS user_id,
    raw->'content'->>'displayname' AS displayname
  FROM gdpr_leak_hard
  WHERE type = 'm.room.member' AND raw->'content'->>'displayname' IS NOT null
) AS n
  LEFT JOIN room_memberships rm
    ON rm.user_id = n.user_id AND rm.display_name = n.displayname
GROUP BY n.user_id, n.displayname
HAVING COUNT(rm.*) = 0
) n;

 count 
-------
    11
(1 row)
```

### Avatars

```sql
select count(distinct raw->'content'->>'avatar_url') from gdpr_leak_hard where type = 'm.room.member' and raw->'content'->>'avatar_url' like 'mxc://%';

 count 
-------
  1472
(1 row)
```

### Media: Avatars

```sql
create table gdpr_leak_hard_avatars as select distinct raw->'content'->>'avatar_url' as media_url from gdpr_leak_hard where type = 'm.room.member' and raw->'content'->>'avatar_url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_hard_avatars;

 count 
-------
  1472
(1 row)
```

### Media: Files

```sql
create table gdpr_leak_hard_media as select distinct raw->'content'->>'msgtype' as type, raw->'content'->>'url' as media_url from gdpr_leak_hard where type = 'm.room.message' and raw->'content'->>'url' like 'mxc://%';
```



```sql
select count(*) from gdpr_leak_hard_media;

 count 
-------
     4
(1 row)
```

### New media

```sql
create table gdpr_leak_hard_media_new as select m.media_url from (select media_url from gdpr_leak_hard_media union select media_url from gdpr_leak_hard_avatars) m left join remote_media_cache rm on concat('mxc://',rm.media_origin,'/',rm.media_id) = m.media_url group by m.media_url having count(rm.*) = 0;
```



```sql
select count(*) from gdpr_leak_hard_media_new;

 count 
-------
   450
(1 row)
```



### Users impacted by the leak

These numbers only account for senders, and more users might be involved in the content of the messages themselves. These numbers are ballpark figures as they also involve bots, bridges, and any other kind of users which do not qualify as data subject under GDPR.

#### Total

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_hard
WHERE sender != '[Individuals Matrix ID]';

 count 
-------
  5873
(1 row)
```

#### With non-related events

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_hard
WHERE sender != '[Individuals Matrix ID]' AND personal = false;

 count 
-------
  5873
(1 row)
```

#### From Matrix.org

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_hard
WHERE sender LIKE '%:matrix.org';

 count 
-------
  4855
(1 row)
```

This means 1018 users involved are not from Matrix.org.

#### From Freenode

Amount of users:

```sql
SELECT COUNT(DISTINCT sender) FROM gdpr_leak_hard WHERE sender LIKE '%freenode%';

 count 
-------
  2018
(1 row)
```

#### From other networks

Only from identified or documented bridges

```sql
SELECT COUNT(DISTINCT sender)
FROM gdpr_leak_hard
WHERE
     sender LIKE '@telegram_%:%'
  OR sender LIKE '@_discord_%:%'
  OR sender LIKE '@_xmpp_%:%'
  OR sender LIKE '@_slack_matrixdotorg_%:%'
  OR sender LIKE '@gitter_%:matrix.org';

 count 
-------
    45
(1 row)
```