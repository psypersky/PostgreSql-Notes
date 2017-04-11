# Postgresql full text search recipes

## Full text search on several fields on the same table

__Description__   
Storing the processed ts_vectors on a field on the same row record, then we create a GIN index on that field, this allow us to make super fast full search queries combining several fields of the table, since we save the processed ts_vector result we need to update the vector on each insert/update on the table.

__Advantages__   
Fast and simple.

__Disadvantages__   
Just fields on the same table, if we want to combine tables to do full search we need to find another solution.

__Source__   
https://blog.lateral.io/2015/05/full-text-search-in-milliseconds-with-postgresql/

```sql
-- Add the field to the table
ALTER TABLE shots ADD COLUMN tsv_search tsvector;

-- Create a GIN index on that field
CREATE INDEX tsv_idx_search ON shots USING gin(tsv_search);

-- Manually update the gin index, this may take a while the first time
UPDATE shots SET tsv_search = setweight(to_tsvector(title), 'A') || setweight(to_tsvector(coalesce(description,'')), 'C');

-- This procedure will update out ts_vector on each insert or update
CREATE FUNCTION shots_search_trigger() RETURNS trigger AS $$
begin
  new.tsv_search :=
    setweight(to_tsvector(new.title), 'A') || setweight(to_tsvector(coalesce(new.description,'')), 'C');
  return new;
end
$$ LANGUAGE plpgsql;

-- Create the trigger
CREATE TRIGGER tsvectorupdate BEFORE INSERT OR UPDATE
ON shots FOR EACH ROW EXECUTE PROCEDURE shots_search_trigger();

-- Search super fastly
SELECT title
FROM (
  SELECT title, tsv_search
  FROM shots
  WHERE tsv_search @@ to_tsquery('tud')
) AS s_search
ORDER BY ts_rank_cd(s_search.tsv_search, to_tsquery('tud')) DESC
LIMIT 5;
```


## Full text search on several fields on different tables
__Description__   
In order to do full search on fields of different tables we create a query that joins the tables and stores a ts_vector in order to create an index in that ts_vector

__Advantages__   
Multi table full text search yo!

__Disadvantages__   
Need to trigger refresh for materialized view each x time, you need to consider if this is acceptable in your app design

__Source__   
http://rachbelaid.com/postgres-full-text-search-is-good-enough/

```sql
-- Create a materialized view to store our post-user info and the ts_vector
CREATE MATERIALIZED VIEW search_index AS
SELECT
  s.id,
  s.user_id,
  s.created,
  s.title,
  s.description,
  s.game_id,
  s.army_id,
  s.bucket_id,
  s.url,
  s.image,
  s.attachments,
  s.likes_count,
  s.views_count,
  u.username,
  u.image AS user_image,
    setweight(to_tsvector(s.title), 'A') ||
    setweight(to_tsvector(coalesce(s.description, '')), 'C') ||
    setweight(to_tsvector(u.username), 'B')
  as document
FROM shots as s
LEFT JOIN users as u ON (u.id = s.user_id);

-- Index our materialized view on the ts_vector
CREATE INDEX idx_fts_search ON search_index USING gin(document);

-- Hyper blast multi-column full text search
SELECT title, username
FROM search_index
WHERE document @@ to_tsquery('tud')
ORDER BY ts_rank(document, to_tsquery('tud')) DESC
LIMIT 5;
```


## Other sources
https://www.compose.com/articles/indexing-for-full-text-search-in-postgresql/
https://robots.thoughtbot.com/implementing-multi-table-full-text-search-with-postgres
https://www.postgresql.org/docs/9.3/static/textsearch.html
