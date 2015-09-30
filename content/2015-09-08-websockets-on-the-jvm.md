title: WebSockets on the JVM
date: Wed September 30 2015
author: Andrew Smith

We had been tossing around the idea of using WebSockets in our app for quite some time now and with the lingering bonus of getting rid of IE 9 we decided to pull the trigger with our notification system.

## Research

There are lots and lots of different implementations of websockets out there across countless languages. Our criteria was that we wanted them to run in the same webapp container as the rest of our product. That being said we needed something to run on the JVM. Another big push our frontend dudes were wanting is something that could use socket.io as the client. This proved a little more frustrating because it was paired with its own backend quite heavily and I didnt want to implement all of our datasources in node.js. After researching many different options I settled on [Atmosphere](https://github.com/Atmosphere/atmosphere) because its extreme portability to several different JVM containers and its socket.io support.

## Installation

After adding the artifacts to our build process I was able to stand up their "chat" example pretty painlessly. Now for the fun part, adding our own custom broadcasters and warping it to do what we were needing for our application.

The task that we had chosen WebSockets as our architecture to replace was our notifications that users receive for events that happen elsewhere in the system (ie things completing in a background task, new modules enabled/disabled for users). Our current implementation of this was a rest call that would "long poll" (Thread.sleep for a second at a time before checking again for a max of ~30s) our datasource until it found something it needed.  One main concern I had with this was we were dedicating one of our HTTP worker threads to every user which doesn't scale as well as one would hope.  With the WebSocket implementation we would free up that worker thread to do other things and also have a dedicated channel to our user we with to push notifications to.

Once we had our webserver successfully making persistent connections with clients the real fun started. We needed to choose a medium that we could send messages across JVMs without getting into another "long poll" situation hopefully. Luckily Atmosphere came with many [extentions](https://github.com/Atmosphere/atmosphere-extensions) jms, jedis, jgroups, and hazelcast just to name a few. Redis and Hazelcast are two tools that we already use quite heavily in our environment. Because Hazelcast gives our sysops guy nightmares at night I decided to choose Redis for our inter JVM pubsub communication.

After developing our broadcasters using Atmospheres Redis example I had things running very well and broadcasting messages from many different JVMs.

## Testing
After a few iterations of frontend updates around our old notification code the WebSocket implementation went live in a QA environment and everything broke down. Publishes were happening on our redis instance and nothing was picking them up. Some users would get messages and some would not. Frustrating to say the least. After a few code updates to put in massive amounts of debug logging I finally started to get a picture of what was going on.  The Jedis.subscribe method was a blocking method, this didn't allow the broadcaster to properly finish subscribing the client. I had blindly trusted a 3rd party example of connecting to another resource without cross referencing a few other examples of how that resource says it should be connecting to.

After researching some Jedis pubsub examples and looking at the HazelcastBroadcaster implementation I finally had a better idea on how subscribing to rooms within Redis.

I extracted the JedisPubSub annoymous class into its own class in their example and then started the subscribe in a new thread.

```
this.subscriber = new JedisSubscriber(RedisUtil.this.callback);
		new Thread() {
			@Override
			public void run() {
				try {
					RedisUtil.this.subscribeClient.subscribe(RedisUtil.this.subscriber, RedisUtil.this.callback.getID());
					LOG.debug("Subscription ended for " + RedisUtil.this.callback.getID());

				} catch (Throwable t) {
					LOG.error("Error subscribing to redis channel", t);
				}
				RedisUtil.this.callback.subscriptionEnded();
			};
		}.start();
```

 When this subscription ended (or threw an exception) I would then trigger a method on the Broadcaster callback that checked to see if I still had any resources still listening and if so re-subscribe.
 
 ```
	private void refreshMessageListener() {
		synchronized (this.redisUtil) {
			if (getAtmosphereResources().size() > 0 && !this.redisUtil.isSubscriberConnected()) {
				this.redisUtil.startSubscriber();

				LOG.debug("Started redis subscriber on room " + getID());
			} else if (getAtmosphereResources().size() == 0) {
				this.redisUtil.stopSubscriber();

				LOG.debug("Stoped redis subscriber on room " + getID());
			} else if (LOG.isDebugEnabled()) {
				LOG.debug("Did not add message listener because resources:" + getAtmosphereResources().size());
			}
		}
	}
```

This same `refreshMessageListener` also got called any time an atmosphere resource was added or removed via the `addAtmosphereResource` and `removeAtmosphereResource` methods causing the subscription to never be stagnant or get tore down if no clients were wanting messages from that room.

## Result
After these changes to our Broadcaster/Pubsub mediums things went much smoother.  It was pretty awesome watching things happen in completely different JVMs and on completion they showed up immediately to clients. Where does this lead us in the road ahead? Having records come into a back-channel datasource and appearing in the clients view as soon as they do? Live feeds of email campaigns as they stream through our system? Being able to watch a singular campaign stream in with the volume steadily increasing as we process more data from it? With this model the possibilities are only limited by the dev time we wish to invest.

## Retrospect
In hindsight the main thing I learned by this experience was to cross reference 3rd parties samples of talking to other datasources before taking them as gold. Most of the time these are just examples on how to start your implementation and go from there. If I would have done this in the beginning the mistakes would have been obvious and the QA/debugging process would have been much less painful.







