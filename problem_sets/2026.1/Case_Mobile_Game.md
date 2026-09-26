# Case Study: Mobile Game

**CSCI 112 - Contemporary Databases**
**Midterm, First Semester SY 2026-2027**

## Background

Bakunawa Games is a Cebu-based studio that publishes *Lakan Arena*, a 5v5 mobile
battle arena game. Two teams of five, each player controlling one of 120 heroes,
fight for about 15 minutes. The game has
22 million monthly players and 6 million daily players across Southeast Asia.
Players are concentrated in a few regions: the Philippines has 45% of them,
Indonesia 30%, and Malaysia, Singapore, Thailand, and Vietnam share the rest.

Players finish about 7 million matches a day. Traffic is uneven through the day.
Between 7:00 PM and 11:00 PM local time, concurrent players peak at 1.8 million,
and about 350 matches end every second. Matches never stop, though: at 4:00 AM in
Manila, about 20 matches still end every second.

The game has kept every player's full match history since launch five years ago.
That history is now over 50 TB and grows by about 30 GB a day. Some veteran
players have more than 40,000 matches. No single server can hold this data or
absorb the evening write load, and the player base grows by about 3% a month.
Bakunawa needs a system that can handle more players by adding servers, without
downtime and without one server doing most of the work while the others sit idle.

## How the Business Works

Every player has a **rank**: a tier (Tanso, Pilak, Ginto, Diyamante, Bituin, and
Alamat) and a number of **rank points** within that tier. Winning a ranked match
gains points and losing one costs points. Reaching a tier's threshold promotes the player, and falling below
zero demotes them. Tiers unlock rewards, such as skins and currency, so players
check their points closely and file support tickets when a point goes missing.

When a match ends, the game server sends the result: the ten players, their teams
and heroes, the winner, the duration, and each player's kills, deaths, assists,
gold, damage, and items. Every player's rank points and stats must be updated from
that result. Game servers retry when the network drops, so the same result
sometimes arrives two or three times. A match must count exactly once.

A player's **profile screen** shows their name, rank tier and points, win rate,
favorite heroes, total matches, and their last 10 matches. Players open their own
profile after almost every match, and open their teammates' and opponents'
profiles in the lobby. It is the most-viewed screen in the game.

Each region has a **leaderboard** of its top 100 ranked players. Each hero also
has a top-100 leaderboard per region, ranked by a hero score earned by playing
that hero in ranked matches.

Players can also open any past match to see its full summary and scroll through
their full match history.

## Access Patterns

| # | Access pattern | Who | Volume / frequency | How current / exact |
|---|---|---|---|---|
| AP1 | View a player's profile: rank, points, stats summary, last 10 matches | Players (own and others') | 90,000/s at peak; 8,000/s off-peak | Rank and points must be exact. The stats summary may be up to 1 minute behind. The last 10 matches must include a match within 10 seconds of it ending. |
| AP2 | Record a finished match for all 10 players, and show each player their rank tier and points change right after ("+18, promoted to Bituin") | Game servers (record); players (view change) | 350 matches/s at peak (3,500 player results/s); 20/s at the lowest | **Exact.** Every player's points change exactly once per match, even when the result arrives more than once. The change shown must match the points used for promotions and rewards. |
| AP3 | View a leaderboard: the regional top 100, or the top 100 for one hero in one region | Players | Regional: 15,000/s at peak. Hero: 6,000/s at peak, spread over 120 heroes; the 10 most popular heroes take half | May be up to 5 minutes behind |
| AP4 | View one match's full summary | Players | 4,000/s at peak | Exact. A match result never changes once recorded. |
| AP5 | Scroll a player's full match history, 20 matches per page, newest first | Players, support staff | 800/s; most views stop at page 3, but some players page back years | Exact |

## Existing Data

Today, each player is one document. Their current rank tier and rank points sit
at the top, and every match they have ever played is appended to a `matches`
array inside it. Each entry is the full match result, all ten players' stats
included, so the same match is copied into ten player documents. When a result
arrives, the game server pushes the match into each of the ten players' arrays
and adds the points change to each player's rank points. Matches recorded before
a 2023 client update store the duration as `"mm:ss"` text and the stat line as a
single `"K/D/A"` string; newer entries store numbers.

No leaderboard is stored. Opening one sorts every player in the region by tier
and points; a hero leaderboard scans players' match arrays for that hero. Win
rate, favorite heroes, and total matches are computed from the array on every
profile view.

Veteran players' profiles take several seconds to open, leaderboards time out in
the evening, and support receives tickets about points counted twice. Two
samples follow, shortened.

```json
{
  "_id": "P-0000417",
  "name": "BagyoNiLolo",
  "region": "PH",
  "tier": "Bituin",
  "rank_points": 64,
  "matches": [
    {"match_id": "M-2022-0081533", "ended": "2022-03-14T21:40:00+08:00",
     "ranked": true, "duration": "14:52", "winner": "blue",
     "players": [
       {"player_id": "P-0000417", "team": "blue", "hero": "Sidapa",
        "kda": "7/2/11", "gold": 11840, "damage": 64210},
       {"player_id": "P-0391102", "team": "red", "hero": "Manananggal",
        "kda": "3/6/4", "gold": 8120, "damage": 40115}
     ],
     "points_change": 17}
  ]
}
```

```json
{"match_id": "M-2026-9920411", "ended": "2026-09-19T20:13:00+08:00",
 "ranked": true, "duration_s": 931, "winner": "red",
 "players": [
   {"player_id": "P-0000417", "team": "blue", "hero": "Sidapa",
    "kills": 4, "deaths": 5, "assists": 6, "gold": 9310, "damage": 51002,
    "items": ["Kampilan", "Anting-Anting", "Bakya"]},
   {"player_id": "P-2210093", "team": "red", "hero": "Tikbalang",
    "kills": 9, "deaths": 1, "assists": 7, "gold": 12604, "damage": 70331,
    "items": ["Kalasag", "Bolo", "Bakya"]}
 ],
 "points_change": -15}
```

The first player has 41,207 entries in the array; one is shown. The second sample
is a newer entry from the same array. The player on the red team is new: their
array holds that match twice, and their points rose by 36 instead of 18.

## Change Request

Six months after the migration, Bakunawa introduces **ranked seasons**. Each season runs about three months.
When a season ends, every ranked player's rank is soft-reset: each player drops a
set number of tiers (for example, Alamat drops to Diyamante) and rank points go
back to zero. The reset must be done for all players during a two-hour
maintenance window.

Players must still see their final rank and season badge from last season on
their profile. A separate "Career" screen, opened rarely, lists the final rank of
every season a player has played. Season-end rewards are based on each player's
final rank and must be exact. From the first day of a new season, the regional
and hero leaderboards show that season only.

None of the existing player ranks or match records have season information.
Everything recorded before this change belongs to "Pre-Season."

## Implementation Task

Write Python code using PyMongo that demonstrates the following.

1. **Insert mock data.** Insert mock data in the **existing** shape shown above:
   15–20 players across 4 regions (most of them in two regions), about 5 heroes,
   and 20–30 match records, each copied into its ten players' arrays. One veteran
   player must have about 20 matches and one new player 2–3. Include both the old
   and new entry formats, and at least one match that appears twice in a player's
   array.

2. **Migrate.** Move the data to your model with an aggregation pipeline. Print
   the document count per collection before and after, and one or two documents
   before and after the migration. Existing ranks and matches become Pre-Season.

3. **Recording, profiles, and leaderboards.** Write the code that records a
   finished match for all 10 players. Record a new match that includes the
   veteran and the new player. For both, print the profile before and after:
   rank, rank points, and the last 10 matches. Send the same match result a
   second time and show that no player's points or match count changed. Print
   the size in bytes of the data read to show each profile. Retrieve page 1 and
   the oldest page of the veteran's history (use 5 matches per page for this
   demo). Then produce the leaderboard for one region and for one hero in that
   region, print the first 10 entries of each, and state how old the data behind
   each leaderboard can be in your model.

4. **Access pattern queries.** Run the queries for your top-ranked access patterns
   (at least AP1, AP2, AP3, and AP5, plus any others you ranked above them). For
   writes, show the state before and after. Include `explain()` output or the
   indexes each query uses.

In section 2 of your design paper, also explain how player and match data would
be divided across multiple servers as Bakunawa grows: what decides which server a
player's or a match's data lives on, and how your plan holds up during the evening
peak and given that two regions hold most of the players. You do not need to run
multiple servers.
