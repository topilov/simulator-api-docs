# simulator-api-docs
Introduction to simulator-api
## Get started
Create our simulator class:

````kotlin
@Simulator(
    inject = [
        // dependency to inject
    ],
)
class SampleSimulator {

    // load function
    @Load
    fun load(plugin: JavaPlugin) {
        // define simulator load logic
        // configure Anime, create Readers and e.t.c
    }
}
````

Create our SimulatorComponent class:

````kotlin
@SimulatorComponent(
    modules = [
        // define dagger modules
    ],
)
interface SimulatorComponentSettings
````

Create our User class for store user data:
````kotlin
@UserEntity
class User(id: UUID) : SimulatorUser(id) {
    var money: Double = 0.0
    var rebirth: Int = 0
    var level: Int = 0
}
````

Create  our UserFactoryImpl for create user:

````kotlin
class UserFactoryImpl : UserFactory<User> {
    override fun createUser(id: UUID): User {
        return User(id)
    }
}
````

Create our PlayerAcceptorImpl for accept player first join:

````kotlin
class PlayerAcceptorImpl : PlayerAcceptor {

    override fun accept(player: Player) {
        // give started items and e.t.c
    }
}
````

### Well done, our sample simulator is ready up to launch

## Deepening into the simulator api

### Boosters
Create enum with booster types:
````kotlin
enum class BoosterType {
    MONEY, EXP, ;
}
````

Create our sample booster configuration:
````kotlin
@Config
data class BoosterConfigurable(
    val type: Map<BoosterType, BoosterTypeConfigEntry>,
) {
    fun getBoosterTypeEntry(boosterType: BoosterType): BoosterTypeConfigEntry {
        return type[boosterType] ?: throw BoosterTypeNotSupportedException(boosterType)
    }
}

data class BoosterTypeConfigEntry(
    val title: String, // booster title
    val content: String, // text content
    val color: String, // text color
    val emoji: String, // emoji in ui hud
)

class BoosterTypeNotSupportedException(boosterType: BoosterType) :
    RuntimeException("BoosterType $boosterType not supported in config")
````

Create json file with our booster configuration:
````json
{
  "type": {
    "MONEY": {
      "title": "treasures.booster.money.title",
      "content": "treasures.booster.money.content",
      "color": "§6",
      "emoji": "\ue03c"
    },
    "EXP": {
      "title": "treasures.booster.exp.title",
      "content": "treasures.booster.exp.content",
      "color": "§b",
      "emoji": "\ue1a6"
    }
  }
}
````
Create wrapper booster from config to simulator api booster
````kotlin
data class BoosterWrapper(
    private val boosterType: BoosterType,
    private val boosterConfig: BoosterConfig,
) : SimulatorBoosterType {

    private val boosterTypeEntry = boosterConfig.getBoosterTypeEntry(boosterType)

    override val title = boosterTypeEntry.title
    override val content = boosterTypeEntry.content
    override val type = boosterType.name.lowercase()
}
````

Create booster factory to map boosters from config to simulator api boosters
````kotlin
@Factory(
    type = BoosterType::class,
    receiverType = SimulatorBoosterType::class,
    wrapper = BoosterWrapper::class,
)
interface BoosterFactorySettings
````

Example usage of factory:
````kotlin
@Singleton
class SampleBoosterController @Inject constructor(
    boosterFactory: BoosterFactory, // BoosterFactory is autogen class by ksp
) {
    init {
        println(boosterFactory.get(BoosterType.EXP))
    }
}
````

Create dagger FactoryModule to provide boosters to simulator-api:

````kotlin
 @Module
class FactoryModule {

    @Provides
    @Singleton
    fun provideBoosterTypes(boosterFactory: BoosterFactory) = boosterFactory.getAll()
}
````

#### Well done, our boosters are ready up to use!

### Leaderboards

Create LeaderboardConfig:
````kotlin
@Config
data class LeaderboardConfigurable(
    val leaderboards: Leaderboards,
)
````

Create json leaderboards configuration:
````json
{
  "leaderboards": [
    {
      "title": "treasures.top.exp.title",
      "field": "exp",
      "nickName": "treasures.top.nick-name",
      "fieldName": "treasures.top.exp.field-name",
      "fieldFormat": "§b%format%",
      "fieldSize": 20,
      "nameSize": 40,
      "limit": 50,
      "location": {
        "name": "top",
        "tag": "exp",
        "yaw": 90,
        "offset": {
          "x": 0.01,
          "y": 0.25,
          "z": 0.5
        }
      }
    }
  ]
}
````

Create dagger ConfigModule to provide our config entries to simulator-api:
````kotlin
@Module
class ConfigModule {
    @Provides
    @Singleton
    fun provideLeaderboards(leaderboardConfig: LeaderboardConfig): Leaderboards = leaderboardConfig.leaderboards
}
````

#### Well done, our leaderboards are ready up to use!

### Daily rewards:

Create DailyConfigurable:
````kotlin
@Config
data class DailyConfigurable(
    val message: DailyRewardsMessageConfigEntry,
    val animateTitle: DailyRewardsTitleConfigEntry,
    val rewards: Map<Int, DailyReward>,
) {
    fun getDailyReward(day: Int): DailyReward {
        return rewards[day] ?: throw UnsupportedDayException(day)
    }
}

data class DailyRewardsMessageConfigEntry(
    val takeReward: String,
)

data class DailyRewardsTitleConfigEntry(
    val title: String,
    val subtitle: String,
)

class UnsupportedDayException(day: Int) :
    RuntimeException("Day $day not supported in config")
````

Create json daily configuration:
````json
{
  "message": {
    "takeReward": "treasures.daily.message.take-reward"
  },
  "animateTitle": {
    "title": "treasures.daily.animate-title.title",
    "subtitle": "treasures.daily.animate-title.subtitle"
  },
  "rewards": {
    "1": {
      "hover": "treasures.daily.reward.1.hover",
      "texture": {
        "type": "CLAY_BALL",
        "data": {
          "skyblock": "donate"
        }
      },
      "onSuccess": [
        "earn-money %player% 25 false",
        "take-daily %player% 1"
      ]
    }
    // ... specify other days to 7
  }
}
````

Add providing to ConfigModule for DailyRewards:
````kotlin
    @Provides
    @Singleton
    fun provideDailyRewards(dailyConfig: DailyConfig): DailyRewards = dailyConfig.rewards
````

#### Well done, our daily rewards are ready up to use!

### Event
Create EventType enum:
````kotlin
enum class EventType {

    SAMPLE_EVENT, ;
}
````
Create EventConfigurable:
````kotlin
@Config
data class EventConfigurable(
    val event: Map<EventType, EventTypeConfigEntry>
) {
    fun getEvent(eventType: EventType): EventTypeConfigEntry {
        return event[eventType] ?: throw UnsupportedEventException(eventType)
    }
}

data class EventTypeConfigEntry(
    val id: String,
    val topEntries: Int,
    val end: Int,
    val start: List<String>,
    val startMessage: String,
    val endMessage: String,
    val reward: Map<Int, Commands>,
)

class UnsupportedEventException(eventType: EventType) :
    UnsupportedOperationException("Event $eventType not supported in config")
````

Create json event configuration:
````json
{
  "event": {
    "SAMPLE_EVENT": {
      "id": "sample",
      "topEntries": 3, // top places
      "end": 300, // duration in seconds
      "start": [
        "0:0:0", // start event in 12:00 AM
        "22:0:0" // start event in 10:00 PM
      ],
      "startMessage": "treasures.event.explosions.start-message", // send message on start event
      "endMessage": "treasures.event.blocks.end-message", // send message on end event
      "reward": {
        "1": [
          "give-key %player% PET", // give pet key for 1 place
          "random-money %player% 50.0 150.0 2147483647" // give random money for 1 place
        ],
        "2": [
          "give-key %player% DEFAULT",
          "random-money %player% 50.0 100.0 2147483647"
        ],
        "3": [
          "give-key %player% DEFAULT",
          "random-money %player% 25.0 100.0 2147483647"
        ]
      }
    }
  }
}
````

Create EventWrapper:
````kotlin
data class EventWrapper(
    private val type: EventType,
    private val eventConfig: EventConfig,
) : IEventType {

    private val eventEntry = eventConfig.getEvent(type)

    override val id = eventEntry.id
    override val topEntries = eventEntry.topEntries
    override val end = eventEntry.end
    override val start = eventEntry.start
    override val startMessage = eventEntry.startMessage
    override val endMessage = eventEntry.endMessage
    override val reward: (Player, Int) -> Unit = { player, position ->
        eventEntry.reward[position]?.invoke(player)
    }
}
````
Create EventFactory:

````kotlin
@Factory(
    type = EventType::class,
    receiverType = IEventType::class,
    wrapper = EventWrapper::class,
)
interface EventFactorySettings
````

Provide events to simulator-api in FactoryModule:
````kotlin
    @Provides
    @Singleton
    fun provideEvents(eventFactory: EventFactory) = eventFactory.getAll()
````
###
#### Well done! events are ready up to use
###

For convenience, create UseCase for earn event point
````kotlin
@Singleton
class EarnEventPointsUseCase @Inject constructor(
    private val eventFactory: EventFactory,
    private val eventController: EventController,
) {

    operator fun invoke(player: Player, eventType: EventType, points: Double) {
        val event = eventFactory.get(eventType)
        eventController.earnPoints(player, event, points)
    }
}
````

### Register Commands

Create kotlin file SampleCommand:
````kotlin
@Command(
    name = "spawn",
    usage = "/spawn <Boolean>"
)
internal fun spawn(@Sender player: Player, @Arg teleport: Boolean, @ArgInject spawn: Location) {
    if (teleport) {
        player.teleport(spawn)
    }
}
````
Explain:
1. Annonate field with @Sender to specify sender: Player or ConsoleCommandSender

2. Optional: annotate parameter as @ArgInject to provide parameter by dagger
3. Optional: annotate parameter as @Arg to provide parameter by command args, argument is auto cast to required type

Use @ArgMapper to auto cast argument to required type:

````kotlin
@ArgMapper
fun String.player(): Player = Bukkit.getPlayerExact(this)
````

Explain: player can specify in command arguments player name, and we can use @Arg player: Player in our command to fetch player

Example:
````kotlin
@Command(
    name = "sudo",
    usage = "/sudo <Player> <vararg String>"
)
internal fun sudo(
    @Sender console: ConsoleCommandSender,
    @Arg player: Player,
    @Args args: Arguments,
) {
    val command = args.readString(1, args.size)
    server.dispatchCommand(player, command)
}
````

### Register Listener
Create kotlin file SampleListener:
````kotlin
@Listener
internal fun EntityDamageEvent.cancel() {
    isCancelled = true
}
````

Note: we can use dagger dependency in listener parameters like this:

````kotlin
@Listener
internal fun PlayerInteractEvent.tryOpenChest(
    tryOpenChestUseCase: TryOpenChestUseCase,
) {
    if (clickedBlock == null) return
    if (action != Action.RIGHT_CLICK_BLOCK) return

    tryOpenChestUseCase(player, clickedBlock.location)
}
````

### Register Key Listener
Create sample kotlin file SpawnListener to teleport player to spawn by press on key H:
````kotlin
@KeyListener(key = "H")
internal fun spawn(player: Player, spawn: Location) {
    player.teleport(spawn)
}
````
Explain: spawn is providing by dagger

### Time schedulers
Create sample kotlin file RestartConfigure to auto restart our server in 6:00 PM
````kotlin
@TimeEvery(time = "6:0:0")
internal fun restart(restartController: RestartController) {
    restartController.restart()
}
````

Explain: restartController is providing by dagger

### Register Payload Handler
Create sample kotlin file PayloadConfigure to handle payloads by service and for example give gifts with lootbox keys:
````kotlin
@Payload(name = "sample-payload")
internal fun defaultGifts(lootboxController: LootboxController) {
    lootboxController.giveGifts(LootboxType.DEFAULT)
}

private fun LootboxController.giveGifts(lootboxType: LootboxType) {
    server.onlinePlayers.forEach { player ->
        giveKey(player, lootboxType)
    }
}
````

### Loadable functions
Create sample kotlin file SampleLoad to run task when simulator is launched:
````kotlin
@Load
internal fun execute() {
    println("Simulator is ready up to use!")
}
````

Note: we can use dagger dependency in function parameters

// soon... im too lazy