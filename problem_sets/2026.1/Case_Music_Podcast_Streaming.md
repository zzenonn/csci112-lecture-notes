# Case Study: Music and Podcast Streaming

**CSCI 112 - Contemporary Databases**
**Midterm, First Semester SY 2026-2027**

## Background

Tugtog is a Philippine streaming service for songs and podcasts. Its catalogue
holds about 6 million songs, both OPM and international, and about 1.2 million
podcast episodes from 40,000 shows. Tugtog has 9 million monthly listeners, and
about 2.5 million of them listen on a typical day. Listeners use the free,
ad-supported tier or Premium at ₱149 a month.

Every month, Tugtog pays rights holders (labels, independent artists, and podcast
networks) based on how often their content was played. Payouts total about
₱180 million a month. A wrong count means overpaying or a dispute with a label.

## How the Business Works

Songs and podcast episodes share basic details: a title, one or more creators,
a duration, artwork, a language, and a genre or category. They also differ. A
song belongs to an album, has a track number, and may list featured artists and
songwriters. An episode belongs to a show, has a season and an episode number,
has a description, and may have a transcript. In the app, listeners search and
browse songs and episodes mixed together.

Each time a listener starts a song or an episode, the app reports a play. A play
is **qualified** if the listener stays for at least 30 seconds of a song or 60
seconds of an episode. Only qualified plays count toward payouts and charts. The
payout rate for a qualified play depends on the listener's country and tier
(free or Premium) at the time of the play.

Every song and episode page, and every row in a list, shows a public play count.
Tugtog publishes a Top 50 chart every morning at 6:00 AM (Manila time) for each
of its 12 countries. The charts cover all songs, each of 20 genres, and podcasts.

Creators use a dashboard to track their content. Monthly, finance issues each
rights holder a statement of qualified plays and amount owed.

Volumes: about 300 million plays a day, of which about 70% qualify. Plays
average 3,500 a second and peak at 20,000 a second between 7:00 and 11:00 PM.
A viral song (a TikTok trend, a surprise midnight OPM release) can receive 3,000 plays a second for several hours.

## Access Patterns

| # | Access pattern | Who | Volume / frequency | How current / exact |
|---|---|---|---|---|
| AP1 | Record a play: which listener played which song or episode, when, from which country and tier, and how long they listened | Listener app | 3,500/s average, 20,000/s peak; up to 3,000/s on one viral song | Every qualified play must be kept. None may be lost or counted twice. |
| AP2 | Show the public play count on a song or episode page and in every list row | Listeners | 150,000 reads/s at peak | May be a few minutes behind and off by up to 1% |
| AP3 | Show a daily Top 50 chart (country, plus all / a genre / podcasts), with rank, title, creator, artwork, and change from yesterday | Listeners, media | ~250 charts; 40,000 reads/s in the hour after 6:00 AM | Built once a day from the previous calendar day's qualified plays; must match those plays exactly once published |
| AP4 | Search and browse songs and episodes together by title or creator, filtered by type, genre, and language | Listeners | 8,000/s | New releases searchable within 15 minutes |
| AP5 | Creator dashboard: qualified plays per song or episode per day for the last 90 days, and top 10 countries | Creators, managers | 60,000 logins/day | Up to 1 hour behind |
| AP6 | Monthly royalty statement: qualified plays and amount owed per song or episode, per country, per tier, for each rights holder | Finance | Once a month for ~4,000 rights holders, finished within 3 days of month end | **Exact.** Must equal the actual qualified plays. On a label's request, finance must be able to reproduce the paid count for one song on any day in the last 12 months. |

The royalty statement (AP6) is contractual. A count that is "roughly right" is
not acceptable for it. The public count (AP2) is display only and is never used
for money.

## Existing Data

Today, songs and episodes live in two separate collections, `songs` and
`episodes`, built by two different teams. Shared details use different field
names and formats: a song has `title`, `artist` (one string), and
`duration_sec`; an episode has `episode_title`, a `show` with its `hosts`, and
`length` as text. Every play is written to a `plays` collection, and the app
also adds 1 to a counter on the song or episode for every play started. Finance
builds monthly statements from these counters, and labels have disputed several.
Each chart is built from all of the previous day's plays whenever it is opened,
and the chart page is slow every morning. Samples follow.

```json
{
  "_id": "S-000812",
  "title": "Maybe The Night",
  "artist": "Ben&Ben",
  "album": {"name": "Limasawa Street", "year": 2019},
  "track_no": 3,
  "duration_sec": 262,
  "genre": "OPM Pop",
  "lang": "fil",
  "art": "https://cdn.tugtog.ph/a/81203.jpg",
  "label": "Sony Music Philippines",
  "play_count": 48211093
}
```

```json
{
  "_id": "E-104417",
  "episode_title": "Bakit Mahal ang Bigas?",
  "show": {"name": "Usapang Piso", "hosts": ["Carla Reyes", "Jun Dela Cruz"],
           "network": "Kwento Media"},
  "season": 2,
  "ep": 14,
  "length": "00:52:14",
  "category": "Business",
  "language": "Filipino",
  "description": "Why rice prices keep rising, and who profits.",
  "plays": 91822
}
```

```json
{"_id": "P-9f31c2", "listener": "L-772104", "track": "S-000812",
 "kind": "song", "at": "2026-09-24T21:14:05+08:00", "country": "PH",
 "tier": "free", "listened_ms": 41200}
```

```json
{"_id": "P-9f31c3", "listener": "L-015530", "track": "E-104417",
 "kind": "episode", "at": "2026-09-24T21:14:06+08:00", "country": "SG",
 "tier": "premium", "listened_ms": 22000}
```

The counters include every play started, including plays of a few seconds.
When the app retries a play report after a network error, the counter can be
incremented twice.

## Change Request

Six months after the migration, Tugtog adds **video podcasts**. A video episode is an
episode that also has video in several resolutions (360p, 720p, 1080p). Each
resolution has its own file size and bitrate. The episode is divided into
chapters, each with a title, a start time, and a thumbnail. Listeners can watch
the video or play the episode as audio only. A play qualifies under the same
60-second rule either way, but finance wants the monthly statement to show
video plays and audio plays separately for video episodes.

None of the 1.2 million existing episodes has any video information, and no
existing play records whether it was watched or heard. These episodes, and the
new audio episodes that continue to arrive, do not change. Video episodes must
appear in search, browse, charts, and dashboards alongside everything
else. Tugtog cannot take the service offline for the change. Older app versions that do not know about
video must keep working and play these episodes as audio.

## Implementation Task

Using Python and PyMongo, build your model on your own MongoDB instance.

1. **Insert mock data.** Insert mock data in the existing shape shown above:
   about 15 songs and episodes (at least 10 songs and 4 episodes from 2 shows)
   across 4–5 creators, 3 rights holders, and several genres; 12–15 listeners
   on both tiers in at least 3 countries; and about 20 plays spread over two
   consecutive days, including some unqualified plays.

2. **Migrate.** Write an aggregation pipeline that moves the existing data into
   your model. Show the document counts per collection before and after, and
   one song and one episode before and after.

3. **Viral song and daily chart.**
   - Simulate about 10,000 plays of one song by many different listeners,
     generated by a loop as fast as your machine allows (these are not seed
     data). Mix qualified and unqualified plays, countries, and tiers. Show the
     public play count before and after the burst; how many plays were
     recorded and how many writes were made to the song's own record; the
     exact qualified play count for that song for that day by country and tier,
     with proof that it equals the qualified plays your loop generated; and how
     far the public count differs from the true count.
   - Write an aggregation pipeline that runs once a day, builds the Top 50 for
     a given day, country, and chart type (all songs, one genre, or podcasts)
     from that day's qualified plays, and stores the result so that AP3 is
     served without scanning plays. Run it for two consecutive days so the
     change in rank from yesterday appears. Show the stored chart and the query
     that reads it.

4. **Access patterns.** Run the queries for your top-ranked access patterns
   (at least your top four). Show their output, the before and after state for
   writes, and `explain()` output or the indexes used.
