---
layout: post
title: may
---

Going to try to keep some log (if I remember). Some of this for future things down the road. 

kafka-confluent is pretty neat (... maybe). But not really that straightforward. For one, confluent talks about not one 
but in fact, multiple sets of clis. There's the cli that is part of the <a href="https://docs.confluent.io/platform/current/tutorials/examples/clients/docs/kafka-commands.html">platform</a>.
Then there's also the <a href="https://docs.confluent.io/confluent-cli/current/index.html">confluent cli</a>. Let's
also not forget the cloud cli. The point is, there are many of these, and they don't all do the same things, e.g., 
if you want to use zookeeper addresses to list kafka topics, you're actually gonna want the platform CLI.

What also isn't that straightforward is how performant it might actually be given less than ideal conditions. Which you 
might somehow be yoked to. Let's say you have a topic dumped into a single partition, which means by design, you are
restricted to using a single consumer in a consumer group. Ok, upsetting, but no biggie, single partition is 
fine. Then upsize that topic so that this single partition handles billions of updates a day. Then add an overly 
restrictive retention period of, say, perhaps an hour or two because the size (or the time) was set to be very 
restrictive. This is like a very stupid hard mode but it was what was given to you and by god you're going to deal it 
(or realize the answer like months down the line, but this is long after the fact). The paradigm for kafka 
is to use poll(), and possibly poll(0), but it's impossible to keep up on a single message call, 
as it turns out. Just testing a single poll for a day got me to half a billion at its finest. During burst periods, it
can't keep up. consume() comes in, and consume _can_ keep up, if you're peeling off the queue in very large chunks.

If you've had the misfortune here to have your data stored as avro, you'll now deal with deserialization separately
as consume() does not deserialize the messages (and DeserializingConsumer currently does not implement consume()). This 
part was .... really not that straightforward how to implement if you have a given client registry. Implemented an
awkward af hack from <a href="https://stackoverflow.com/questions/44407780/how-to-decode-deserialize-avro-with-python-from-kafka">here</a>
after registering the rest url as a client, looking up the subject names and getting the right one to have 
it spit out the schema. And then you can pass this string to your homemade deserializer (or anything else really).

```
from confluent_kafka.schema_registry import SchemaRegistryClient

client = SchemaRegistryClient({'url': 'resturl_blah'})
print(client.get_subjects())

# for the schema itself
client.get_latest_version('subject_name').schema

# given the subject name, also the subject id if you don't happen to know it
client.get_latest_version('subject_name').schema_id

# for the schema string
client.get_latest_version('subject_name').schema.schema_str
```

AvroDeserializer as of 1.7.0 requires not only the client, but you need the schema in string 
format (it is not an optional argument, as it turns out, or maybe the registry client just didn't really work to make
it optional, idk, this was a long day). But it oddly won't accept the schema_str as passed by the SchemaRegistryClient,
or it didn't in my case. 

Interestingly enough, in the above situation, you can keep a consume() going up indefinitely (as long as an EOF isn't 
passed), whereas you'll end up having a poll() die. I still haven't worked out what exactly is going on, presumably
some internal queue and retention scheme doing peculiar things.
