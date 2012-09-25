# stweam

![tweety](https://github.com/poptip/stweam/raw/master/tweety.gif)

This module contains several data structures that ease working with twitter streams. In particular, a site stream pool is available that auto manages several site streams as needed and respects twitter's request limits. Includes stream reconnecting with exponential backoff to keep your streams running forever.

# Features

* Limits requests to a set amount per second to avoid being rate limited.
* Reconnect streams on timeout, errors, or disconnects.
* Back off linearly or exponentially depending on error type when reconnecting.
* Respect Twitter's limits when adding users to a site stream.
* Make sure users are added to site streams with an extra request.
* If there is an HTTP error adding a user to a site stream, retries with exponential timeouts.
* Automatically manage a pool of site streams creating new streams as needed.
* Convenient events emitted for various tweet types.
* Uses a streaming JSON parser for faster less memory hogging processing.

# Usage

```js
var Stweam = require('stweam');

var client = new Stweam({
  consumer_key: 'twitter',
  consumer_secret: 'API',
  access_token_key: 'keys',
  access_token_secret: 'go here'
});
var pool = client.createPool();

pool.addUser(...);

pool.on('tweet', function(data) {
  console.log('someone tweeted!');
  console.log(data);
});

pool.on('reply', function(data) {
  console.log('someone replied to a tweet on site streams');
  console.log(data);
});
```

# API

* [new Stweam(credentials)](#new-stweam)
  * [Event: 'connect'](#event-connect)
  * [Event: 'beforeConnect'](#event-beforeconnect)
  * [Event: 'reconnect'](#event-reconnect)
  * [Event: 'disconnect'](#event-disconnect)
  * [Event: 'destroy'](#event-destroy)
  * [Event: 'timeout'](#event-timeout)
  * [Event: 'retry'](#event-retry)
  * [Event: 'retryMax'](#event-retrymax)
  * [Event: 'data'](#event-data)
  * [Event: 'warning'](#event-warning)
  * [Event: 'delete'](#event-delete)
  * [Event: 'scrub_geo'](#event-scrub_geo)
  * [Event: 'limit'](#event-limit)
  * [Event: 'status_withheld'](#event-status_withheld)
  * [Event: 'user_withheld'](#event-user_withheld)
  * [Event: 'tweet'](#event-tweet)
  * [Event: 'tweet:retweet'](#event-tweetretweet)
  * [Event: 'tweet:retweet:`retweeted_status.id`'](#event-tweetretweetretweeted_status_id)
  * [Event: 'tweet:reply'](#event-tweetreply)
  * [Event: 'tweet:reply:`in_reply_to_status_id`'](#event-tweetreplyin_reply_to_status_id)
  * [Event: 'tweet:@reply'](#event-tweetreply-1)
  * [Event: 'tweet:@reply:`in_reply_to_screen_name`'](#event-tweetreplyin_reply_to_screen_name)
* [Stweam#createPublicStream([parameters])](#stweamcreatepublicstreamparameters)
* [Stweam#createSampleStream([parameters])](#stweamcreatesamplestreamparameters)
* [Stweam#createFirehose([parameters])](#stweamcreatefirehoseparameters)
* [Stweam#createUserStream([parameters])](#stweamcreateuserstreamparameters)
  * [Event: 'friends'](#event-friends)
  * [Event: 'block'](#event-block)
  * [Event: 'unblock'](#event-unblock)
  * [Event: 'favorite'](#event-favorite)
  * [Event: 'unfavorite'](#event-unfavorite)
  * [Event: 'follow'](#event-follow)
  * [Event: 'list_create'](#event-list_create)
  * [Event: 'list_destroy'](#event-list_destroy)
  * [Event: 'list_update'](#event-list_update)
  * [Event: 'list_member_add'](#event-list_member_add)
  * [Event: 'list_member_remove'](#event-list_member_remove)
  * [Event: 'list_user_subscribe'](#event-list_user_subscribe)
  * [Event: 'list_user_unsubscribe'](#event-list_user_unsubscribe)
  * [Event: 'user_update'](#event-user_update)
* [Stweam#createSiteStream([follow], [parameters])](#stweamcreatesitestreamfollow-parameters)
  * [SiteStream#addUser(twitterID)](#sitestreamaddusertwitterid)
  * [SiteStream#addUsers(twitterIDs)](#sitestreamadduserstwitterids)
  * [SiteStream#removeUser(twitterID, [callback(err)])](#sitestreamremoveusertwitterid-callbackerr)
  * [SiteStream#userCount()](#sitestreamusercount)
  * [SiteStream#hasUser(twitterID)](#sitestreamhasusertwitterid)
  * [SiteStream#info(callback(err, info))](#sitestreaminfocallbackerr-info)
  * [Event: 'addUsers'](#event-addusers)
  * [Event: 'failedToAdd'](#event-failedtoadd)
  * [Event: 'removeUser'](#event-removeuser)
  * [Event: 'token revoked'](#event-token-revoked)
* [Stweam#createPool(options)](#stweamcreatepooloptions)
  * [Pool#addUser(twitterID, queue)](#pooladdusertwitterid-queue)
  * [Pool#addUsers(twitterIDs)](#pooladduserstwitterids)
  * [Pool#removeUser(twitterID)](#poolremoveusertwitterid)
  * [Pool#hasUser(twitterID)](#poolhasusertwitterid)
  * [Pool#simulate(tweet, [twitterID])](#poolsimulatetweet-twitterid)

### new Stweam(credentials)

First argument must be an object with oauth credentials.

```js
{
  consumer_key: 'twitter',
  consumer_secret: 'API',
  access_token_key: 'keys',
  access_token_secret: 'go here'
}
```

Stweam currently extends the [ntwitter](https://github.com/AvianFlu/ntwitter) library. All methods added are specifically designed to facilitate usage of Twitter streams.

All streams have a `connected` key indicating if the stream is currently connected. Constructors last argument can be an object of parameters. [See here](https://dev.twitter.com/docs/api/2/get/user) for a list of parameters and their descriptions.

### Events
All streams emit the following events.

### Event: 'connect'

Stream has connected. This is also emitted if it's a reconnection.

### Event: 'beforeConnect'

Emitted right before a connection is made.

### Event: 'reconnect'

Emitted right before a reconnection is made.

### Event: 'disconnect'

Stream was disconnected.

### Event: 'destroy'

Stream was destroyed.

### Event: 'timeout'

Stream has timed out and will be reconnecting soon.

### Event: 'retry'
* `string` - Algorithm. Either `linear` or `exponential`.
* `string` - Method.
* `number` - Milliseconds until next retry.
* `number` - Amount of attempts the method has been retried without success.
* `number` - Total number of retries.

There was an error calling a method. This indicates the method will be called again.

### Event: 'retryMax'
* `string` - Algorithm. Either `linear` or `exponential`.
* `string` - Method. So far only `reconnect` and `addUsers` are retried.

This is emitted if the maximum retry is reached for a method with a certain algorithm. At which point you can pause the stream or whatever until you figure out what went wrong.

### Event: 'data'
* `Object` - Contains all of the data from the message received.

This event is emitted every time a full JSON object is received from twitter. Useful for debugging.

### Event: 'warning'
* `string` - Code, ex: `FALLING_BEHIND`.
* `string` - Message.
* `number` - Queue full percentage.

Emitted when the client is falling behind on reading tweets. Will occur a maximum of about once in 5 minutes. It really should not happen because this is node, come on.

### Event: 'delete'
* `number` - ID of status to delete.
* `number` - User for which status belongs to.

These messages indicate that a given Tweet has been deleted. Client code must honor these messages by clearing the referenced Tweet from memory and any storage or archive, even in the rare case where a deletion message arrives earlier in the stream that the Tweet it references.

### Event: 'scrub_geo'
* `string` - The ID of the user.
* `string` - Up to which status to scrub geo info from.

These messages indicate that geolocated data must be stripped from a range of Tweets. Clients must honor these messages by deleting geocoded data from Tweets which fall before the given status ID and belong to the specified user. These messages may also arrive before a Tweet which falls into the specified range, although this is rare.

### Event: 'limit'
* `number` - Number of undelivered tweets.

These messages indicate that a filtered stream has matched more Tweets than its current rate limit allows to be delivered. Limit notices contain a total count of the number of undelivered Tweets since the connection was opened, making them useful for tracking counts of track terms, for example. Note that the counts do not specify which filter predicates undelivered messages matched.

### Event: 'status_withheld'
* `string` - ID of status.
* `string` - ID of user.
* `Array.string` - An array of two-letter countries codes. Example: `['de', 'ar']`

Indicates a status has been withheld in certain countries.

### Event: 'user_withheld'
* `string` - ID of user.
* `Array.string` - An array of two-letter country codes.

Indicates a user has been withheld in certain countries.

### Event: 'tweet'
* [tweet](#tweet)

Someone tweets!

### Event: 'tweet:retweet'
* [tweet](#tweet)

Someone retweeted a tweet. Only emitted when it's a retweet through the API and not a manual retweet ie `RT: hello world`.

### Event: 'tweet:retweet:`retweeted_status.id`'
* [tweet](#tweet)

Convenient event for listening for retweets of a certain tweet.

### Event: 'tweet:reply'
* [tweet](#tweet)

Someone replied to a tweet.

### Event: 'tweet:reply:`in_reply_to_status_id`'
* [tweet](#tweet)

Convenient event for listening for replies of a certain tweet.

### Event: 'tweet:@reply'
* [tweet](#tweet)

Someone replied to another user. Emitted even if the user is mentioned manually.

### Event: 'tweet:@reply:`in_reply_to_screen_name`'
* [tweet](#tweet)

Convenient event for listening for replies to a certain user.


### Stweam#createPublicStream([parameters])
Create an instance of a public stream.


### Stweam#createSampleStream([parameters])
Create an instance of a sample stream. Emits random sample of public statuses.


### Stweam#createFirehose([parameters])
Create an instance of a firehose. Emits all public tweets. Requires special permission to use.


### Stweam#createUserStream([parameters])
Create an instance of a user stream. [See here](https://dev.twitter.com/docs/api/2/get/user) for a list of parameters.

### Event: 'friends'
* `Array.string` - An array of user IDs.

Upon establishing a User Stream connection, Twitter will send a preamble before starting regular message delivery. This preamble contains a list of the user's friends. 

### Event: 'block'
* [user](#user) - Current user.
* [user](#user) - Blocked user.
* `Date` - Created at date.

User blocks someone.

### Event: 'unblock'
* [user](#user) - Current user.
* [user](#user) - Blocked user.
* `Date` - Created at date.

User removes a block.

### Event: 'favorite'
* [user](#user) - User that favorited the tweet.
* [user](#user) - Author of the tweet.
* [tweet](#tweet) - Favorited tweet.
* `Date` - Created at date.

User favorites a tweet.

### Event: 'unfavorite'
* [user](#user) - User that favorited the tweet.
* [user](#user) - Author of the tweet.
* [tweet](#tweet) - Favorited tweet.
* `Date` - Created at date.

User unfavorites a tweet.

### Event: 'follow'
* [user](#user) - Following user.
* [user](#user) - Followed user.
* `Date` - Created at date.

User follows someone.

### Event: 'list_create'
* `list`
* `Date` - Created at date.

User creates a list.

### Event: 'list_destroy'
* `list`
* `Date` - Created at date.

User deletes a list.

### Event: 'list_update'
* `list`
* `Date` - Created at date.

User edits a list.

### Event: 'list_member_add'
* [user](#user) - Adding user.
* [user](#user) - Added user.
* `list`
* `Date` - Created at date.

User adds someone to a list.

### Event: 'list_member_remove'
* [user](#user) - Removing user.
* [user](#user) - Removed user.
* `list`
* `Date` - Created at date.

User removes someone from a list.

### Event: 'list_user_subscribe'
* [user](#user) - Subscribing user.
* [user](#user) - List owner.
* `list`
* `Date` - Created at date.

User subscribes to a list.

### Event: 'list_user_unsubscribe'
* [user](#user) - Unsubscribing user.
* [user](#user) - List owner.
* `list`
* `Date` - Created at date.

User unsubscribes from a list.

### Event: 'user_update'
* [user](#user) - New profile data.
* `Date` - Created at date.

User updates their profile.


### Stweam#createSiteStream([follow], [parameters])
Create an instance of a site stream. `follow` can be an Array of twitter IDs to initially add to the stream when it first connects. If `follow` has more users than the allowed users to connect with, they will be queued to be added later. [See here](https://dev.twitter.com/docs/api/2b/get/site) for a list of parameters. Access is restricted.

### SiteStream#addUser(twitterID)
Add a user to the stream.

### SiteStream#addUsers(twitterIDs)
Add several users to the stream.

### SiteStream#removeUser(twitterID, [callback(err)])
Remove a user from the stream.

### SiteStream#userCount()
Returns the number of users in stream, including the number of queued users that are going to be added to the stream.

### SiteStream#hasUser(twitterID)
Returns true if user is in site stream.

### SiteStream#info(callback(err, info))
Gets information about the stream from twitter. Sample:

```js
{
    "info":
    {
        "users":
        [
            {
                "id":119476949,
                "name":"oauth_dancer",
                "dm":false
            }
        ],
        "delimited":"none",
        "include_followings_activity":false,
        "include_user_changes":false,
        "replies":"none",
        "with":"user"
    }
}
```

### Events
Site streams receive the same events as user streams. But for multiple users instead of one. To identify which user an event belongs to, each event includes a user's ID as the first argument. Except for the events: `connect', `reconnect`, `disconnect`, `destroy`, `retry`, `retryMax`, `data`, and `tweet`. 

For example, the event `friends` would be emitted like this: `function (userid, friends) { }`.

In addition, an event with the user's Twittter ID appended to the event name is emitted. For user with an ID of `1234` the event `friends:1234` would be emitted with `friends` as the first argument.

### Event: 'addUsers'
* `Array.string` - Array of user IDs.
* `Object` - Contains user IDs as keys with their respective screen names as values.

When a batch of users are successfully added to this site stream. Users are checked that they've been actually added using the `SiteStream#info()` method.

### Event: 'failedToAdd'
* `Array.string` - Array of user IDs.

If there was a user did not show up in the `SiteStream#info()` call after sending a request to add that user, they will show up here.

### Event: 'removeUser'
* `string` - User ID.

After a call to `SiteStream#remove()`, either this or an `error` event will be emitted, even if a callback was given to the method.

### Event: 'token revoked'
* `string` - Code of the event.
* `string` - Stream name.

A user has revoked access to your app.


### Stweam#createPool([options])
Creates an instance of a site stream pool. Automatically creates and removes site streams as needed respecting twitter's request demands. `client` must be an instance of ntwitter. Options defaults to

```js
{
  // amount of time to wait when adding users in order to add more
  // together in one request
  addUserTimeout: 1000
}
```

### Pool#addUser(twitterID, queue)
Adds a user to the pool. Set `queue` to true if you want to queue this user to be added after `options.addUserTimeout` in case there might be more users in the queue later. This saves making unnecessary requests to twitter.

### Pool#addUsers(twitterIDs)
Add several users to the pool at once.

### Pool#removeUser(twitterID)
Remove a user from pool.

### Pool#hasUser(twitterID)
Returns true if user has been added to pool.

### Pool#simulate(tweet, [twitterID])
Simulates a tweet coming in from twitter. Useful if you get tweets through the REST API and want to emit it through the pool.

### Events
Pool instances are proxied all events from all underlying site stream instances.


# Response Objects

Several events emit different response objects. Here you'll find examples of what they look like. [Go here](https://dev.twitter.com/docs/platform-objects "twitter objects") to read more details about each field in the objects.

### [User](https://dev.twitter.com/docs/platform-objects/users)

```js
{
  "id" : 92572086,
  "screen_name" : "StudyTravelNL",
  "verified" : false,
  "profile_sidebar_fill_color" : "ffffff",
  "location" : "Nijmegen, Nederland",
  "statuses_count" : 481,
  "default_profile" : false,
  "contributors_enabled" : false,
  "following" : true,
  "geo_enabled" : true,
  "profile_background_color" : "610e44",
  "utc_offset" : null,
  "time_zone" : null,
  "name" : "StudyTravel Talen",
  "show_all_inline_media" : false,
  "listed_count" : 2,
  "protected" : false,
  "profile_background_image_url" : "http://a0.twimg.com/profile_background_images/398762728/Brochure_2012-2013.JPG",
  "friends_count" : 101,
  "notifications" : false,
  "profile_link_color" : "82b7f0",
  "is_translator" : false,
  "profile_use_background_image" : false,
  "description" : "Een taal beleven is meer dan studeren. Je verdiept je in de taal, cultuur en gewoonten van een ander land. Wie wil er niet op zo'n 'spraakmakende' taalreis!",
  "profile_text_color" : "08090a",
  "profile_image_url_https" : "https://si0.twimg.com/profile_images/1225384939/StudyTravel_Taalreizen_NL_normal.JPG",
  "id_str" : "92572086",
  "profile_background_image_url_https" : "https://si0.twimg.com/profile_background_images/398762728/Brochure_2012-2013.JPG",
  "default_profile_image" : false,
  "profile_image_url" : "http://a0.twimg.com/profile_images/1225384939/StudyTravel_Taalreizen_NL_normal.JPG",
  "follow_request_sent" : false,
  "lang" : "en",
  "profile_sidebar_border_color" : "08090a",
  "favourites_count" : 1,
  "followers_count" : 121,
  "url" : "http://www.studytravel.nl",
  "created_at" : "Wed Nov 25 17:41:07 +0000 2009",
  "profile_background_tile" : false
}
```

### [Tweet](https://dev.twitter.com/docs/platform-objects/tweets)

```js
{
  "entities" : {
    "user_mentions" : [
      {
        "screen_name" : "espenotienenick",
        "indices" : [0, 16],
        "id_str" : "186170312",
        "name" : "Esperanza Was There",
        "id" : 186170312
      }
    ],
    "hashtags" : [ ],
    "urls" : [ ]
  },
  "in_reply_to_user_id" : 186170312,
  "in_reply_to_status_id" : 217924341025869820,
  "text" : "@espenotienenick Yo, de lo blanca que estoy, cuando me meto en el mar, parezco una linterna. jejeje",
  "in_reply_to_user_id_str" : "186170312",
  "place" : null,
  "retweeted" : false,
  "in_reply_to_screen_name" : "espenotienenick",
  "truncated" : false,
  "retweet_count" : 0,
  "source" : "<a href=\"http://www.tweetdeck.com\" rel=\"nofollow\">TweetDeck</a>",
  "coordinates" : null,
  "geo" : null,
  "id_str" : "217924715484946432",
  "contributors" : null,
  "favorited" : false,
  "created_at" : "Wed Jun 27 10:17:55 +0000 2012",
  "user" : {
    "default_profile" : false,
    "lang" : "en",
    "profile_background_tile" : false,
    "screen_name" : "mandolinaes",
    "is_translator" : false,
    "show_all_inline_media" : false,
    "profile_sidebar_fill_color" : "ffeedc",
    "profile_image_url_https" : "https://si0.twimg.com/profile_images/1517593937/Bcubbins_small_normal.jpg",
    "following" : null,
    "profile_sidebar_border_color" : "ffeedc",
    "description" : "Stranger in a strange land. Out of this world. Mathematician. Follow your dreams no matter what. Jared Leto is my inspiration in life http://www.mandolinaes.com",
    "default_profile_image" : false,
    "profile_use_background_image" : true,
    "favourites_count" : 74,
    "friends_count" : 221,
    "id_str" : "23666389",
    "created_at" : "Tue Mar 10 21:59:13 +0000 2009",
    "profile_text_color" : "500601",
    "profile_background_image_url_https" : "https://si0.twimg.com/profile_background_images/545298383/tumblr_bea.jpg",
    "profile_background_image_url" : "http://a0.twimg.com/profile_background_images/545298383/tumblr_bea.jpg",
    "time_zone" : "Madrid",
    "followers_count" : 1064,
    "protected" : false,
    "url" : "http://mandolinaes.tumblr.com/",
    "profile_image_url" : "http://a0.twimg.com/profile_images/1517593937/Bcubbins_small_normal.jpg",
    "profile_link_color" : "910c00",
    "name" : "Beatriz",
    "listed_count" : 53,
    "contributors_enabled" : false,
    "geo_enabled" : false,
    "id" : 23666389,
    "follow_request_sent" : null,
    "statuses_count" : 52122,
    "verified" : false,
    "notifications" : null,
    "utc_offset" : 3600,
    "profile_background_color" : "070d1a",
    "location" : "Spain"
  },
  "id" : 217924715484946430,
  "in_reply_to_status_id_str" : "217924341025869824"
}
```


# Tests
Tests are written with [mocha](http://visionmedia.github.com/mocha/)

```bash
npm test
```

# Links
* https://dev.twitter.com/docs/streaming-apis
* https://dev.twitter.com/docs/streaming-apis/connecting
* https://dev.twitter.com/docs/streaming-apis/messages
* https://dev.twitter.com/docs/streaming-apis/parameters
* https://dev.twitter.com/docs/platform-objects
* https://dev.twitter.com/docs/twitter-ids-json-and-snowflake
