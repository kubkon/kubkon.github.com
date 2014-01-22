---
layout: post
title: Conduit with Persistent example in Haskell
---

# {{ page.title }}
At work, I have to deal with increasingly larger datasets. By large, I mean a PostgreSQL table with over 7 millions rows of data. I could probably push it and be able to load it all into RAM...maybe, I haven't even tried as it just sounds silly, and especially given the fact that I have quite a few more than one monstrous table like that to deal with. At the same time, I would also like to do as much processing in parallel as possible (you know, put those multi-core CPUs and/or GPUs into action). In short, big data problem.

This problem has made me look into Haskell yet again. After all, Haskell is meant to make it much easier for the programmer to implement their algorithms in parallel. [This book by Simon Marlow](http://chimera.labs.oreilly.com/books/1230000000929/index.html) is well worth a look at when it comes to parallel programming in Haskell, by the way.

There are quite a few packages that make it easy (or _easier_) to interface with a database in Haskell. I've decided to give [Persistent](http://www.yesodweb.com/book/persistent) a shot. Persistent is the default database interface used by [Yesod Web Framework](http://www.yesodweb.com), works with the majority of popular databases (SQLite, PostgreSQL, MySQL, etc.), and is quite well documented.

## Create test SQLite database

{% highlight haskell %}

-- createDB.hs
{-# LANGUAGE EmptyDataDecls    #-}
{-# LANGUAGE FlexibleContexts  #-}
{-# LANGUAGE GADTs             #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}

import Database.Persist
import Database.Persist.Sqlite
import Database.Persist.TH

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
MyRecord
  value Int
  deriving Show
|]

main :: IO ()
main = runSqlite "test.sqlite" $ do

  -- Create test DB and populate with values
  let n = 100000
  runMigration migrateAll
  insertMany $ map MyRecord [1..n]

  return ()

{% endhighlight %}

{% highlight console %}

$ ghc createDB.hs
$ ./createDB

{% endhighlight %}

## Print all records as pairs using traditional list approach

{% highlight haskell %}

-- selectList.hs
{-# LANGUAGE EmptyDataDecls    #-}
{-# LANGUAGE FlexibleContexts  #-}
{-# LANGUAGE GADTs             #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}

import Control.Monad (mapM_)
import Control.Monad.IO.Class  (liftIO)
import Data.Conduit
import qualified Data.Conduit.List as CL
import Database.Persist
import Database.Persist.Sqlite
import Database.Persist.TH

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
MyRecord
  value Int
  deriving Show
|]

main :: IO ()
main = runSqlite "test.sqlite" $ do

    -- Select all records from DB and unpack into a list of Ints
    records <- selectList [] []
    let values = map (myRecordValue . entityVal) records
    mapM_ (liftIO . putStrLn . show) $ zip values $ tail values

{% endhighlight %}

{% highlight console %}

$ ghc selectList.hs -rtsopts
$ ./selectList +RTS -s

{% endhighlight %}

{% highlight console %}

   2,484,819,856 bytes allocated in the heap
     143,509,032 bytes copied during GC
      30,422,416 bytes maximum residency (8 sample(s))
       3,174,208 bytes maximum slop
              68 MB total memory in use (0 MB lost due to fragmentation)

                                    Tot time (elapsed)  Avg pause  Max pause
  Gen  0      4763 colls,     0 par    0.09s    0.09s     0.0000s    0.0010s
  Gen  1         8 colls,     0 par    0.06s    0.11s     0.0133s    0.0390s

  INIT    time    0.00s  (  0.00s elapsed)
  MUT     time    1.92s  (  2.43s elapsed)
  GC      time    0.15s  (  0.20s elapsed)
  EXIT    time    0.00s  (  0.00s elapsed)
  Total   time    2.07s  (  2.64s elapsed)

  %GC     time       7.1%  (7.6% elapsed)

  Alloc rate    1,292,298,464 bytes per MUT second

  Productivity  92.9% of total user, 73.0% of total elapsed

{% endhighlight %}

## Print all records as pairs using Conduit approach

{% highlight haskell %}

-- selectSource.hs
{-# LANGUAGE EmptyDataDecls    #-}
{-# LANGUAGE FlexibleContexts  #-}
{-# LANGUAGE GADTs             #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE QuasiQuotes       #-}
{-# LANGUAGE TemplateHaskell   #-}
{-# LANGUAGE TypeFamilies      #-}

import Control.Monad.IO.Class  (liftIO)
import Data.Conduit
import qualified Data.Conduit.List as CL
import Database.Persist
import Database.Persist.Sqlite
import Database.Persist.TH

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
MyRecord
  value Int
  deriving Show
|]

entityToValue :: Monad m => Conduit (Entity MyRecord) m Int
entityToValue = CL.map (myRecordValue . entityVal)

printPairs :: Monad m => Conduit Int m String
printPairs = do
  mi1 <- await
  mi2 <- await
  case (mi1, mi2) of
    (Just i1, Just i2) -> do
      yield $ show (i1, i2)
      leftover i2
      printPairs
    _ -> return ()

main :: IO ()
main = runSqlite "test.sqlite" $ do

    -- Select all records from DB and return them as a Source
    selectSource [] [] $$
      entityToValue =$
      printPairs =$
      CL.mapM_ (liftIO . putStrLn)

{% endhighlight %}

{% highlight console %}

$ ghc selectSource.hs -rtsopts
$ ./selectSource +RTS -s

{% endhighlight %}

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

