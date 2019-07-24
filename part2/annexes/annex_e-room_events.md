# Annex E - GDPR Data Extract - Room events

This annex is part of the [Notes on privacy and data collection of Matrix.org, Part 2](../README.md) research document.

**If you would like to link to this document, you may only do so using the header link of the Annexes section title found on the [main document](../README.md) to ensure vital context is seen by any reader.**

This transcript is provided for educational purposes only, in the public interest of individuals who want to understand and see how a GDPR Request in the scope of the Matrix protocol. Use of this document, for any other purpose, is **NOT** allowed without the express approval of the authors of the research document. The transcript is personal data of an individual performing a GDPR Request under EU GDPR law.

---

This document contains a set of tables, extracts and correlated data to validate the claims done on the main research document.

All Matrix IDs are partial and the last characters have been redacted. To match in your database, use the following approach, replacing `<VALUE>:

- For Room IDs: `WHERE room_id LIKE '<VALUE>%'`
- For Event IDs: `WHERE event_id LIKE '<VALUE>%'`

---

Layout of the ZIP file:

```
export-[REDACTED]
\--- rooms
     +--- !AA-RoomID:example.org
     |    +--- events
     |    \--- state
     |         +--- $EventIdOne:example.org
     |         +--- $EventIdTwo:example.org
     |         \--- $EventIdThree:example.org
     |--- !AB-RoomID:example.org
     |     \--- events
     |
     +--- ...
     |
     \--- ...
```

---

Total bytes size:

```bash
$ du -bd 1 | grep rooms
8687851718	./rooms
```

---

Total number of raw events:
```bash
$ cd rooms
$ find . -type f -print0 | wc -l --files0-from - | grep total
7660193 total
```

---

Total number of E2EE events:

```bash
$ grep -riF 'm.room.encrypted' * | wc -l
21556
```

Only `0.28 %` was encrypted.

---

Numbers for two types of servers:

- A corporate server, labelled "*Corp*", that:
  - Only has a handful of individuals using it on a daily basis
  - Is for users who have advanced technical knowledge about IT and Matrix
  - Is joined to all big rooms of the federation (Matrix HQ, Riot, etc.)
- A Friends & Family server, labelled "*F&F*", that:
  - Has about 20 individuals using it on a daily or weekly basis
  - Is for users without any significant IT knowledge that use it as an alternative to Facebook Messenger, Whatsapp, etc.
  - Is not joined to the open federation, and only connects with two other servers of similar size and purpose.

Over the last 30 months, we see the following numbers:

|  | People | Event Count | % of Extract | Event Size (bytes) | % of Extract |
|---|-----------|----------|-------|--------|--------|
|**Corp**| `2` | `5102055` | `66.604 %` | `11811160064` | `135.95 %` |
| **F&F** | `17` | `158767` | `2.07 %` | `191889408` | `2.21 %` |

---

Number of events generated per user on each using query (will also list bots that are counted as users, but ignore appservice users). This includes all events, not just IM messages.

```sql
SELECT u.name, COUNT(e.*) AS total
FROM users u LEFT JOIN events e ON e.sender = u.name
WHERE appservice_id IS NULL
GROUP BY u.name ORDER BY total DESC;
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

And as a sanity check, over the same period:

- Count of events visible from the Corp server, sent by one member of the `matrix.org` core team: `41430`
- Count of events visible from `matrix.org` (common rooms), sent by the top Corp user: `76588`

Highest numbers are all in the same order of magnitude and in line with the known activity of those users, if compared to each other.

## Room analysis

### Overview

What one would normally check for:

- By membership:
  - Rooms that the individual left, as a result of a ban.
  - Rooms that the individual left, as a result of a kick.
  - Rooms that the individual left, as a result of leaving.
  - Rooms that the individual is currently invited to.
  - Rooms that the individual is currently joined to.
  - Rooms where the event visibility would prevent us from seeing an event, but would allow matrix.org HS to see it (joined before us per example).
  - Rooms where we are joined, that are private (invite only, read if joined) where matrix.org bot/appservice is present.

### Rooms that the individual is no longer in

This extract checks the data dump for rooms from which the individual was banned, which is the "strongest" level of access control on the Matrix platform, and thus must be also respected in a GDPR data dump. Some of those rooms were test rooms not involving Matrix.org and should not appear in the data dump.

We check that:

- Only events that are directly related to the individual have been provided, e.g. those containing their Matrix ID.
- No event that was sent to the room after leaving it is provided, enforcing the access control of the system that the Privacy Notice claims to follow.

#### As seen by the Individual's Homeserver

The following table is an extract of the Homeserver database serving the Individual with the list of rooms they are banned from, and the corresponding Event ID of the ban. That event represent the moment in time after which the individual should not be able to access any newer events when acting as a regular user and within the context of a GDPR data access request.

```sql
SELECT room_id,event_id FROM room_memberships WHERE user_id = '[REDACTED]' AND membership = 'ban' ORDER BY room_id;
            room_id             |                   event_id                   
--------------------------------+----------------------------------------------
 !DYgXKez                       | $155230940352810bq
 !FPUfgzX                       | $155230952553390RE
 !FyQrGcO                       | $155230939752786Dw
 !HsxjoYR                       | $1549238649420087N
 !IgoVCuQ                       | $155230939652780al
 !QtykxKo                       | $155230940052801ay
 !TjpQZyM                       | $155230940752833jM
 !UcYsUzy                       | $1516043593504592t
 !YkZelGR                       | $155230940652828cg
 !boLskYi                       | $155230939852796un
 !fILrmqI                       | $155230940552821TI
 !iNmaIQE                       | $155230946053143QT
 !lnjwXgU                       | $155230939652782pG
           [REDACTED]           |     [REDACTED]
 !svJUttH                       | $155230939552776uN
 !uewiild                       | $155230939852792uK
 !yZHTGeD                       | $155230952153377bo
           [REDACTED]           |     [REDACTED]
(18 rows)
```


#### Matching with the data extract

Column name meanings:

- **Localpart**: The room `localpart` matching with the SQL extract above.
- **Provided:** Was the room provided in the dump
- **Others:** Was there private data (Identifiers, avatar URLs, etc) from other people in the extract, **which did not contain personal data of The Individual.**
- **Latest Extract**: `localpart` of the latest event for the room, ordered by `origin_server_ts` within the extract.
- **ACL Bypass**: After comparing the latest event seen by The Individual Homeserver and the latest event in the extract, were there newer events that are not visible to The Individual using the regular access controls of the system.

Columns with values in **BOLD CAPITALS** are problematic values.
The two redacted rooms did not have any users from `matrix.org` and therefore are out of scope of this analysis.

| **Localpart**   | **Provided** | **Others** | **Latest Extract** | **ACL Bypass** |
| -------------------- | ------------------------- | -------------------------- | ------------------------------ | ---------------------------------- |
| `!DYgXKez` | Yes                       | **YES**                    | `$155230940352810bq`         | No                                 |
| `!FPUfgzX` | Yes                       | **YES**                    | `$15623199283546iR`          | **YES**                            |
| `!FyQrGcO` | Yes                       | **YES**                    | `$15633589224733BV`          | **YES**                            |
| `!HsxjoYR` | Yes                       | **YES**                    | `$1556532989627Ge`           | **YES**                            |
| `!IgoVCuQ` | Yes                       | **YES**                    | `$155230939652780al`         | No                                 |
| `!QtykxKo` | Yes                       | **YES**                    | `$1561475801557le`           | **YES**                            |
| `!TjpQZyM` | Yes                       | **YES**                    | `$155230940752833jM`         | No                                 |
| `!UcYsUzy` | Yes                       | **YES**                    | `$153651141455EO`            | **YES**                            |
| `!YkZelGR` | Yes                       | **YES**                    | `$155230940652828cg`         | No                                 |
| `!boLskYi` | Yes                       | **YES**                    | `$155230939852796un`           | No                                 |
| `!fILrmqI` | Yes                       | **YES**                    | `$155230940552821TI`           | No                                 |
| `!iNmaIQE` | Yes                       | **YES**                    | `$15550147934913kF`            | **YES**                            |
| `!lnjwXgU` | Yes                       | **YES**                    | `$155230939652782pG`           | No                                 |
| `!svJUttH` | Yes                       | **YES**                    | `$1554994209991wL`             | **YES**                            |
| `!uewiild` | Yes                       | **YES**                    | `$155230939852792uK`           | No                                 |
| `!yZHTGeD` | Yes                       | **YES**                    | `$155230952153377bo`           | No                                 |

