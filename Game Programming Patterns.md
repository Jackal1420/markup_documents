# Game Programming Patterns 
### An architectural approach by Robert Nystrom, notes by Jack Aulabaugh 

![anAwesomeImage](https://m.media-amazon.com/images/M/MV5BMTEyNTMzNDg1ODZeQTJeQWpwZ15BbWU3MDU5MDQ3NzQ@._V1_.jpg)

[Please support the author and buy the book.](https://www.amazon.com/Game-Programming-Patterns-Robert-Nystrom/dp/0990582906/ref=sr_1_1?crid=154ESHPLJ7T4O&keywords=Game+programming+patterns&qid=1669420037&sprefix=game+programming+patterns%2Caps%2C89&sr=8-1) It's extremely well written and it has witty sense of humor. You can also access the content on [Nystrom's website](https://gameprogrammingpatterns.com/contents.html).

## *Part 1: Design Patterns from "The Gang of Four"*
---
Nystrom revisits code architecture from the book, [*Design Patterns: Elements of Reusable Object-Oriented Software*](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612) by ["the gang of four"](https://en.wikipedia.org/wiki/Design_Patterns), and brings these patterns into the context of programming for game engine specific architecture. 

Not only are these patterns applicable to game development but they also are a great foundation to keeping a codebase you can grow and keep tidy - Robert defines good architecture as a system that adapts seamlessly to change. I also can't stress how important this is, because I have watched corporate code bases spiral out of control from top-down "waterfall" management and spaghettification (definitely not the right word for it but I like calling it that over "ball of mud"). 


![](https://static.wikia.nocookie.net/villains/images/4/4a/Zorg.jpg/revision/latest?cb=20190112122204)

*"Life, which you so nobly serve, comes from destruction, disorder and chaos."* - Jean-Baptiste Emanuel Zorg (and board executives irl)

## *Six patterns to change the program for good:*
---
A big positive of programming in the 2020s is that a lot of good practices have already been figured out for us by people who have way more experience than we do. The downside is... the code that helped programmers figure out what bad architecture is still exists and is being buried in a pile of tech debt (just look at how often airline systems fail... I mean gees guys, rewrite that crap already). This book is under the kind assumption that we're starting from scratch.

 The gang of four gives us some really good starting points that hopefully (fingers crossed) will limit our need to deal with nonsense and extensive changes later.
### **1.) Command**
A `command` is an object that encapsulates an action (a reified method call). Instead of directly executing an action we "wrap" that action in a more abstract form, and pass it around to be invoked by code. 

This allows us to handle actions (e.g. game control mappings to behaviors like 'jump' or 'run') in several unique ways: 
- Queueing actions and keeping track of execution order
- Swapping out existing actions for different types of actions 
- Supporting the ability to undo actions  

Here's an example: In a game we have some controller (xbox remote) and some action that the buttons correspond to (press "x" to jump). If we weren't using `commands` we would write the code like this: 

``` kt
fun handleControllerInput(pressed: ButtonType){
    when (pressed){
        ButtonType.BUTTON_X -> jump()
        ButtonType.BUTTON_Y -> fire()
        ButtonType.BUTTON_A -> sprint()
        ButtonType.BUTTON_B -> reloadWeapon()
    }
}
```
The above is a terrible design pattern we should avoid at all costs. It's not super hard to realize this... imagine trying to let users re-map your game buttons in an `options menu`. 

Instead we write an interface to encapsulate actions - a `command` class that we can override to perform any action we desire. 

``` kt
interface Command {
    open fun execute(){/*No op*/}
}

class JumpCommand : Command() {
    override fun execute(){
        jump()
    }    
}

class FireCommand : Command() {
    override fun execute(){
        fire()
    }    
}
// And so on...
```
Instead of calling game actions directly from the input handler, we can now pass in (or initilize) generic commands for each of the buttons:

```kt
class InputHandler(
    var buttonX: Command = JumpCommand(),
    var buttonY: Command = FireCommand(),
    var buttonA: Command = SprintCommand(),
    var buttonB: Command = CrouchCommand()
){  
    fun handleInput(button: ButtonType){
        when (button){
        	ButtonType.BUTTON_X -> buttonX.execute()
        	ButtonType.BUTTON_Y -> buttonY.execute()
        	ButtonType.BUTTON_A -> buttonA.execute()
        	ButtonType.BUTTON_B -> buttonB.execute()
        }
    }
}
```
This level of indirection gives us the flexibility to swap out existing button mappings with different types of `commands`, and keeping our actions encapsulated in objects allows us to enqueue them and keep a record of actions performed.

However, we are assuming that `JumpCommand()`, `FireCommand()` and other methods know exactly which game object to act on (the player's avatar in this instance). What if we wanted to change the 'actor' and allow our controls to pilot any object in the game? 

To reach a higher level of indirection, we need to communicate information about our `GameActor` by passing it into our execute methods:

```kt
/*
* A [GameActor] could be the player avatar, a wall, a bird, anything...
*/
interface Command {
    open fun execute(actor: GameActor){/*No op*/}
}

class JumpCommand : Command() {
    override fun execute(actor: GameActor){
        actor.jump()
    }    
}
```
Now, instead of directly executing the `commands`, we can get a lot more bang for our buck if we return the `commands` themselves from our `InputHandler` class to be executed 
```kt
class InputHandler(
    var buttonX: Command = JumpCommand(),
    var buttonY: Command = FireCommand(),
    var buttonA: Command = SprintCommand(),
    var buttonB: Command = CrouchCommand()
){  
    fun handleInput(button: ButtonType): Command?{
        when (button){
        	ButtonType.BUTTON_X -> return buttonX
        	ButtonType.BUTTON_Y -> return buttonY
        	ButtonType.BUTTON_A -> return buttonA
        	ButtonType.BUTTON_B -> return buttonB
        }
        return null
    }
}
```

Now we are able to delay when the `command` is actually executed, and this allows us to do a lot more with our game control `commands`: 

```kt
val command = inputHandler.handleInput(inputButton) // returns an action we can perform based on button presses 

if (command != null){
    commandHistory.add(command)
    command.execute(actor)
}
```
An actor can be literally anything! We can enqueue, dequeue, and undo `commands`! And we can apply our architecture to so much more. An example Nystrom gives is using our game AI to generate a stream of `commands` we enqueue to puppeteer game objects or enemies. 

The most important thing `commands` do is decouple producers from consumers and save us a lot of time that we'd spend pointlessly re-writing and maintaining lots of code.


### 2.) **Flyweight**
A `flyweight` is a pattern focused on optimization. It is used to isolate reusable data away from heavily used objects. 

For example, if we are trying to render a group of game objects (Nystrom uses trees as his example), we don't want to redraw the object mesh for thousands of unique game objects -- it's hard on processing power and it's far more efficient if we can defer to a static type for data that doesn't vary between objects. 

Imagine we need to draw a whole forest of trees for our game. All of those trees have pretty similar meshes, so we can defer to a standard "tree mesh", and vary the height, color, and texture instead of passing along a large array of data with each seperate tree. 

A `flyweight` seperates the *intrinsic* (shared data) state from the *extrinisic* (unique data) state.

```kt
// Instance specific properties 
class Tree(
    val position: Vector,
    val height: Int,
    val thickness: Int, 
    val barkTint: Color,
    val leafTint: Color 
){
    // Static members for shared data, 
    // initilized only once regardless of the number of objects created.
    companion object{
        val mesh = arrayOf<MeshPolygon>(/*...*/)
        val barkTexture = Texture(/*...*/)
        val leaves = Texture(/*...*/)
    }
}
```


### 3.) **Observer**
Observers are a staple of most MVVM pattern object oriented programming (you are almost guranteed to be familiar with this concpet if you are a web or mobile programmer).

The observer pattern relies on two different patterns
1. An object that broadcasts information **without caring who consumes it**
2. An object that listens for information **without caring who broadcasts it** 

It is crucial that an **observer** and **broadcaster** aren't aware of who they listening to or broadcasting for because otherwise we risk potentially tying together very different separations of concern. 

For example (Nystrom's example), think about the Xbox 360s `achievements system`. Imagine we're developing Skyrim and we want to give our players an achievement for falling off a bridge. 

We could directly call the `achievement sevice` from the `physics logic` that manages falling in the phsycis engine, but if we do that, we are directly coupling our physics engine with logic it should never touch and cluttering the game's ecosystem. 

Instead consider this pattern: 
- The `physics logic` broadcasts an event, `PLAYER_ON_BRIDGE`
- The `physics logic` broadcasts an event, `IS_FALLING`
- The `achievements service` runs an observer that sees all events from the physics engine
- The `achievements service` observes `PLAYER_ON_BRIDGE` followed by `IS_FALLING` within a certain time interval 
- The `achievements service` unlocks the special achievement 

Now our concerns are separated and operate autonomously from each other! 

*How can we actually implment an observer from scratch?* 

Let's start with an interface: 

```c++
class Observer {
    public:
        virtual ~Observer(){}
        virtual void onNotify(const Entity& entity, Event event){}
};
```
Any class that adopts this pattern becomes an observer, able to tap into events when it is linked with other game objects that hold observers. 

For example: 
```c++
class Achievements : public Observer {
    public:
        virtual void onNotify(const Entity& entity, Event event){
            switch (event){
                case EVENT_ENTITY_FELL:
                    if(entity.isHero() && heroIsOnBridge){
                        unlock(ACHIEVEMENT_FELL_OFF_BRIDGE);
                    }
                    break;
            }
        }
    private:
        void unlock(Achievement achievement){...}
        bool heroIsOnBridge
};
```
Subjects that employ observers can tap into them by adopting a subject pattern: 
```c++
class Subject{
    public:
        void addObserver(Observer* observer){...};
        void removeObserver(Observer* observer){...};
    private:
        Observer* observers[MAX_OBSERVERS];
        int numObservers;
    protected:
        void notify(const Entity& entity, Event event){
            for (int i = 0; i < numObservers; i++){
                observers[i] -> onNotify(entity, event)
            }
        }
}
```
Now if we wanted to unlock achievements from our physics engine it's a matter of setting up a subject for our physics root-level class 

```c++
class Physics : public Subject{
    public:
        void updateEntity(Entity& entity);
};
```
`Observers` are notorious for doing too much dynamic allocation. Right now we have a different problem -- we've already allocated a space of size `MAX_OBSERVERS` for classes that implement `Subject`. 

We can solve our allocation problems by changing how we implement our list of `Observers` for our `Subject` to a linked list.
```c++
void Subject::addObserver(Observer* observer){
    observer->next = head_;
    head_ = observer;
}

void Subject::removeObserver(Observer* obserever){
    if (head_ == observer){
        head_ = observer->next_;
        observer->next_ = NULL;
        return;
    }

    Observer* current = head_;
    while (current != NULL){
        if(current->next_ == observer){
            current->next_ = observer->next_;
            observer->next_ = NULL;
            return;
        }
        current = current->next_;
    }
}
```
One last note -- most languages will have a Garbage Collector that cleans up the mess of observers that you hook up within their framework. But I know all too well from the android world that you should NEVER trust the GC to take care of your mess. You will cause memory leaks if you handle this pattern impoperly. 

If you fail to unregister an observer from a subject when it's served its purpose, it will not be garbage collected and it will linger as an unusable deadweight in your application since the GC interprets that remaining linkage as "oh... this thing is still being used."

### 4.) **Prototype**

Prototypes are a pattern of quickly cloning or creating an object. If we are making a game that involves creating lots of enemies of different varieties we could try implementing a tangled mess of "spawners" or we could enact the prototype pattern: 
> The key idea is that an object can spawn other objects similar to itself. 

For example: 
```kt
interface Monster {
    fun clone(): Monster
}

class Ghost: Monster{
    override fun clone(): Monster{
        return Ghost(health, speed)
    }
}
```
Now to create any kind of `Monster` all we need is a single generic `Spawner` to invoke `clone()`.
```kt
val ghostPrototype = Ghost(STANDARD_HEALTH, STANDARD_SPEED)
val ghostSpawner = Spawner(ghostPrototype)
``` 
We could also make spawn functions for each monster class instead: 
```kt
fun Ghost.spawn() = Ghost(STANDARD_HEALTH, STANDARD_SPEED)
```
Prototypes are a bit of an older pattern and can make code more cluttered and hard to work with. It's more often useful to use delegation for data modeling in JSON: 

```json
{
    "name":"goblin grunt",
    "minHealth":10,
    "maxHealth":30,
    "weaknesses":["fireball", "lightning"]
}

{
    "name":"goblin wizard",
    "prototype":"goblin grunt",
    "spells":["sapHealth","iceBolt"]
}

{
    "name":"goblin archer",
    "maxHealth":50,
    "prototype":"goblin grunt",
    "weapons":["short bow"]
}
```

### 5.) **Singleton**
Singletons are a singular object that can be referenced from anywhere. They are a larger cousin of global variables and should be avoided like global variables because they turn our code into a ball of mud much faster. There are reasons to use singletons (i.e. if you want to provide access to an external api, my current employer uses a singleton to provide feature flags from LaunchDarkly's api), but it's generally something you will want to avoid.

Singletons are bad for these reasons 
- They are GOBAL and make it harder to track down problems (e.g. if you trying to diagnose a problem involving a singleton, you have to look into all code that accesses it)
- They tightly couple your code (systems that should have no intertion may both rely on a singleton and become intimately involved with each other)
- They SUCK with concurrent processes (you need to worry about locking and state A LOT)
- You lose control of performance via lazy instantiation (e.g. if you are running a game and a singleton sets up suddenly, it eats memory)

One example from our book is providing access to a computer's file system, which is something that varies between machines that we want to wrap in our own API (for logging, content loading, saving game states, etc).

```kt
class FileSystem{
    //...
}

class FileSystemProvider{
    private var instance: FileSystem? = null
	
    /*
        This is a process known as lazy initilization, 
        it allows us to keep memory free if we end up 
        never using the instance. 

        It also allows us to create an instance as 
        "late as possible" so that it has access to 
        lots of information that wouldn't be known 
        prior to runtime.
    */
    fun getFileSystem():FileSystem{
        if(instance == null){
            instance = FileSystem()
        }
        return instance!!
    }
}
```
Having a singleton like this can also benefit us by being 'vague.' If we want our code to run on MacOS, Windows, Linux, etc, all we need to do is subclass this singleton and tailor it to each of these systems. The rest of our game logic will only interact with a "FileSystem", the FileSystem subclasses are the only objects that have to worry about different paradigms. This is very much a concept game companies will create with "hardware abstraction layers" to allow their games to run on devices with different GPUs / Hardware. 

**What to do instead?**
Very often, most singletons are poorly designed helpers that add functionality to an existing class, and finding a way to migrate them into a class is the solution (e.g. making static members of a class instead of a new singleton).

We could also try providing a single instance without global access (e.g. a class that rejects all but one attempt to instantiate it).

Convienent access is the primary reason that these monsters exist. Sometimes it's necessaray but we could accomplish the same level of access by 
- Passing on a class to the functions that need it 
- Having a 'singleton' functionality be inherited through OOP subclassing. 
- Attaching 'singleton' functionality to something already global (e.g. extension functions (?))

### 6.) **State**

State is a programming pattern that allows us to manage behavior in a concrete and logical way rather than managing a whole range of wild conditions.

For instance, if you wanted to create this functionality 

- "Press B to jump."
- "Press DOWN to duck while on the ground." 

You would need a lot of conditional logic to control the player's motions AND account for the bugs that you'll notice crop up from initial intuition: 
```c++
void Heroine::handleInput(Intput input){
    switch(input){
        // We can only jump when not already jumping or ducking
        case PRESS_B:
            if(!isJumping && !isDucking){
                isJumping = true;
                Jump() // sets isJumping to false when on ground 
            }
            break;
        // We can only duck when not jumping
        case DOWN:
            if(!isJumping){
                isDucking = true;
                Duck()
            }
            break;
        // Handle a case where the crouch animation shows up in air
        case RELEASE_DOWN:
            isDucking = false;
            setGraphics(IMAGE_STANDING) 
            break;

    }
}
```
This gets messy very quickly. Imagine wanting to add a diving state that allows 'jump attacks' from the air.

Instead we need to go back to the game drawing board ... literally, draw out the exact states you want to allow for the player actions, and organize them in a flow chart. This Finite State Machine shows us the exact path we should take programatically to get the best managable code. 


![State Diagram](https://gameprogrammingpatterns.com/images/state-flowchart.png)


A Finite State Machine has 
- A fixed set of states the machine (player) can be in
- The machine (player) can only be in one state at a time
- A sequence of inputs or events triggers change between states 
- Each state has a set of transitions associated with an input and transitioning to another state

A great resource for drawing FSMs is [draw.io](https://app.diagrams.net/). 

We can add game states programmatically by storing them in enum classes
```kt
enum class PlayerState{
    IS_JUMPING,
    IS_DUCKING,
    IS_STANDING,
    IS_DIVING
}

fun handleInput(input:Input){
    when(state){
        IS_JUMPING -> {
            if(input == PRESS_DOWN){
                state = IS_DIVING
            }
        }
        IS_DUCKING -> {
            if(input == RELEASE_DOWN){
                state = IS_STANDING
            }
        }
        IS_STANDING ->{
            if(input == A_BUTTON){
                state = IS_JUMPING
            }
        }
        else -> {
            // error 
        }        
    }
}
```

Suddenly our code becomes far less conditional and much easier to manage. 

However, the full `state` design pattern is a little more nuanced and takes advantage of OOP concepts: 

- We definie our `state` as an `interface` that we implement
- Every state is a class that impements that interface 
- We replace our `switch` with a `virtual` method in the `interface`

```c++
class HeroineState{
    public:
        virtual ~HeroneState() {}
        virtual void handleInput(Heroine& heroine, Input input){}
        virtual void update(Heroine& heroine){}
};


class DuckingState : public HeroineState(){
    public:
        
        DuckingState() : chargeTime_(0){}
        
        virtual void handleInput(Heroine& herione, Input input){
            if (input == RELEASE_DOWN){
                heroine.setGraphics(IMAGE_STANDING)
            }
            if (input == PRESS_B){
                heroine.setGraphics(IMAGE_JUMP)
                heroine.state_ = &HeroineState::jumping;
            }
        }

        virtual void update(Heroine& heroine){
            chargeTime_++;
            if(chargeTime > 10){
                heroine.superBomb();
            }
        }

    private:
        int chargeTime_; // for charging an attack while in a ducking state 
};
```

Now, if we delegate the `state` to be on the `Heroine` class itself we can simplify how we handle input.

```c++
class Heroine{
    public:
        // Called on each key press 
        virtual void handleInput(Input input){
            HeroineState* newState = state->handleInput(*this, input);
            if(newState != NULL){
                delete state_;
                state_ = newState;
            }
        }
        // Called on each game frame 
        virtual void update(){
            state_->update(*this) 
        }
    private:
        HeroineState* state_;
};
```
While this conveys most of the point we can also set up states to use an entry action that allows them to own their own graphical settings. 
```c++
virtual void enter(Heroine& heroine){
    heroine.setGraphics(IMAGE_STAND)
}
```
```c++
delete _state;
state_ = newState;
state_->enter(*this);
```
![](https://academics.winona.edu/povwinona/wp-content/uploads/sites/4/2022/03/12-monkeys-review-willis-pitt.jpg)

"She couldn't turn back time, thank you Einstein!" - Jeffrey Goines 

## *Part 2: Sequencing Patterns, time and motion*
---
Description goes here.

### 1.) Double Buffer 
### 2.) Game Loop 
### 3.) Update Method 


![](https://media.criticalhit.net/2018/11/AustinPowers_1.jpg)

## *Part 3: Behavioral Patterns - Everything, everywhere all at once*
---
Description goes here.

### 1.) Double Buffer 
### 2.) Game Loop 
### 3.) Update Method 

![](https://i.kym-cdn.com/entries/icons/original/000/032/744/maneuver.jpg)

## *Part 4: Decoupling Patterns - Get ready for some changeeeee*
---
Description goes here.

### 1.) Component
### 2.) Event Queue  
### 3.) Service Locator  
