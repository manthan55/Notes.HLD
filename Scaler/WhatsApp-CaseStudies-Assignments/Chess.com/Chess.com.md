# Problem Description
Chess.com allows people to play chess with random people, friends & computer opponent. One of the key features of the site is that it allows spectating games. People can follow their favorite players & can tune into their live games as spectators.

I want to focus on this game spectating feature.

---
# Functional Requirements / MVP

### MVP Features
I think the MVP features are as follows
- Create a game
- Play the game
	- be able to send commands & board should be updated for both players
	- order of moves must be maintained
- Spectate a live game
- Spectate an older (non-live) game - move-by-move

There are other features like
- customizing game while creation
	- time between moves, preset position, etc.
- Player matching based on rating

which I don't think are really necessary for MVP

### API's needed
<u>**createGame()**</u>
```
createGame(player_1_id, player_2_id, <options>): gameId
```
Here, `<options>` can be for future features like game customization options seen above.

<u>**sendMove()**</u>
```
sendMove(game_id, move): success/failure
```
`move` can be "E3", "D5", etc.

<u>**subscribeGame()**</u>
```
subscribeGame(game_id)
```
This API is for players to subscribe to the moves happening in the current game they are part of.

<u>**spectateGame()**</u>
```
spectateGame(game_id, <options>)
```
This API will return all the moves made in the game. If its called for an older game, all the moves are returned. If its called for a live game, all the moves made so far in the game are returned. For live games, we can make use of `<options>` for paginating moves based on timestamps.

---
# Non-Func Requirements / Design Goals
### PACELC Theorem
There can definitely be partitions as this is distributed system - so we have to account for P in our design.
##### What does consistency mean here?
Consistency here means all moves are delivered to opponent player immediately. Eventual consistency means the moves can be delivered out of order or some moves might even not be delivered. So eventual consistency will not work in this case and we need immediate consistency.

Now that we have decided we want Consistency, we can see how low a latency we can get. Low latency is important as players are playing live and lower-latency would mean more smoother gameplay.

| Options                     | Choice                                                         |
| --------------------------- | -------------------------------------------------------------- |
| Consistency vs Availability | Consistency                                                    |
| Consistency vs Latency      | As low latency as possible without compromising on consistency |

---
# Scale Estimation
### Data Storage (Is sharding needed?)
#### users
| Property | Size (bytes) |
| -------- | ------------ |
| id       | 16           |
| username | 20           |
| email    | 20           |
| pw       | 20           |
|          | ~100         |

- \#no of players = `150 M`
- \#size for each user = `100 bytes`
- \#size for all users = `15 GB`

Users data can be stored on a single machine - **No Sharding**. Not that users data does not include the games they played - this will be stored in other DB.

#### games
We can represent a move object like follows

| Property  | Size (bytes) |
| --------- | ------------ |
| player_id | 16           |
| move      | 2            |
|           | ~20          |

move can be "E6", "D3", etc -- only 2 characters. On average, a game of chess can have `40~50` moves. Then the game can be represented as

| Property    | Size (bytes) |
| ----------- | ------------ |
| id          | 16           |
| player_1_id | 16           |
| player_2_id | 16           |
| moves       | [moves x50 20bytes]            |
|             | ~1100        |

On average a user plays 3 games per day. Heavy users might play more but this can be offset by light users.

- \#no of games/player/day = `3`
- \#no of games/day = `3 * 150M` = `300M`
- size of one game = `~1MB`
- size of `300M` games = `300M * 1MB` = `300TB`

So chess.com generates `300TB` of data per day. If we assume chess.com has been up since 10 years then,

- size of all games in a month = `300TB * 30days` = `9PB` --> round off to `10PB` for easier calc
- size of all games in a year = `10PB * 12months` = `~120PB`
- size of all games past 10 years = `120PB * 10years` = `1200PB`

This amount of data cannot fit on a single machine - **need Sharding**. I prefer to shard by `gameId` as it promotes even distribution, good cardinality and is usually part of request when calling API's like `sendMove()`, `spectate_()` & `subscribeGame()`

### QPS Calculation
<u>**createGame()**</u>
We estimated the total \#no games played per day are `300M`.
- \#no of `createGame()` calls/day = `300M`
- \#no of `createGame()` calls/sec = `300M / 86400s`

assuming `10000s` in a day for easier calc
- \#no of `createGame()` calls/sec = `300M / 10000s` = `30K`

<u>**sendMove()**</u>
We estimated that each game can have `40~50` moves.
- \#no of `sendMove()` calls/game = `50`
- \#no of `sendMove()` calls/day = `300M * 50` = `15B`
- \#no of `sendMove()` calls/sec = `15B / 10000` = `1.5M`

<u>**spectateGame()**</u>
This API will be hit quite significantly during peak times. I want to assume out of `150M` players on chess.com, `50M` players would spectate the live tournament games.

##### Peak Load Analysis
Chess.com hosts tournaments among the worlds best players. During such tournaments a lot of players spectate live games and thus the no of calls to `spectateGame()` increase by a lot.

##### Read/Write/both Heavy
I think its fair to assume that system is both read & write heavy. Moreover, during peak time (tournaments), the system becomes more read heavy as a lot of people want to spectate games played by the top players. 

# System Design
#### How will we store the data?
We are mostly interested in sequence of moves made in a particular game. We want to datastore to be able to store the moves in the order in which they are made. I think Wide Column DB like Cassandra would be a good fit here. We can have a column for moves.

#### How will live spectating work?
Spectators can establish a 2-way connection with server and the moves can be pushed as notifications as-and-when they happen.

Alternatively, we can have the clients periodically poll the server which can then return the moves from a distributed cache like redis. This will help reduce the amount of servers needed to maintain the 2-way connections. Also its not really important for spectators to see the moves in real-time. There can be some delay for the caches to update which is fine.

![[system_design.svg]]