# An adventure in Datalog

Last year I stumbled on a video by Mihai Budiu about [DBSP](https://www.youtube.com/watch?v=eYJA-ZBs-KM) . I would recommend watching the video but the tldr is that Prof Budiu and others have created a theoretical framework which describes how to build incremental materialized views for databases. 
The research paper is available [here](https://mihaibudiu.github.io/work/budiu-vldb23.pdf)

I find materialized views very compelling because they have several applications, for example preemptively computing expensive queries on the server side and not having to rely on database index performance or using incremental updates to views for triggering business logic.

So far DBSP as mostly been applied to SQL databases, I think it could be a great fit for Datomic like databases because of the simplicity of the query syntax. The transaction log and eavt index readily provides all the deltas needed to implement a DBSP circuit.

I think materialized views can also be applied on the client side. State management is a hard problem regardless of the domain, there are multiple tools like Reframe, Redux which help to synchronize state from the server and also manage complex state transitions in the UI. In my opinion there are two main orthogonal problems that need to be solved for state management on the client side.

- Structuring state in a way that is easily navigable
- Performing efficient updates on application state so that the UI can be updated accordingly

I believe the first one is solved by simply using a database, in the clojure world we have Datascript which can run in the browser as well as in a JVM process.
The second one could perhaps be handled by incremental views, so I set out to validate the idea.

[Wizard](https://github.com/jumraiya/wizard) is an experimental Clojure(script) library which maintains incremental views of datalog queries in memory.
The logic in Wizard is based on ideas described in the DBSP paper but not exactly 1 to 1, for instance the ZSets don't have an associated count, instead just a true/false value indicating an assertion or a retraction. This means the queries in Wizard can only return sets.
Each view can also be attached to functions to so that everytime the results of the view change, the relevant functions get a list of assertions, retractions and the accumulated state.

In order to see what that looks like let's build an application with it, we could do something boring like a todo app but let's do something more interesting, a text based escape room.

I am not a text adventure expert so forgive me if the following breaks any cardinal rules of text adventures.

Let's describe the game first

The player is locked in a house with multiple rooms, the player has to solve puzzles, unlock doors to escape the house.
There are several actions the player can take:

- Move in four directions (north, south, east, west). For simplicity's sake we assume that the player is ALWAYS facing due north. The player cannot move within a room only between rooms.

- Inspect objects

- Pickup objects

- Interact with objects


So how do we model this world?

First we need to describe the house itself. We do that by creating entities for each room called `location`
e.g.
```clojure
{:db/ident :my-room
 :location/description "This is some room"}
```

then we need to model where the rooms are in relation to each other, since the only way the player can move between rooms is through doors it makes sense to model them as entities.

                    ┌────────────────────────┐
                    │                        │
                    │                        │
                    │                        │
                    │     :my-room           │
                    │                        │
                    │                        │
                    │                        │
                    │                        │
                    │            │           │
┌───────────────────┐────────────┼───────────┐
│                   │            │           │
│                   │                        │
│                   │                        │
│                   │                        │
│     :third-room  ─┼──   :another-room      │
│                   │                        │
│                   │                        │
│                   │                        │
│                   │                        │
└───────────────────┘────────────────────────┘

e.g.
```clojure                                          
{:exit/location-1 :my-room               
 :exit/location-1-wall :south
 :exit/location-2 :another-room
 :exit/location-2-wall :north
 :exit/locked? false}

{:exit/location-1 :my-room
 :exit/location-1-wall :west
 :exit/location-2 :third-room
 :exit/location-2-wall :east
 :exit/locked? false}
```
here location-* is a ref to the location entity above and location-*-wall which wall it's situated (when facing due north).

Based on the above two entity types we can determine when a player can move to a different room.
So for instance if the player is in `:my-room` then the only valid moves are to the south and west. Similarly if the player is in `:another-room` then they can only move north.

Finally we need to model objects present in the rooms. The player itself is also represented as an object entity

```clojure
{:object/description "object"
 :object/detailed-description "Detailed description"
 :object/location :my-room}

{:object/description "player"
 :db/ident :player
 :object/location :my-room}
```

If the player picks up something, it is means the object's location is the player i.e. `[:db/add object-id :object/location :player]`



