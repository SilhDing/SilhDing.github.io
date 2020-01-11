---
title: DFS_notes
date: 2019-10-03 11:27:09
tag: 
    - design
    - algorithm
categories:
    - technique
    - course
---

## Carcassonne Game Design

### Design Goal

The design goal in this project is to construct a comprehensive system that contains all concepts in the game and is able to execute all operations during the game. It should also correctly records the state of the game and return the correct results once the game is over.

As the game contains multiple objects, coupling between each object should be low, which may be helpful for debugging and implementation. Thus, each object should have clear responsibility and appropriate size of information, which should be helpful for implementation of algorithm in the future.

### Hierarchy and structure

The system contains the following classes:

#### `GameEntry`
The class is responsible for interacting with users or GUI. It also manages all other components that are critical to the game.In fact, this class would not be responsible for executing operations during the game; it would delegate these operations to other classes, which would be described letter.

#### `Player`
Instances of this class represents the players in this game. The class contains many information of this player, such as the the ID and points of the player.

#### `TileTower`
The tile deck. It manages all tiles that are unseen during the game; it also should poll a tile to the user once called. When the game is initially started, game entry would initialize one instance of this class and then, during the game, would iteratively get a tile from the tile tower until no tile is available (i.e., game is over);

#### `GameBoard`
The class is responsible for managing all placed tiles. It records all such tiles and their positions. When a user attempts to place a new tile, this class is also responsible for checking if the placement is valid and, if yes, placing the tile on the board. Another critical functionality of this class is to conduct scoring (if possible) when a newly placement happens.

#### `Tile`
Each instance of this class represents one separate tile in this game. It only contains the information about itself (e.g., its position, its segments, and positions of each segment) and it is independent of any other tiles in this game. The tile could also be rotated by the user: once it is rotated, all its segments should also be rotated.

#### `Segment`
This is an abstract class for all features in this game. One segment would records all its position in a tile, and one possible Meeple it has.

#### `CitySeg`, `RoadSeg`, `Monastery`

All three classes are subclasses of `Segment`. They share all variables and methods from `Segment` except the method `rotateSeg`, as each of them has different way to rotate the segment.

### Design highlights

In this design, I used several design pattern or strategies.

#### Inheritance

I used inheritance for all classes that represents features (e.g., city, road, monastery). It is good for code reuse, and `Tile` does not have to exactly know what kind of segment it contains, and does not have to record the meeple on it (`Segment` would record the meeple instead).

#### Responsibility

The game may contains multiple operations and each operation would change game states greatly. Thus, responsibility assignment is important. 

Here in my design, I follow the "information expert heuristic" principle to assign the responsibility for each class. Specifically, the class which has the information required for one operation is responsible for conducting the operation. For example, `GameBoard` records information of all placed tiles, so this class is responsible for detecting completed cities or roads. Such design choice has low representational gap and low coupling.