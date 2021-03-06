---
layout: docs
title: Endpoints
---

# {{ page.title }}

## Generic endpoint

```
POST /api/v1/mpd
Content-Type: application/json

{ "command": "some mpd command" }
```

This is the endpoint that hands a command as is to MPD. The return is
tha answer of MPD converted to JSON. For a list of commands, see [mpds
protocol
specification](https://musicpd.org/doc/protocol/command_reference.html).

For convenience, there are special endpoints for some commands (see
below).

Responses from MPD are encoded in JSON wrapped in an outer
object. Here is an example for an idle answer:

```json
{
  "success": true,
  "type": "IdleAnswer",
  "result": {
    "changes": [
      "player"
    ]
  }
}
```

The outer object contains a field `success` which indicates whether
the response was an _OK_ from MPD. If MPD responded with an _ACK_ this
is false. If it is `true`, then there is a `result` field containing
the payload of the MPD response. If it is `false`, there is an `ack`
field that contains the JSON representation of MPD's ACK response. An
additional `type` field indicates which kind of response it is.

## Websocket

Using the endpoint `/api/v1/mpd` with a GET request allows to create a
websocket connection. This can be used to get notified about MPD
events and also to send commands (same json as above) to MPD using the
same connection. The websocket connection is directly connected to
MPD; the http server also uses only one connection to MPD for the
websocket.

When the websocket is opened, it responds with `IdleAnswer` events or
with answers as results to sending some commands through the
websocket. A json decoder can (for example) check the `type` field to
distinguish answer types.


## Special Endpoints

These are just convenient endpoints to execute mpd commands. Each one
corresponds to a single mpd command.

- `GET /api/v1/mpdspecial/search?tag=value&tag=value&range=0:2` This allows
to use the `search` and `find` command directly with an http GET
request. You can specify multiple tag/value pairs to search for and a
window range.
- `GET /api/v1/mpdspecial/list?listtype=value&genre=Classical&track=2`
  Executes MPD's `list` command. Here the `listtype` is literal and
  means the tag to list values for. Optionally multiple tag/value
  pairs can be specified for filtering.
- `GET /api/v1/mpdspecial/count?tag=value` Executes MPD's `count`
  command. There must be exactly one tag/value pair.
- `GET /api/v1/mpdspecial/currentsong` Executes the `currentsong` command.
- `GET /api/v1/mpdspecial/status` Executes the `status` command.
- `GET /api/v1/mpdspecial/playlistinfo?pos=2&range=1:3` Returns the current
  playlist or only a part of it if `pos` or `range` are specified. If
  both are given, then `pos` wins.
- `GET /api/v1/mpdspecial/listplaylists` Lists all stored playlists.


## Cover and Booklet files

There are endpoints that deliver cover art or booklet files for an
album. One takes a song file and tries to find the corresponding cover
image. The other takes an album name and looks up a file using MDP
`find` command and uses it to look up the cover or booklet file.

    GET /api/v1/cover/album?name=<albumName>[&size=<sz>]
    GET /api/v1/cover/file/<file-uri.flac>[?size=<sz>]
    GET /api/v1/booklet/album?name=<albumName>
    GET /api/v1/booklet/file/<file-uri.flac>

The cover or booklet is expected to be a file next to the songs in a
directory. Some names are tried (which can be
[configured](configuration.html)), while `cover.jpg` (and
`booklet.pdf`, respectively) is the default (it is tried first). For
this to work, you need to configure the same music directory as in
your MPD config file. The cover images and booklet files are delivered
directly from the file system and are not requested from MPD.

For cover art, a `size` parameter can be specified. If this is
present, the cover art is resized (if it exceeds `size`) and a squared
thumbnail of the given size is returned. The thumbnail is created on
the fly and stored on the file system, so it might take a while when
requesting for the first time. Keep in mind that this slows down the
library page a lot on first access. But once all the thumbnails are
created, they are then served from the file system and everything goes
pretty fast. If you don't like this feature, you can turn it off
“globally” in the [configuration file](configuration.html); the `size`
parameter is then ignored and all clients receive the original cover
art file. If an error occurs during resizing, it is logged and the
original file is returned. Please see the [config
file](configuration.html) for how to configure this feature. Also note
that the directory of thumbnails is not cleaned up automatically.

If you have your cover art only inside each song, then currently that
is not supported. You could simply export all of them into its own
file; for example for flac files:

```bash
find /music/dir -name "*.flac" -printf "%h\n" | uniq | while read f;
do
  cover="$f/cover.jpg"
  flac=$(ls -1 $f/*.flac | head -n1)
  if ! [ -r "$cover" ]; then
    echo "$f"
    metaflac --export-picture-to="$cover" "$flac"
  fi
done
```

Note that this script assumes that every song in a folder belongs to
the same album and shares the same cover art!


### Missing Covers

A missing cover is replaced by a generated image from
[robohash](https://robohash.org) using the album name as input. This
way you get unique images per album.


### Cache

When looking for cover art for an album, one song of that album is
requested from mpd and then its directory is searched for a cover
file. Since this always requires a call to mpd, the final paths to a
cover file are cached in memory. The assumption is, that these files
don't change (often). If they do, there is an endpoint to clear the
cache:

- `POST /api/v1/cover/clearcache`

So after startup the first load of all album covers takes a while; but
this should then speed up significantly.

Cover images are by default resized. This is faster when there are
high qualitiy cover art, since all resized images are cached once they
are created.

Booklets and covers share the same cache. So while there is the same
route for booklets (`/api/v1/booklet/clearcache`), they have an
identical effect.

## Multiple MPD connections

If multiple mpd connections are configured, they are available using
its id after the usual endpoint ends. For example, `/api/v1/mpd` would
be `/api/v1/mpd/myid`, or `/api/v1/mpdspecial/currentsong` would be
`/api/v1/mpdspecial/myid/currentsong`. The connection with id
`default` (which is mandatory) is always reachable at the “default”
endpoints; additionally to its concrete form. For example, both
endpoints:

- `/api/v1/mpd` and
- `/api/v1/mpd/default`

are the same.


## Custom data

If you like to serve some static content, it is possible by
configuring a custom endpoint. Simply specify a directory and
everything below that is served at the `/custom/` endpoint.
