---
name: Appendix
---
# Appendix

> The Delta API should be considered as a preview. Some stuff in it will be changed before its final release. You can use it if you want, but we do not recommend it yet. Once the delta API is complete, we'll state a list of the breaking changes so you can adapt your code to work with the final v2 version

This file contains further information explaining additional stuff on the Delta API not explained in the [main document](v2).

If you'd want to see something implemented in the API, you can open an issue at [this GitHub repository](https://github.com/osuripple/api-features), and we'll look into it.

<!-- toc -->

* [Rate limiting](#rate-limiting)
* [Stability](#stability)
* [Authorization](#authorization)
* [Privileges](#privileges)
* [API Identifiers](#api-identifiers)
* [Multiplayer match IDs](#multiplayer-match-ids)
* [Automatic multiplayer matches disposing](#automatic-multiplayer-matches-disposing)
* [Yellow Notifications](#yellow-notifications)
* [Parameters](#parameters)
* [Pretty printing](#parameters)
* [Response format](#response-format)
* [Arrays](#arrays)
* [Boolean strings](#boolean-strings)
* [Usernames](#usernames)
* Enums
  * [Modes IDs](#modes-ids)
  * [Client types](#client-types)
      * [A note on Fake clients](#†%3A-a-note-on-fake-clients)
  * [Actions](#actions)
  * [Bancho Privileges](#bancho-privileges)
  * [Multiplayer Scoring Types](#multiplayer-scoring-types)
  * [Multiplayer Team Types](#multiplayer-team-types)
  * [Multiplayer Teams](#multiplayer-teams)
  * [Multiplayer Slot Statuses](#multiplayer-slot-statuses)

<!-- tocstop -->

## Rate limiting

TODO

## Stability

This API release is still in major version 0. Please note that none of the following is final, and subject to change while we are still in version 0.

## Authorization

You can use a Ripple API Token to identify yourself to the API. You can get a Ripple API Token from the following URL: https://ripple.moe/dev/tokens.

When required, the token can be passed (in order of priority):

* With the HTTP header `X-Ripple-Token`
* With the querystring parameter `token`
* With the querystring parameter `k`

If you're using an OAuth bearer token, please use the the `Authorization` header
and prefix it with `Bearer `.

## Privileges

The privileges used by the Delta API are the same as the ones used by the Ripple API. Please refer to [this section of the Ripple API documentation appendix](../api/appendix#privileges) for more information.

## API Identifiers
In the Delta API, there's the concept of API Identifiers: strings identifiying clients in a unique way. They're largely used in the `clients` and `multiplayer` API handlers. They are in the format `@uuid`. An example of an API identifier may be `@0f23d692-ee91-4a68-af68-080bcfdc1dcf`.  
Note that **the API Identifiers are not the tokens used by the Bancho Protocol to authenticate game clients**, they're different and API Identifiers can only used in the Delta API. API Identifiers are needed because a user may be connected from multiple clients at the same time (eg: two IRC clients and one game client), and sometimes you may want to do something on a specific client, not on all clients that belong to a specific user.

## Multiplayer match IDs
A "Multiplayer Match" is a multiplayer "room". Each match is identified in the API by an ID (also called "Internal ID" in this documentation). Multiplayer match IDs go from 1 to 2147483647. The counter is reset each time the bancho server emulator is restarted. You can use this ID to manipulate matches through the Delta API (see the [Multiplayer](v2#multiplayer) section). When the first game of a multiplayer room is completed, the match is stored permanently in [vinse](https://vinse.ripple.moe), and a new ID, the vinse ID, is generated. The vinse ID is available through the Delta API as well, but it cannot be used to identify a match through the Delta API.   
For the courious out there, the vinse ID is calculated like this:  
```python
(u // (60 * 15)) << 32 | id
```
Where:

  * `u` is the UNIX timestamp, in seconds, of when the first game in that match was completed;
  * `id` is the "Delta API" (also known as "internal") match id, the one the one relative to the counter that is reset when the server restarts.

However, you don't need to care about the Vinse ID, since the Ripple stack takes care of generating it. _It's only used by vinse to keep track of multiplayer matches for their match history, even when the server restarts._

## Automatic multiplayer matches disposing
Multiplayer matches normally get disposed when all players in it leave the multiplayer match. Matches created through the API (with [POST /multiplayer](v2#post-%2Fmultiplayer)), however, will not act like that. These kind of matches can only be disposed:

  - When manually calling the [DELETE /multiplayer/{id}](v2#delete-%2Fmultiplayer%2F{id}) API handler
  - When the Multiplayer Manager flags the match as inactive. This happens when the match is empty and there are no changes to the match for too long.

However, sometimes you may want to keep an API match alive even when it's been empty for a long time. In order to achieve this, you can periodically call the [GET /multiplayer/{id}](v2#get-%2Fmultiplayer%2F{id}) API handler, which will mark the match as active. Calling that handler once per minute is enough to keep the match alive. Having some interaction in the match (eg: a user joins) or calling any other multiplayer-related API handler will mark that match as alive as well.

## Yellow notifications
There are some API handlers that refer to "yellow notifications". The way these notifications are displayed varies based on the type of the client:

- Game clients display "yellow notifications" as (you guessed it) yellow messages in the bottom right corner of the screen
- IRC clients will see them as messages by FokaBot
- Web socket clients will receive them as [server_announce](websocket#server_announce-(server)) messages

---

![](/docs/images/rollsrois.jpg)  
_A yellow notification on a game client_

---

## Parameters

In GET requests, all parameters are passed through the querystring, while in POST they are passed through a JSON-encoded request body.

## Pretty printing
You can pass the `pretty=1` GET parameter to pretty-print the JSON response:
```
$ curl http://c.ripple.moe/api/v2/ping
{"code":200,"version":"19.0.0"}

$ curl http://c.ripple.moe/api/v2/ping?pretty=1
{
    "code":200,
    "version":"19.0.0"
}
```

## Response format
The HTTP response will always have `Content-Type: application/json; charset=utf-8` and thus it'll always be a JSON object. A `code` field is in **every** response (even when not specified) and it coincides with the HTTP response code. When an error occurrs or when no "response" section is specified in the documentation of a specifid API handler, a `message` field will be present as well, describing the outcome of the request.

The HTTP response codes will always be the same as the internal `code` of the response, if any.

* `404` is used, apart from when a method is missing, also when a specified resource is not found.
* Other used response codes are: TODO (check each api handler for now)

## Arrays

Unlike the Ripple API, the Delta API returns empty JSON arrays (`[]`) instead of `null` when an empty list should be returned.

## Boolean strings
Sometimes, a `bool` field is required in a GET parameter. This may arise a bit of confusion, because all parameters in a GET request are strings. `bool` parameters in GET requests are interpreted as follows:

* false ⇔ `'false'`, `'0'` (case insensitive)
* true ⇔ all other values

tldr: You can use either `'true'`/`'false'` or `'1'`/`'0'`. Note that this is valid only for GET requests. When a `bool` field is required in a JSON field, you must use `true` and `false` (as booleans, not strings!).

## Usernames
There are two format of usernames on the Ripple stack: safe usernames and non-safe usernames.

* **Non-safe usernames**: the exact username the user has chosen when signing up on Ripple.
* **Safe usernames**: a version of the non-safe username that is obtained by:
    * Making all letters lowercase
    * Replacing all spaces with underscores

    Safe usernames were introduced when IRC support was first added to ripple.
    
    The uniqueness of a username is determined by its safe counterpart, meaning that two users whose safe username is the same, can't exist. Eg: Albeit `- Zino -` and `-_ZiNO -` are different usernames, their safe username is the same (`-_zino_-`), so they refer to the same user.

<!--
## Pagination

Pagination is very common in the API, and can be used to get a specific amount of elements, or get all of them in different "chunks". It follows the same pattern: querystring parameters `p` and `l`, which stand for page and limit (possibly one of the few abbreviated querystring parameters in the API). If a request implements pagination, `Implements pagination.` will be written on the description of a request. If the limit maximum is not 50, it will be specified (``Implements pagination (1 < x <= 100)``, in case there's a maximum of 100).

## Time

Time is a tool you can put on the wall, or wear it on your wrist. And, apart from that, it's also that stupid thing humans use to calculate in which part of the day they are.

Time is passed as JSON strings, and formatted using RFC3339. This makes it super-easy to translate times into your programming language's native time, for instance in JavaScript:

```js
> new Date("2016-10-28T21:10:55+02:00")
Fri Oct 28 2016 21:10:55 GMT+0200 (CEST)
```


## IN Parameters

These have a peculiarity. They can be passed multiple times in a query string. What it basically means is that you can look for values being "in a certain group". If you pass in the querystring: `ids=1000&ids=1001&ids=1002`, you will get results for IDs 1000, 1001 and 1002.

## Sorting

The API allows sorting elements. To do so, you will need to pass the parameter `sort`, with the value being the field being sorted, a comma and then asc/desc. By default everything is sorted desc. For instance, `sort=id,desc` will sort by `id` descendently, and also `sort=id` will. When there's a sorting section in an endpoint, the fields that can be sorted will be specified.

## Play style

Play style sometimes appears in the stats of an user. The bitwise enum for it can be found [Here](https://git.zxq.co/ripple/playstyle/src/master/playstyle.go#L11-L21). Must be read as explained in [Privileges](#privileges)
-->

## Enums
These are the enum-like fields used by the Delta API.

### Modes IDs
Just like on the Ripple API:

Game Mode     | Value
--------------|----------
osu!standard | 0
osu!taiko | 1
osu!catch | 2
osu!mania | 3

### Client types

Client type    | Value
---------------|----------
Game | 0
IRC | 1
Fake (†) | 2
Web socket | 3

#### †: A note on fake clients
Fake clients are dummy clients used by the server to send chat messages when something occurs. The only instance where fake clients are used, is when the server sends messages as FokaBot on its own. Most FokaBot messages are sent by a Web socket client (which is controlled by the actual FokaBot chat bot software), but a few FokaBot messages, like the "match history available here" message sent after the first match in a multiplayer match, are sent directly by the server with a fake FokaBot client._
TODO: List all instances where the fake client is used.

### Actions

Action     | Value
-----------|----------
Idle | 0
Afk | 1
Playing | 2
Editing a beatmap| 3
Modding a beatmap| 4
In a multiplayer match | 5
Watching | 6
~~UNUSED~~ | ~~7~~
Testing a beatmap | 8
Submitting a beatmap | 9
Paused | 10
In multiplayer lobby | 11
Playing in a multiplayer match | 12
Using osu!direct | 13

### Bancho Privileges

Privilege     | Value | Username colour
--------------|-------|----------------
Normal | 1 | Pale yellow
Community Manager/Chat Mod | 2 | Red
Donor | 4 | Bright yellow
Developer | 8 | Blue
Tournament Staff | 16 |

* _Please note that these can be combined. Eg: `20 <=> 16 | 4` is a Tournament Staff who has Donor._
* _Username color priority: `Blue > Red > Bright Yellow > Pale Yellow`_
* _Also note that the Pink name that "BanchoBot" has on the official server is hardcoded in the client, based on the username, so it's not possible to give any other user a pink name, unless their username is "BanchoBot"_


### Multiplayer scoring types

Scoring type     | Value
-----------------|----------
Score | 0
Accuracy | 1
Combo | 2
Score v2 | 3

### Multiplayer team types

Team type     | Value
--------------|----------
Head to head | 0
Tag coop | 1
Team vs | 2
Tag Team vs | 3

### Multiplayer teams

Team      | Value
----------|----------
No team | 0
Red | 1
Blue | 2

TODO: check this as it doesn't make any sense


* _Please note that `No team (0)` is used only for empty slots or if the team type is "Head to Head" or "Tag Coop" (which have no teams). When the team type is "Team vs" or "Tag Team Vs", all slots (including empty ones) will be assigned to a team._


### Multiplayer slot statuses

Team      | Value
----------|----------
Open | 1
Locked | 2
Not ready | 4
Ready | 8
No map | 16
Playing | 32
Complete | 64
Quit | 128
