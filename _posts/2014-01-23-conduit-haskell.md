---
layout: post
title: Example of streaming data from the database using Persistent and Conduit libraries in Haskell
---

# {{ page.title }}
At work, I have to deal with increasingly larger datasets. And by large, I mean a PostgreSQL table with over 7 millions rows of data. If I really wanted to, I could probably be able to load it all into RAM...maybe, I haven't even tried as it just sounds silly, and especially given the fact that I have quite a few more than one monstrous table like that to deal with. At the same time, I would also like to do as much processing in parallel as possible (you know, put those multi-core CPUs and/or GPUs into action). In short, big data problem.

This problem has made me look into Haskell yet again. After all, Haskell is meant to make it much easier for the programmer to parallelise their algorithms. [This book by Simon Marlow](http://chimera.labs.oreilly.com/books/1230000000929/index.html) is well worth a look when it comes to parallel programming in Haskell, by the way.

There are quite a few packages that make it easy (or _easier_) to interface with a database in Haskell. Personally, I've decided to give [Persistent](http://www.yesodweb.com/book/persistent) a shot. Persistent is the default database interface used by [Yesod Web Framework](http://www.yesodweb.com), works with the majority of popular databases (SQLite, PostgreSQL, MySQL, etc.), and is quite well documented.

In this post, I will provide a (hopefully informative) example of how to use Persistent together with Conduit library. [Conduit](https://www.fpcomplete.com/user/snoyberg/library-documentation/conduit-overview) is a Haskell library that allows streaming data (not necessarily from a database) in constant memory. In other words, with Conduit, it is possible to pull into memory just enough data for processing and not more, thus minimising unnecessary memory use.

## The problem
Suppose we are given a database with one massive table of integers, and our objective is to print the records as pairwise tuples to screen. That is, if `xs :: [Int]` is the list of all the records, we want to zip it with its tail and print it to the screen:

{% highlight haskell %}

Prelude> mapM_ (putStrLn . show) $ zip xs $ tail xs

{% endhighlight %}

## Create test SQLite database
Firstly, let's define the structure of the table using Persistent. Persistent relies on Template Haskell to specify the structure of the database tables. For example, the following code (the source code for all the snippets in this post can be found on [GitHub](https://github.com/kubkon/conduit-persistent-example)):

{% highlight haskell %}

-- specs.hs
{-# LANGUAGE EmptyDataDecls    #-}
{-# LANGUAGE FlexibleContexts  #-}
{-# LANGUAGE GADTs             #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}

module Specs where

import qualified Database.Persist.TH as TH

TH.share [TH.mkPersist TH.sqlSettings, TH.mkMigrate "migrateAll"] [TH.persistLowerCase|
MyRecord
  value Int
  deriving Show
|]

{% endhighlight %}

generates the following SQL:

{% highlight sql %}

CREATE TABLE "my_record" ("id" INTEGER PRIMARY KEY, "value" INTEGER NOT NULL);

{% endhighlight %}

Having specified the table structure, let's go ahead and create an SQLite database with a single table with 100000 integers starting from 1 and incrementing by 1:

{% highlight haskell %}

-- createDB.hs
{-# LANGUAGE OverloadedStrings #-}

import Database.Persist (insertMany)
import Database.Persist.Sqlite (runSqlite,runMigration)
import qualified Specs as S

main :: IO ()
main = runSqlite "test.sqlite" $ do

  -- Create test DB and populate with values
  let n = 100000
  runMigration S.migrateAll
  insertMany $ map S.MyRecord [1..n]

  return ()

{% endhighlight %}

When compiled and executed, this code creates a dummy SQLite database "test.sqlite" with a table consisting of 100000 rows of integers.

## Print all records as pairs using traditional list approach
In order to query the database for all the records, Persistent provides a helper function, `selectList :: [Filter val] -> [SelectOpt val] -> m [Key val]`. [This function](http://hackage.haskell.org/package/persistent-1.3.0.2/docs/Database-Persist-Class.html) takes two lists containing search criteria as arguments: a list of SQL filter constraints (`WHERE`), and a list of other `SELECT` options (e.g., `ASC` or `LIMIT`). The result is a list enclosed in a monad `m` of all the records matching the search criteria. Therefore, in order for us to get all the records from our dummy database, it suffices to call the function with two empty lists:

{% highlight haskell %}

records <- selectList [] []

{% endhighlight %}

All that remains to be done, is to extract the integer values from the list of records into a new list, zip the resultant list with its tail, and print the result to screen. That is,

{% highlight haskell %}

let values = map (myRecordValue . entityVal) records
mapM_ (liftIO . putStrLn . show) $ zip values $ tail values

{% endhighlight %}

You'll notice three odd functions used in here: `myRecordValue`, `entityVal` and `liftIO`. The first one, `myRecordValue` is inferred by Template Haskell and represents the constructor of `MyRecord` data type. If you recall the definition of the SQL table used in this example:

{% highlight haskell %}

TH.share [TH.mkPersist TH.sqlSettings, TH.mkMigrate "migrateAll"] [TH.persistLowerCase|
MyRecord
  value Int
  deriving Show
|]

{% endhighlight %}

Persistent, behind the scenes, translates this template into Haskell data type of the following structure:

{% highlight haskell %}

data MyRecord = MyRecord
  { myRecordValue :: !Int
  }
  deriving (Show, Read, Eq)

{% endhighlight %}

Thus, we can use `myRecordValue` to access the underlying `Int` value from `MyRecord` data type.

The `entityVal` function, on the other hand, is a constructor for `Entity` data type, and can be used to access the record data type that's stored within it. That is, Persistent wraps up every record received from the database in `Entity` data type. Therefore, in our example, as `selectList` returns a list `[Entity MyRecord]` (wrapped in a monad), it is necessary to first recover `MyRecord` from the `Entity`, and that can be accomplished via the use of `entityVal` function.

Lastly, `liftIO` function is used to lift a computation from the `IO` monad. In our example, we use it to lift (or complete) an internal `IO`-type computation used by Persistent, and return back to the standard `IO` monad.

The full code looks like this:

{% highlight haskell %}

-- selectList.hs
{-# LANGUAGE OverloadedStrings #-}

import Control.Monad (mapM_)
import Control.Monad.IO.Class  (liftIO)
import Database.Persist (selectList,entityVal)
import Database.Persist.Sqlite (runSqlite)
import Specs (myRecordValue)

main :: IO ()
main = runSqlite "test.sqlite" $ do

    -- Select all records from DB and unpack into a list of Ints
    records <- selectList [] []
    let values = map (myRecordValue . entityVal) records
    mapM_ (liftIO . putStrLn . show) $ zip values $ tail values

{% endhighlight %}

If we now run the compiled version of the code with the statistics flag `-s`; that is,

{% highlight console %}

$ ./selectList +RTS -s

{% endhighlight %}

the following output will be generated:

{% highlight console %}

   1,242,454,680 bytes allocated in the heap
      71,087,496 bytes copied during GC
      15,219,032 bytes maximum residency (7 sample(s))
       1,563,864 bytes maximum slop
              35 MB total memory in use (0 MB lost due to fragmentation)

                                    Tot time (elapsed)  Avg pause  Max pause
  Gen  0      2379 colls,     0 par    0.05s    0.05s     0.0000s    0.0008s
  Gen  1         7 colls,     0 par    0.03s    0.04s     0.0059s    0.0181s

  INIT    time    0.00s  (  0.00s elapsed)
  MUT     time    0.97s  (  1.24s elapsed)
  GC      time    0.08s  (  0.10s elapsed)
  EXIT    time    0.00s  (  0.00s elapsed)
  Total   time    1.05s  (  1.34s elapsed)

  %GC     time       7.6%  (7.1% elapsed)

  Alloc rate    1,277,517,423 bytes per MUT second

  Productivity  92.4% of total user, 72.7% of total elapsed

{% endhighlight %}

Notice the total memory used of 35 MB.

## Print all records as pairs using Conduit approach
Now, let's do the same, but this time, let's use the Conduit library to stream the data from the database. In order to stream the data, we need to have a Conduit `Source`. Persistent provides a helper function that is much like `selectList` but returns a `Source` rather than a list. It's not difficult to guess its name which is `selectSource`. [The function](http://hackage.haskell.org/package/persistent-1.3.0.2/docs/Database-Persist-Class.html) has a signature `selectSource :: [Filter val] -> [SelectOpt val] -> Source m (Entity val)`.

Having acquired the `Source` of our data, we now need to extract the underlying `Int` from the input data type `Entity MyRecord`, combine the extracted integers into pairwise tuples, convert the tuples into their `String` representation, and finally, print them to screen. 

In order to extract `Int`s from the upstream, we can use `Data.Conduit.List.map` helper function. [This function](http://hackage.haskell.org/package/conduit-1.0.12/docs/Data-Conduit-List.html) applies the supplied function to the data provided by the upstream. In our case, `selectSource` returns data of type `Entity MyRecord`, therefore, we can use a composition of `myRecordValue` with `entityVal` to extract the underlying `Int` from input. That is,

{% highlight haskell %}

import qualified Data.Conduit.List as CL

entityToValue :: Monad m => Conduit (Entity MyRecord) m Int
entityToValue = CL.map (myRecordValue . entityVal)

{% endhighlight %}

Note that the type of `entityToValue` function is `Conduit (Entity MyRecord) m Int`. This can be read as: for every input of type `Entity MyRecord` I, the `Conduit`, will do some processing and pass an `Int` forward to the downstream (for other `Conduit`s or a `Sink` to receive).

The next thing to do is to combine the `Int`s into pairs, and cast them into `String`. In order to achieve this, we can use the `await`, `yield` and `leftover` helper functions provided by `Data.Conduit` package. The idea is thus to `await` for two consecutive `Int`s, wrap them in a tuple, convert it to `String`, `yield` it back to the downstream, and pass the second of the `Int`s (a `leftover`) back to the upstream. In code, that would go like this:

{% highlight haskell %}

showPairs :: Monad m => Conduit Int m String
showPairs = do
  mi1 <- await -- get the next value from the input stream
  mi2 <- await
  case (mi1, mi2) of
    (Just i1, Just i2) -> do
      yield $ show (i1, i2) -- pass tuple of Ints converted
                            -- to String downstream
      leftover i2           -- pass the second component of
                            -- the tuple back to itself (to
                            -- the upstream)
      showPairs
    _ -> return ()

{% endhighlight %}

Finally, all that remains is to print each tuple to screen, and terminate the flow of the data stream. To accomplish this, we'll use the `Data.Conduit.List.mapM_` which applies a monadic action to all values in the stream. It's not difficult to guess that the action we are going to apply is printing the tuples to screen. That is,

{% highlight haskell %}

printString :: (Monad m, MonadIO m) => Sink String m ()
printString = CL.mapM_ (liftIO . putStrLn)

{% endhighlight %}

Having all the components defined, we can link them into the following data stream which will result in the desired outcome:

{% highlight haskell %}

selectSource [] [] $$ entityToValue =$ showPairs =$ printString

{% endhighlight %}

The full code looks like this:

{% highlight haskell %}

-- selectSource.hs
{-# LANGUAGE OverloadedStrings #-}

import Control.Monad.IO.Class  (MonadIO,liftIO)
import Data.Conduit (await,yield,leftover,Conduit,Sink,($$),(=$))
import qualified Data.Conduit.List as CL
import Database.Persist (selectSource,entityVal,Entity)
import Database.Persist.Sqlite (runSqlite)
import Specs (MyRecord(..),myRecordValue)

-- |Unpacks incoming values from upstream from MyRecord to Int
entityToValue :: Monad m => Conduit (Entity MyRecord) m Int
entityToValue = CL.map (myRecordValue . entityVal)

-- |Converts pairwise tuples of Ints into String
showPairs :: Monad m => Conduit Int m String
showPairs = do
  mi1 <- await -- get the next value from the input stream
  mi2 <- await
  case (mi1, mi2) of
    (Just i1, Just i2) -> do
      yield $ show (i1, i2) -- pass tuple of Ints converted
                            -- to String downstream
      leftover i2           -- pass the second component of
                            -- the tuple back to itself (to
                            -- the upstream)
      showPairs
    _ -> return ()

-- |Prints input String
printString :: (Monad m, MonadIO m) => Sink String m ()
printString = CL.mapM_ (liftIO . putStrLn)

main :: IO ()
main = runSqlite "test.sqlite" $ do

  -- Select all records from DB and return them as a Source
  selectSource [] [] $$ entityToValue =$ showPairs =$ printString

{% endhighlight %}

If we now run the compiled version of the code with a statistics flag `-s`; that is,

{% highlight console %}

$ ./selectSource +RTS -s

{% endhighlight %}

the following output will be generated:

{% highlight console %}

   3,152,019,584 bytes allocated in the heap
       7,109,552 bytes copied during GC
          68,928 bytes maximum residency (2 sample(s))
          21,120 bytes maximum slop
               1 MB total memory in use (0 MB lost due to fragmentation)

                                    Tot time (elapsed)  Avg pause  Max pause
  Gen  0      6041 colls,     0 par    0.03s    0.04s     0.0000s    0.0000s
  Gen  1         2 colls,     0 par    0.00s    0.00s     0.0003s    0.0006s

  INIT    time    0.00s  (  0.00s elapsed)
  MUT     time    2.70s  (  3.18s elapsed)
  GC      time    0.03s  (  0.04s elapsed)
  EXIT    time    0.00s  (  0.00s elapsed)
  Total   time    2.74s  (  3.22s elapsed)

  %GC     time       1.3%  (1.2% elapsed)

  Alloc rate    1,167,680,199 bytes per MUT second

  Productivity  98.7% of total user, 83.8% of total elapsed

{% endhighlight %}

Note that the total memory used equals only 1 MB compared to 35 MB when using the list approach! I don't know about you, but I'm impressed.
