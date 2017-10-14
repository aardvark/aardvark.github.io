---
layout: post
title:  "Building Starcraft2 Bot"
date:   2017-09-17 00:00:00 +0300
categories: starcraft c++ tutorial
---
**Introduction**

Starcraft game is easy to grasp hard to master. Goal of this posts to
create scripted AI capable of beating 'Impossible' level Starcraft game
AI opponent. Bot will be created using [sc2client-api](https://github.com/Blizzard/s2client-api). sc2client-api is
a open c++ api library created by [Blizzard](http://eu.blizzard.com/en-gb/) for creation of scripted
and AI bots.

**Disclaimer #1**. I have very vague idea how c++ build and runtime
process works under the hood. I do get paid for creating a computer
programs of different complexity and functionality, but none of then
was written in c++. I also didn't do any AI development in my life
except maybe "Guess a Number" when i was 10.

As any strategy game Starcraft has micro(unit control and coordination) and
macro (producing unit and accumulation of resource) levels.  Our first
step will be to create a simple script bot that will drown enemy in
steady waves of marines.

**Disclaimer #2.** I am also not amazing at Starcraft. I can beat
'Impossible' AI myself, but this is as far as it goes.

If you want to run code examples for yourself i urge you to proceed with 
documentation in official repository [docs](https://github.com/Blizzard/s2client-api/tree/master/docs).

Code snippet's are based on code present in tutorial2 present in api
repository.

Lets start by limiting number of scv that will be alocated to mine
minerals.

**Limit scv to the optimal amount on minerals**

We should build scv when our command center is idle.

Ideal number of scv assigned to mineral patch is 2. This account for
round trip time of minerals delivery to the command center. This gives
us an ideal number of scv assigned to the mineral harvest duty as
number of patches x 2.

So to get ideal number of scv we need to count number of mineral
patches around command center and then count number of scv assigned to
command center and if we have less than ideal number we will order
command center to build scv.

As it happens api already provide ideal and assigned harvester
numbers.  This make our task easier. Now we only need to wrap our
train scv call into simple if check and we done.

{% highlight c++ %}
    case UNIT_TYPEID::TERRAN_COMMANDCENTER: {
        if (unit.assigned_harvesters < unit.ideal_harvesters) {
            Actions()->UnitCommand(unit, ABILITY_ID::TRAIN_SCV);
        }
        break;
    }
{% endhighlight %}

Now we can test it.  Aaaand&#x2026; nothing happens. Why?  

Apparently (or obviously if we read documentation) unit are only
signaled idle when they are first become idle. When game starts there
a couple of steps when all our SCV is still on route to the minerals
and not registered as assigned on command center. We can fix this by
adding additional condition of assigned<sub>harvest</sub> == 0 to the guarding
if.

{% highlight c++ %}
    case UNIT_TYPEID::TERRAN_COMMANDCENTER: {
        if (unit.assigned_harvesters == 0 || unit.assigned_harvesters < unit.ideal_harvesters) {
            Actions()->UnitCommand(unit, ABILITY_ID::TRAIN_SCV);
        }
        break;
    }
{% endhighlight %}

Yay! We done it. 

*Build Barracks*

Now we can:
-   create supply depots when we need them.
-   'saturate' mineral patches with our worker units.

We ready to build our first army.

Basic combat producing facility for Terran race is Barracks. Barracks
cost 150 minerals and require Supply Depot as a prerequisite. Let's
adapt existing supply depot building function to build Barracks.

{% highlight c++ %}
    bool TryBuildBarracks() {
        Units depots = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_SUPPLYDEPOT));
        if (depots.size() >= 1) {
            return TryBuildStructure(ABILITY_ID::BUILD_BARRACKS);
        }
        return false;
    }
{% endhighlight %}

Result:

![Barracks](/assets/img/lotsOfBarracks.png){:class="img-responsive"}

This works kinda too well. Our bot build barracks indefinitely. This
is not what we want now. Let's limit total number of barracks by 3.

{% highlight c++ %}
    bool TryBuildBarracks() {
        Units depots = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_SUPPLYDEPOT));
        Units barracks = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_BARRACKS));
        if (barracks.size() < 3 && depots.size() >= 1) {
            return TryBuildStructure(ABILITY_ID::BUILD_BARRACKS);
        }
        return false;
    }
{% endhighlight %}

And we have 3 Barracks! Eventually. 

**Build 8 marines and attack**

Now we can start building an army. We can try to reuse exiting 
OnUnitIdle callback to build Marines. And remembering our experience 
with barracks we will set limit to 8.

{% highlight c++ %}
    case UNIT_TYPEID::TERRAN_BARRACKS: {
        Units marines = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_MARINE));
        if (marines.size() < 8) {
            Actions()->UnitCommand(unit, ABILITY_ID::TRAIN_MARINE);
        }
    }
{% endhighlight %}

Why did we build more than 8?  

Our marine check is flawed, obviously. We only count existing marines 
and discard any knowledge about ones we producing. Let's amend check
logic to also count marines in training.

Queue information stored as simple vector of orders. So all we need to 
do is count how many TRAIN_MARINE orders contained in barrack orders
and add it to our total.  

{% highlight c++ %}
    case UNIT_TYPEID::TERRAN_BARRACKS: {
        Units marines = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_MARINE));
        Units barracks = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_BARRACKS));
        int marines_in_prod = 0;
    
        for (const auto& barrack : barracks) {
            for (const auto& order : barrack.orders) {
                if (order.ability_id == ABILITY_ID::TRAIN_MARINE) {
                    marines_in_prod++;
                }
            }
        };
    
        if (marines.size() + marines_in_prod < 8) {
            Actions()->UnitCommand(unit, ABILITY_ID::TRAIN_MARINE);
        }
    }
{% endhighlight %}

Now lets order our marines to attack enemy. 
To order attack we need 3 things:
-   attacker
-   location to attack
-   id of attack ability

In case of scv we can use smart command to harvest minerals. If we
order marine to use smart command on location he proceed to move to
this location ignoring all distractions until he arrive to the ordered
point.

This is viable tactics but we prefer to send army units in attack
move.  Unit moving with attack move will attack everything he can on the
way to the destination.

Location of the enemy base we can get from game_info object. As we
currently have only one enemy we can simple take first value.

{% highlight c++ %}
    bool TryAttackWithMarines() {
        Units marines = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_MARINE));
        if (marines.size() >= 8 ) {
            GameInfo game_info = Observation()->GetGameInfo();
            Point2D enemy_start_location;
            enemy_start_location = game_info.enemy_start_locations.front();
    
            std::vector<Tag> marine_tags;
            for (const auto &marine : marines) {
                marine_tags.push_back(marine.tag);
            }
            Actions()->UnitCommand(marine_tags, ABILITY_ID::ATTACK, enemy_start_location);
            return true;
        }
        return false;
    }
{% endhighlight %}

So we have our basic bot. 

<iframe width="560" height="315" src="https://www.youtube.com/embed/nbNGC_Lxvhg" frameborder="0" allowfullscreen></iframe>

What can be improved? 
1.  Barracks are build one by one and as a result we are getting very
    high amount of minerals in the bank
2.  We are not building additional marines after we send
    first 8. Marine production should be constant. Ideally we should
    produce more marines as game progress.

**Build barracks concurrently**

TryBuildStructure allow only one type of building in build queue over
time.  Lets create new function that allow us to build more than one
building concurrently.

{% highlight c++ %}
    bool TryBuildStructureConcurrently(ABILITY_ID ability_type_for_structure, 
                                       int concurrent_number = 1, 
                                       UNIT_TYPEID unit_type = UNIT_TYPEID::TERRAN_SCV) {
        const ObservationInterface *observation = Observation();
    
        Units all_scvs = observation->GetUnits(Unit::Alliance::Self, IsUnit(unit_type));
        std::vector<Unit> builders;
        int already_building = 0;
        for (const auto& scv : all_scvs) {
            if (already_building >= concurrent_number) {
                return false;
            }
            for(const auto& order : scv.orders) {
                if (order.ability_id == ability_type_for_structure) {
                    already_building++;
                    break;
                } else {
                    builders.push_back(scv);
                }
            }
        }
    
        builders.resize(concurrent_number - already_building);
    
        for (const auto& scv : builders) {
            float rx = GetRandomScalar();
            float ry = GetRandomScalar();
    
            Actions()->UnitCommand(scv,
                                   ability_type_for_structure,
                                   Point2D(scv.pos.x + rx * 15.0f, scv.pos.y + ry * 15.0f));
        }
        return true;
    }
{% endhighlight %}

We can now replace existing TryBuildStructure function with ours.

**Build marines constantly**

Now let's address deadly lack of marines. So we build our initial 8 
marines and send them to the glorious death. And we expect to build 8 
more when population decrease due to natural causes. This doesn't 
happen. Let's check how and when OnUnitIdle event is fired. 

From documentation of OnUnitIdle event:

    //! Called when a unit becomes idle, this will only occur as an event so will only be called when the unit becomes
    //! idle and not a second time. Being idle is defined by having orders in the previous step and not currently having
    //! orders or if it did not exist in the previous step and now does, a unit being created, for instance, will call both
    //! OnUnitCreated and OnUnitIdle if it does not have a rally set.

So we can't use OnUnitIdle to stop production and then resume it. 
Looks like we will need to add our building logic to the OnStep event. 

Lets modify our function to work in OnStep context.
[expand on what required to migrate function to OnStep]

{% highlight c++ %}
    bool TryBuildMarines(){
        int marine_limit = 100;
        Units marines = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_MARINE));
        Units barracks = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_BARRACKS));
        int marines_in_prod = 0;
    
        std::vector<Tag> idle_barracks;
        for (const auto &barrack : barracks) {
            for (const auto &order : barrack.orders) {
                if (order.ability_id == ABILITY_ID::TRAIN_MARINE) {
                    marines_in_prod++;
                }
            }
            if (barrack.orders.size() == 0) {
                idle_barracks.push_back(barrack.tag);
            }
        }
    
        if (idle_barracks.size() > 0 && marines.size() + marines_in_prod < 100) {
           Actions()->UnitCommand(idle_barracks, ABILITY_ID::TRAIN_MARINE);
        }
        return true;
    }
{% endhighlight %}

And now we have a steady stream of marines happily walking to their 
death. Even Easy AI can defend from 1 marine at the time. We need send
our forces in a more massive or grouped way. 

To achieve this we need:

1.  Specify rally point for all barracks.
2.  Send all marines in vicinity of rally point to attack enemy.

**Attack with group of marines**

In game rally point can be set from barracks by clicking with right 
mouse button anywhere on map or by using special "Set rally point" 
command. This mean two things for us: 

1.  We can set rally point from barrack
2.  We will need some coordinates on map to set rally point.

We can try to teach bot to calculate rally point or we can hardcode 
coordinates for time being. So hardcode it is. 

Bel'Shir Vestige is a 2 players map, so we will need to get 2 location
and chose one depending on starting location.
for those are: 

Starting location | Rally Point location
111.5, 25.5  | 77.574707, 30.615479
29.5, 134.5  | 67.003418, 127.894043

We will add marine_staging_location variable to the Bot class

{% highlight c++ %}
    Point2D marine_staging_location;
{% endhighlight %}


And will set staging location in OnGameStart event like this: 

{% highlight c++ %}
    Unit cc = Observation()->GetUnits(Unit::Alliance::Self, IsUnit(UNIT_TYPEID::TERRAN_COMMANDCENTER)).front();
    if (cc.pos.x == 29.5) {
        marine_staging_location = Point2D(67.003418f, 127.894043f);
    } else {
        marine_staging_location = Point2D(77.574707f, 30.615479f);
    }
{% endhighlight %}

Next we will nedd to use this location to set rally point on newly created barracks
it can be done like this:

{% highlight c++ %}
    virtual void OnUnitCreated(const Unit& unit) final {
        switch(unit.unit_type.ToType()) {
        case UNIT_TYPEID::TERRAN_BARRACKS: {
            Actions()->UnitCommand(unit, ABILITY_ID::RALLY_UNITS, marine_staging_location);
        }
        default: {
            break;
        }
        }
    }
{% endhighlight %}

Are we there yet? 

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/kC7cr5JuL3E" frameborder="0" allowfullscreen></iframe>

I say we are done for today. 
