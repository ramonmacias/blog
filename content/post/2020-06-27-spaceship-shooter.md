---
layout: post
title: "Spaceship Shooter Game"
subtitle: "My first game implemented in Go"
date: 2020-06-27
author: "Ramon"
URL: "/2020/06/27/spaceship-shooter/"
image: "img/newcastle.jpg"
tags:
    - Go
    - Game
---

> 90% of what is considered "impossible" is, in fact, possible. The other 10% will become possible with the passage of time & technology. -- Hideo Kojima

## Game

Probably one of the main motivation that I had when I was young, was the passion for video games, and this leads me to explore what is now my current professional career, computer engineer. Of course my main focus when I was young, is just play for fun, basically most of time I was thinking on having fun and nothing more, but is also true that sometimes I realised how real are some games. I remember been playing with a football game, and a part of having fun, I though on how real it is and that this is not magic, this game is made but some human and I wanted to be part of this group of people that has the chance of develop this "magic".

Sadly to say, that nowadays I'm not working as a game developer (who knows the future!), but my professional career has led me to discover and learn one of my favourites programming languages, Go.

So, of course, if at some point I need to code on my spare time a video game, this should be coded using Go!. Also I had no idea about how to start, how to structure the basics of a game, so I took some inspirations from this blog, https://mortenson.coffee/blog/making-multiplayer-game-go-and-grpc/ and I tried to do a similar game but from my point of view. The game is basically a shooter, you have a spaceship using an A on the map, and you are able to shoot a laser, represented by a red X on the map, and the enemies are represented as Y on the map. So the game ends when you die or when you kill all the enemies on the map.

## Code insights

My idea on this section is to talk a little bit about the code itself, and to go deep on some details I found interesting to comment. So on the next screenshot you will see how I arch the game.

![](/img/tree_go_spaceship.png)

We can see different things on the screenshot above, first point is that we use go modules on this project and we have a Makefile for manage our game, this makefile has only two commands, run and test.

```
// Will do a go run, this will build the go binary and will run the game
$ make run

// This command will execute all the tests
$ make test
```

We can find all the base code inside the two main directories, **cmd and internal**, on the cmd directory we will find the main package which is used for define and start the basics of the game and will be the start point for run our game. On the other hand we have the internal directory, inside this directory we keep all the game core, we divide this core code into two packages, **game and view**. So is a basic split between the logic of the game and how we draw the game itself, with that we can easily plug another library for render the game, on this case we used https://github.com/rivo/tview for render the game on the terminal. Let's take a look now at some pieces inside the game package.

## Engine

```
// Engine type will keep all the main information related with the game
type Engine struct {
	// Actors keep the information and link about all the interactors of the
	// game
	Actors map[uuid.UUID]Actor
	// GameMap keep link to the current map is playing
	GameMap Map
	// ActionChan is a buffered channel used for comunication between view and the
	// engine
	ActionChan chan Action
	// Score keep the info related with points and actors
	Score map[uuid.UUID]int
	// RoundWinner keep the id for the winner
	RoundWinner uuid.UUID
	// LevelComplete is the flag that determines when the level is complete
	LevelComplete bool
	// GameOver is the flag that determines when the player dies
	GameOver bool
	// Lasers keep the information about each lasers on the map
	Lasers sync.Map
	// Bots keep the information about bots on the map
	Bots sync.Map
}
```

The Engine type hold the core central structure of the game, some of the interesting things here could be the own custom Map that use an array of arrays of runes as a base type, also we use a go channel, ActionChan, that will in charge of receive all the different actions (movement and lasers) from the view and also we use sync.Map for all this data that needs to avoid any kind of collision and be sure there is no concurrency problems.

## Bots

```
// Bot represents the basic information needed for handle all the AI enemies on
// the game
type Bot struct {
	ID       uuid.UUID
	Life     int
	Position Point
	Strategy BotStrategy
}
```

As we can see, our enemies on the game are bots, the type above represents the basic information for define a Bot, which is a unique identifier, how many life it has, the current position on the map and the strategy that is following. We define four different bot strategies:

```
// BotStrategy holds the specific strategy in terms of movement and shootin that
// the bot is going to follow
type BotStrategy int

const (
	// NoMovementStrategy defines the bot will be just stopped
	NoMovementStrategy BotStrategy = iota
	// OnlyMovementStrategy defines the bot will be in movement but no shooting
	OnlyMovementStrategy
	// OnlyShootingStrategy defines the bot will be shooting all the time but without
	// movement
	OnlyShootingStrategy
	// ShootAndMoveStrategy define the bot will be in movement and shooting
	ShootAndMoveStrategy
)
```

So the first option, **NoMovementStrategy** means that this bot is doing nothing, just stuck on his initial position and nothing more. The second option is a bit more funny **OnlyMovementStrategy**, this means the bot with this strategy will keep moving on random direction until dies, the third option is **OnlyShootingStrategy**, that means is on a fixed position but shooting on random directions and the last one is a mix of the last two, **ShootAndMoveStrategy** defines a bot that is moving and shooting in random directions. On the next piece of code you will see how we code the last strategy, the following code belongs to a switch case inside a function

```
case ShootAndMoveStrategy:
  movementTicker := time.NewTicker(200 * time.Millisecond)
  shootingTicker := time.NewTicker(900 * time.Millisecond)
  for {
    b, exists := e.Bots.Load(bot.ID)
    if !exists {
      return
    }
    bot := b.(Bot)
    select {
    case <-movementTicker.C:
      e.ActionChan <- &BotMoveAction{
        BotID:     bot.ID,
        Direction: RandomDirection(),
        CreatedAt: time.Now(),
      }
    case <-shootingTicker.C:
      laserID := uuid.Must(uuid.NewV4())
      e.Lasers.Store(laserID, Laser{
        ID:       laserID,
        Position: bot.Position,
        Origin:   OriginBot,
      })
      e.ActionChan <- &LaserAction{
        LaserID:   laserID,
        Direction: RandomDirection(),
        CreatedAt: time.Now(),
      }
    default:
    }
  }
```

I’m using two time.Tickers, that will send a signal over a channel on a period of time, once I received the movementTicker tick, we create a new movement action and we send it using the engine action channel, so the shootingTicker will do the same but sending a laser action instead. So as I said on the comment above, this piece of code is running inside a function, and this function is running over a new go routine, the next piece of code we can see how we do that.

```
// startBots will apply all the strategies linked to each bot
func (e *Engine) startBots() {
	e.Bots.Range(func(key interface{}, value interface{}) bool {
		bot := value.(Bot)
		go bot.Strategy.perform(e, bot)
		return true
	})
}
````

## Main

```
func main() {
	player := game.Actor{
		ID:   uuid.Must(uuid.NewV4()),
		Name: "Ramon",
		Position: game.Point{
			X: 0,
			Y: 0,
		},
		Life: 3,
	}
	actors := make(map[uuid.UUID]game.Actor)
	actors[player.ID] = player
	engine := game.NewEngine(
		game.SetMap(MapDefault),
		game.SetActors(actors),
		game.SetBots([]game.BotStrategy{
			game.OnlyMovementStrategy, game.OnlyMovementStrategy, game.OnlyMovementStrategy, game.OnlyMovementStrategy,
			game.OnlyMovementStrategy, game.OnlyMovementStrategy, game.ShootAndMoveStrategy, game.OnlyMovementStrategy,
		}),
	)
	engine.Start()
	userInterface := view.New(engine)
	userInterface.MainPlayerID = player.ID
	userInterface.Start()

	err := <-userInterface.ErrChan
	if err != nil {
		log.Fatal(err)
	}
}
```

So the main function is pretty simple, we create an actor as a main player, and we fill with basic information needed, as the unique identifier, the name, initial position and the life. After this we add this player on the actors map and we generate the game engine using functional programming, setting the map, actors and all the bots with their own strategies.

With all this setup we can start the engine, that is going to listen for the events, on this case through the action channel. After this we build the view, attaching the game engine as a dependency and we start the view, drawing all the needed (map, player and bots) and as well setup all the listeners for react to the user (shooting lasers and moving the main spaceship).

## TODO

So if you want to check all the other details about the implementation, feel free to go https://github.com/ramonmacias/go-spaceship-shooter and take a look into the code and try the game!

On the original post, Mortenson also develop a gRPC server and a multiplayer. I will take that as thing TODO on the next game, I think is a good idea for understand a bit more the world of gRPC. The next video will show you a gameplay

<video width="100%" height="340" controls>
  <source src="/img/recording_game.mov" type="video/mp4">
</video>
