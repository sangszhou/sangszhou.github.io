---
layout: post
title: akka actor
categories: [akka]
description: akka
keywords: akka, actor
---

## actor lifecycle

![](/images/posts/scala/actor_lifecycle1.png)

All actors are supervised, i.e. linked to another actor with a fault handling strategy. 
Actors may be restarted in case an exception is thrown while processing a message (see Supervision and Monitoring). 
This restart involves the hooks mentioned above:

The old actor is informed by calling preRestart with the exception which caused the restart and the 
message which triggered that exception; the latter may be None if the restart was not caused by processing 
a message, e.g. when a supervisor does not trap the exception and is restarted in turn by its supervisor, 
or if an actor is restarted due to a sibling’s failure. If the message is available, then that 
message’s sender is also accessible in the usual way (i.e. by calling sender).

This method is the best place for cleaning up, preparing hand-over to the fresh actor instance, etc. 
By default it stops all children and calls postStop.

The initial factory from the actorOf call is used to produce the fresh instance.

The new actor’s postRestart method is invoked with the exception which caused the restart. 
By default the preStart is called, just as in the normal start-up case.

An actor restart replaces only the actual actor object; the contents of the mailbox is unaffected by 
the restart, so processing of messages will resume after the postRestart hook returns. The message that 
triggered the exception will not be received again. Any message sent to an actor while it is being restarted 
will be queued to its mailbox as usual.







