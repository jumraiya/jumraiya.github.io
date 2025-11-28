# An escape room(s) in Datalog

Well this is more about state machines really but a game sounds more fun. I wrote a text based escape room in Clojurescript to test a library I have been working on, and wanted to share my experience. All of the game state is expressed as a Datascript database, all game actions are driven via Datalog queries. However, these queries are incremental and reactive, meaning every change in the result of a query is available after every transaction.

You can access the the game [here](https://jumraiya.github.io/posts/escape-room.html) .

Unfortunately the game is very slow to initialize and pretty short, I am still working on it.

There is a view on the bottom made with portal that shows how user actions trigger state transitions.

## Why Datalog?

- Datalog is declarative and closer to pure logic programming. Other relational algebra languages like SQL while feature rich are more imperative and verbose.

- It is composable, logic can be expressed as rules and easily reused across queries.

- It supports mutiple access patterns e.g. Column, Row, Graph


## Incremental materialized views

Last year I stumbled on a video by Mihai Budiu about [DBSP](https://www.youtube.com/watch?v=eYJA-ZBs-KM) . I would recommend watching the video but the tldr is that Prof Budiu and others have created a theoretical framework which describes how to build incremental materialized views for databases. 
The research paper is available [here](https://mihaibudiu.github.io/work/budiu-vldb23.pdf)

I find materialized views very compelling because they have several applications, for example 

- Preemptively computing expensive queries on the server side and not having to rely on database index performance. 

- Using incremental updates to views for building state machines.

So far DBSP as mostly been applied to SQL databases, I think it could be a great fit for Datomic like databases because of the simplicity of the query language. The transaction log and eavt index readily provides all the changesets needed to implement a DBSP circuit.

I think materialized views can also be applied on the client side. State management is a hard problem regardless of the domain, there are multiple tools like Reframe, Redux which help to synchronize state from the server and also manage complex state transitions in the UI. In my opinion there are two main orthogonal problems that need to be solved for state management on the client side.

- Structuring state in a way that is easily navigable
- Performing efficient updates on application state so that the UI can be updated accordingly

I believe the first one is solved by simply using a database, in the clojure world we have Datascript which can run in the browser as well as in a JVM process.
The second one could perhaps be handled by incremental views, so I set out to validate the idea.

The following assumes we can register Datalog queries whose results are incrementally updated on each transaction and we have access to the diff as well.

## The escape room(s)

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

```

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
```

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


So how does the game state evolve as the player executes actions? We describe player actions as entities and use datalog queries to determine if the action is valid or not

e.g. When a player attempts a move action
> move west

we transact an entity
```clojure
{:action/type :move
 :action/arg "west"}
```

We have a query that listens for that action and validates it.
```clojure
'[:find ?a ?dest ?locked
             :in $ %
             :where
			 
			 ;; Listen for actions that haven't been processed
             [?action :action/type :move]
             [?action :action/arg ?wall]
             (not-join [?action]
                 [?action :action/move-processed? true])

             ;; Find the player's current location
             [?p :object/description "player"]
             [?p :object/location ?loc]
			 
			 ;; Validate if the player can move based on current location
             (or-join [?loc ?wall ?dest ?locked]
                      (and
                       [?exit :exit/location-1 ?loc]
                       [?exit :exit/location-1-wall ?wall]
                       [?exit :exit/location-2 ?dest]
                       [?exit :exit/locked? ?locked])
                      (and
                       [?exit :exit/location-2 ?loc]
                       [?exit :exit/location-2-wall ?wall]
                       [?exit :exit/location-1 ?dest]
                       [?exit :exit/locked? ?locked])
					   
					   
				      ;; Fall through case, there is no door in that direction
                      (and
                       (not-join [?loc ?wall]
                                 (or-join [?loc ?wall]
                                          (and
                                           [?e :exit/location-2 ?loc]
                                           [?e :exit/location-2-wall ?wall])
                                          (and
                                           [?e :exit/location-1 ?loc]
                                           [?e :exit/location-1-wall ?wall])))
                       [(ground :not-found) ?dest]
                       [(ground false) ?locked]))]
```
When a valid action is found, something like the following result is returned
```clojure
#{[123 3431 true]}
```
The `true` in the above tuple means that it should be added to the view, whereas `false` would mean it should be removed if it exists.

Once we update the player's location, we can transact a datom marking this action as processed e.g. `[:db/add 123 :action/move-processed? true]`


## Implementation
The source code for the game is available [here](https://github.com/jumraiya/escape-room) . The most relevant namespace to look at is https://github.com/jumraiya/escape-room/blob/main/src/main/state_machine.cljs . It contains all the view definitions and the logic for state transitions. 
Overall the game logic turned out to be more complex than I expected, it's perhaps because I am trying to account for a lot of inconsequential states. 

The game was made by [Wizard](https://github.com/jumraiya/wizard). It is an experimental Clojure(script) library which maintains incremental views of datalog queries in memory (Currently only supports Datascript). The logic in Wizard is based on ideas described in the DBSP paper but not exactly 1 to 1, for instance the ZSets don't have an associated count, instead just a true/false value indicating an assertion or a retraction. This means the queries in Wizard can only return sets.

