eventsim
========

Eventsim is a program  that generates event data for testing and demos. It's written in Scala (which provides
a good balance of scalability, brevity, and clarity). It's designed to replicate page requests for a fake music
web site (picture something like Spotify); it's designed to look like real use data, but be totally fake.

Statistical Model
=================

I wrote this simulator based on observations about how real users behave. I wanted to make sure that data looked
real: users would come and go randomly, some users would stay much longer than others, users would be more likely to
use the service in the middle of the day than the middle of the night, and much less likely to use the service on
weekends and holidays.

To make this work, I did the following:

* If you set the "damping" factors to zero, then users randomly arrive at the site according to a Poisson (memoryless)
process, but with a minimum gap of 30 minutes between sessions.
* The time between events is given by a log-normal disribution
* Once a sessions has started, the user will randomly traverse a set of states until the session ends. The probability
of each state transition (including end of session) depends on the current state.
* On average, users will behave the same way in a session, regardless of the time of day or day of week
* If you enable damping for weekends and holidays, the probability that a user arrives on weekends and holidays drops.
The odds are scaled linearly over a course of a few hours (by default) around midnight (by default).
* If you enable damping for nighttime, the probability that a user arrives in the middle of the night is lower than
the probability that they arrive in the middle of the day. The odds roughly follow a sine wave.


How the simulation works
========================

When you run the simulator, it starts by generating a set of users with randomly picked properties. This includes
attributes like names and location as well as usage characteristics, like user engagement. Eventsim uses a
pseudo-random number generator: the generator is deterministic, but looks random. (We picked a Mersenne Twister as the
PRNG because it's fast, and well respected.)

You need to specify a configuration file (a sample is included in `examples/example-site.json`). This file
specifies how sessions are generated, and how the fake website works. In this file you can specify several parameters:

* Alpha. This is the expected number of seconds between events for a user. This is randomly generated from a lognormal
distrbution
* Beta. This is the expected session interarrival time (in seconds). This is thirty minutes plus a randomly selected
value from an exponential distribution
* Damping. This is the damping factor for daily cycles (larger values yield stronger cycles, smaller yield milder)
* Weekend Damping. This controls when and how quickly weekend traffic falls off
* Seed. Seed for the random number generator.

You also specify the event state machine. Each state includes a page, an HTTP status code, a user level, and an
authentication status. Status should be used to describe a user's status: unregistered, logged in, logged out,
cancelled, etc. Pages are used to describe a user's page. Here is how you specify the state machine:

* Transitions. Describe the pair of page and status before and after each transition, and the
probability of the transition.
* New user. Describes the page and status for each new user (and probability of arriving for the
first time with one of those states).
* New session. Describes that page and status for each new session.
* Show user details. For each status, states whether or not users are shown in the generated event log.

When you run the simulator, you specify the mean values for alpha and beta and the simulator picks values for specific
users.

The simulator works by creating a priority queue of user sessions, ordered by the timestamp of the next event in
each session. The simulator picks each session off the queue, outputs the details of the event, then determines the
next event in the session for each user (or creates a new session for the user), and puts the session back in the queue.

When the simulator figures out the next event in the sessions, it looks at all of the possible transitions from the
current state to other states. If the total probability of all the transitions is 1.0, then there will always be a
next page. However, if the probability is n < 1.0, then with probability 1.0 - n the session will end, and the
user will next be seen at a future session.

Most of the time, the next event will occur at a non-deterministic, log-normally distributed time after the current
event. But there are two exceptions: "nextSong" events and redirects. The next song events will typically occur
after the duration of the current song. Redirects occur at a fixed time afterwards (we did this to model the action of
submitting a form then being redirected to a new page).

By the way: the current version of the simulator is hard-coded for a music web site. You can modify it to work for
other types of sites, but doing so will probably require modifications to the code (and not just to the config files).

Usage
=====

To build the executable, run

    $ sbt assembly
    $ # make sure the script is executable
    $ chmod +x bin/eventsim

(By the way, eventsim requires Java 8.)

The program can accept a number of command line options:

    $ bin/eventsim --help
        -a, --attrition-rate  <arg>    annual user attrition rate (as a fraction of
                                       current, so 1% => 0.01) (default = 0.0)
        -c, --config  <arg>            config file
            --continuous               continuous output
            --nocontinuous             run all at once
        -e, --end-time  <arg>          end time for data
                                       (default = 2015-08-12T14:56:25.006)
        -f, --from  <arg>              from x days ago (default = 15)
            --generate-counts          generate listen counts file then stop
            --nogenerate-counts        run normally
            --generate-similars        generate similar song file then stop
            --nogenerate-similars      run normally
        -g, --growth-rate  <arg>       annual user growth rate (as a fraction of
                                       current, so 1% => 0.01) (default = 0.0)
            --kafkaBrokerList  <arg>   kafka broker list
        -k, --kafkaTopic  <arg>        kafka topic
        -n, --nusers  <arg>            initial number of users (default = 1)
        -r, --randomseed  <arg>        random seed
        -s, --start-time  <arg>        start time for data
                                       (default = 2015-08-05T14:56:25.040)
            --tag  <arg>               tag applied to each line (for example, A/B test
                                       group)
        -t, --to  <arg>                to y days ago (default = 1)
        -u, --userid  <arg>            first user id (default = 1)
            --help                     Show help message

       trailing arguments:
        output-file (not required)   File name

Only the config file is required.

Parameters can be specified in three ways: you can accept the default value, you can specify them in the config file,
or you can specify them on the command line. Config file values override defaults; command line options override
everything.

Example for 2.5 M events:

    $ bin/eventsim -c "examples/site.json" --from 365 --nusers 1000 --growth-rate 0.01 data/fake.json
    Initial number of users: 1000, Final number of users: 1010
    Starting to generate events.
    Damping=0.0625, Weekend-Damping=0.5
    Start: 2013-10-06T06:27:10, End: 2014-10-05T06:27:10, Now: 2014-10-05T06:27:07, Events:2468822

Example for more events:

    $ bin/eventsim -c "examples/site.json" --from 365 --nusers 30000 --growth-rate 0.30 data/fake.json

Config File
===========
Take a look at the sample config file. It's a JSON file, with key-value pairs.  Here is an explanation of the values
(many of which match command line options):

* `seed` For the pseudo-random number generator. Changing this value will change the output (all other parameters
being equal).
* `alpha` See above
* `beta` See above
* `damping` Controls the depth of daily cycles
* `weekend-damping` Controls the difference between weekday and weekend traffic volume
* `weekend-damping-offset` Controls when the weekend/holiday starts (relative to midnight), in minutes
* `weeeknd-damping-scale` Controls how long traffic tapering lasts, in minutes
* `session-gap` Minimum time between sessions, in seconds
* `start-date` Start date for data (in ISO8601 format)
* `end-date` End date for data (in ISO8601 format)
* `n-users` Number of users at start-date
* `first-user-id` User id assigned to first user (these are assigned sequentially)
* `growth-rate` Annual growth rate for users
* `tag` Tag added to each line of the output

Building huge data sets in parallel
===================================
You can run multiple instances of this application simultaneously if you need to generate a lot of data very quickly.
To do this, we recommend the following strategy:

* Use a different random seed for each instance. This assures that each instance will produce different data.
* Use a different first user id for each instance. This assures that each instance will produce data with different
(and not overlapping) user ids.
* Create a set of configuration files, one for each instance of eventsim. This will help you re-create your data,
and help you remember the details of the data
* Do not generate data for different time periods. The generator doesn't generate full sessions if they cross the start
and end dates; you will find some incomplete data between files

More tricks
===========

Some more ideas on how to use the simulator

* To simulate A/B tests, crate multiple data sets for the same time period with different sets of member ids, different
tags, and different  parameters for alpha, beta, transition probabilities, or growth rates.

Future Work
===========

We haven't made the generator multi-threaded yet, but there isn't a good reason that we can't do that. (The only
state that needs to be shared between threads is the priority queue, and access to that can be easily controlled).

About the source data
=====================

The results of this simulation are fake... but they are based on real data. (We thought that using real data on
songs would make the simulation more colorful and interesting.)

The song data is from the Million Song Dataset, official website by Thierry Bertin-Mahieux,
available at: http://labrosa.ee.columbia.edu/millionsong/. For more information, see this paper:

> Thierry Bertin-Mahieux, Daniel P.W. Ellis, Brian Whitman, and Paul Lamere.
> The Million Song Dataset. In Proceedings of the 12th International Society
> for Music Information Retrieval Conference (ISMIR 2011), 2011.

The last names come from the US Census Bureau (see http://www.census.gov/genealogy/www/data/2000surnames/index.html).

The first names come from the Social Security Administration (see http://www.ssa.gov/oact/babynames/#&ht=1); we
took the top 1000 names for each sex from this site.

(Note that the first and last names are chosen independently. This leads to some unexpected, but awesome results.)

Location names are from the Census Bureau (see https://www.census.gov/popest/data/datasets.html).

User agents are from http://techblog.willshouse.com/2012/01/03/most-common-user-agents/