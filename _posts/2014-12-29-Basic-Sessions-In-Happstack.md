---
layout: post_page
title:  "Basic Sessions in Happstack"
date:   2014-12-29 22:04:06
categories: haskell
comments: true
---

I recently started getting into Happstack recently and realized that there isn't a easy redis backed session store available, so I decided to build one as a way to learn more about Happstack and Haskell in general.

We will be using the Hedis package for making the redis calls as well as the Aeson package for serializing the session data.

{% highlight haskell %}

{-# LANGUAGE OverloadedStrings,DeriveGeneric #-}
import Database.Redis as Redis
import Data.Aeson
import Data.Either(either)
import Data.Maybe(maybe)
import System.Random(newStdGen, randomRs)
import Control.Monad(msum)
import Control.Monad.IO.Class(liftIO)
import Happstack.Server
main :: IO ()
main = do
	redisConn <- Redis.connect Redis.defaultConnectInfo
	simpleHTTP nullConf $ routeHandlers redisConn

{% endhighlight %}

With the imports and language definitions out of the way, we can focus on defining the `routeHandlers` which is where our application will be defined

{% highlight haskell %}

routeHandlers :: Redis.Connection -> ServerPartT IO Response
routeHandlers redisConn = msum [withSessCookie redisConn, withoutSessCookie redisConn]

{% endhighlight %}

The idea is that we will call `withSessCookie`. If there is no cookie present, then it will fail and try the `withoutSessCookie`

{% highlight haskell %}

withSessCookie :: Redis.Connection -> ServerPartT IO Response
withSessCookie redisConn = do
	-- This will fail if the cookie does not exist and fail through to the withoutSessCookie
	sessID <- lookCookieValue "_sess"
	sess   <- liftIO $ getSessData redisConn sessID
	if auth sess then
		ok $ toResponse "You are logged in!"
	else
		seeOther "/login" $ toResponse ()

withoutSessCookie :: Redis.Connection -> ServerPartT IO Response
withoutSessCookie redisConn = do
	newSessID <- liftIO $ genSessID
	addCookie Session (mkCookie "_sess" newSessID)
	liftIO $ setSessData redisConn (mkEmptySess newSessID)
	seeOther "/login" $ toResponse ()

{% endhighlight %}

Before we get any further, we need to define a type for the session data. In this, I will only be using a simple counter but you can keep track of anything as long as it is something that is an instance of `FromJSON` and `ToJSON`.

{% highlight haskell %}

data Session = Session {sessID :: ByteString, count :: Int} deriving (Generic)
-- We auto derive the json serialization
instance FromJSON Session
instance ToJSON Session

mkEmptySess :: ByteString -> Session
mkEmptySess sessID = Session sessID 0

{% endhighlight %}

Now that we have a type for the session, we can define the getters and setters for the sessions.

{% highlight haskell %}

getSessData :: Redis.Connection -> ByteString -> IO Session
getSessData redisConn sessID = do
	reply <- Redis.runRedis redisConn $ Redis.get ("sess:" ++ sessID)
	either (\_ -> emptySess) (\rawSess -> maybe emptySess decodeSess rawSess) reply
	where
	emptySess :: IO Session
	emptySess = return $ mkEmptySess sessID
	decodeSess :: ByteString -> IO Session
	decodeSess rawSess = maybe emptySess (\sess -> return sess) (decode' rawSess)

setSessData :: Redis.Connection -> Session -> IO ()
setSessData redisConn sess@(Session sessID _) = do
	Redis.runRedis redisConn $ do
		_ <- Redis.psetex ("sess:"++sessID) sessionLife (encode sess)
		return ()
	where
	-- Session life is given in ms. We set it to 1 hour as a default here.
	sessionLife = 1000 * 60 * 60

{% endhighlight %}

Finally, we can top it off with the utility function that generates the random session ids.

{% highlight haskell %}

genSessID :: IO ByteString
genSessID = do
	gen <- newStdGen
	return . (take 20) . (randomRs ('a', 'z')) $ gen

{% endhighlight %}

While this code is does work for me, I think that a lot can be done to improve it in terms of how well it adheres to idiomatic haskell. I will be on the lookout for doing this more elegantly and update this post when I do.
