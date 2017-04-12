# Inner join

## Initial database State

### Author table

+-----+---------+-----------+
|id   |fname    |lname      |
+-----+---------+-----------+
|1    |Simon    |PJ         |
+-----+---------+-----------+
|2    |Erik     |Meijer     |
+-----+---------+-----------+

### Book table

+-----+-------+--------+
|id   |title  |isbn    |
+-----+-------+--------+
|1    |title1 |dkfa1   |
+-----+-------+--------+
|2    |title2 |2kdfja  |
|     |       |        |
+-----+-------+--------+

### AuthorBook table

+-----+-----------+----------+
|id   |authorId   | bookId   |
+-----+-----------+----------+
|1    |1          |1         |
+-----+-----------+----------+
|2    |1          |2         |
+-----+-----------+----------+
|3    |2          |1         |
+-----+-----------+----------+
|3    |2          |2         |
+-----+-----------+----------+


## Code

``` haskell
#!/usr/bin/env stack
{- stack
     --resolver lts-6.24
     --install-ghc
     runghc
     --package yesod
     --package blaze-html
     --package persistent
     --package text
     --package bytestring
     --package persistent-postgresql
     --package persistent-template
     --package esqueleto
     --package monad-logger
     --package mtl
-}

{-# LANGUAGE EmptyDataDecls             #-}
{-# LANGUAGE FlexibleContexts           #-}
{-# LANGUAGE GADTs                      #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses      #-}
{-# LANGUAGE OverloadedStrings          #-}
{-# LANGUAGE QuasiQuotes                #-}
{-# LANGUAGE TemplateHaskell            #-}
{-# LANGUAGE TypeFamilies               #-}
import           Control.Monad.IO.Class  (liftIO)
import qualified Database.Esqueleto as E
import Database.Esqueleto (LeftOuterJoin)
import           Database.Persist
import           Database.Persist.Postgresql
import           Database.Persist.TH
import Control.Monad.Logger
import Control.Monad.Reader

share [mkPersist sqlSettings, mkMigrate "migrateAll"] [persistLowerCase|
Person
    name String
    age Int Maybe
    deriving Show
BlogPost
    title String
    authorId PersonId
    deriving Show
|]

loadData :: MonadIO m => ReaderT SqlBackend m ()
loadData = do
  p1 <- insert $ Person "Sitaram" (Just 26)
  p2 <- insert $ Person "Rajesh" (Just 30)

  insert_ $ BlogPost "test" p1
  insert_ $ BlogPost "test 2" p1


connStr = "host=localhost dbname=test user=postgres password=postgres port=5432"

testQuery :: MonadIO m => ReaderT SqlBackend m [(Entity Person, Maybe (Entity BlogPost))]
testQuery = E.select $ E.from $ \(p `E.LeftOuterJoin` b ) -> do
              E.on (E.just (p E.^. PersonId) E.==. b E.?. BlogPostAuthorId)
              E.where_ $ (p E.^. PersonId) `E.in_` E.valList [toSqlKey 1]
              return (p,b)


main :: IO ()
main =
  runStderrLoggingT $
  withPostgresqlPool connStr 10 $
  \pool ->
     liftIO $
     do flip runSqlPersistMPool pool $
          do runMigration migrateAll
             -- loadData
             dat <- testQuery
             liftIO $ mapM_ print dat >> putStrLn ""
             return ()
```

## Output:

``` shellsession
[Debug#SQL] SELECT "person"."id", "person"."name", "person"."age", "blog_post"."id", "blog_post"."title", "blog_post"."author_id"
FROM "person" LEFT OUTER JOIN "blog_post" ON "person"."id" = "blog_post"."author_id"
WHERE "person"."id" IN (?)
; [PersistInt64 1]

(Entity {entityKey = PersonKey {unPersonKey = SqlBackendKey {unSqlBackendKey = 1}}, entityVal = Person {personName = "Sitaram", personAge = Just 26}},Just (Entity {entityKey = BlogPostKey {unBlogPostKey = SqlBackendKey {unSqlBackendKey = 1}}, entityVal = BlogPost {blogPostTitle = "test", blogPostAuthorId = PersonKey {unPersonKey = SqlBackendKey {unSqlBackendKey = 1}}}}))
(Entity {entityKey = PersonKey {unPersonKey = SqlBackendKey {unSqlBackendKey = 1}}, entityVal = Person {personName = "Sitaram", personAge = Just 26}},Just (Entity {entityKey = BlogPostKey {unBlogPostKey = SqlBackendKey {unSqlBackendKey = 2}}, entityVal = BlogPost {blogPostTitle = "test 2", blogPostAuthorId = PersonKey {unPersonKey = SqlBackendKey {unSqlBackendKey = 1}}}}))
```

Try changing the where clause to match on `2` in the above code and you will get this output:

``` shellsession
[Debug#SQL] SELECT "person"."id", "person"."name", "person"."age", "blog_post"."id", "blog_post"."title", "blog_post"."author_id"
FROM "person" LEFT OUTER JOIN "blog_post" ON "person"."id" = "blog_post"."author_id"
WHERE "person"."id" IN (?)
; [PersistInt64 2]

(Entity {entityKey = PersonKey {unPersonKey = SqlBackendKey {unSqlBackendKey = 2}}, entityVal = Person {personName = "Rajesh", personAge = Just 30}},Nothing)
```
