+++
date = '2025-08-08T19:14:25+10:00'
draft = false 
title = 'Building a Lightning-Fast Address Search with SQLite and FTS5'
+++

## Intro

Ever needed to lookup your address when you're ordering something online?
It's pretty common, I thought it would be cool to implement an address lookup
database in SQLite3 to get a better understanding of full-text search.

### The Dataset we will use

The Australian Government release a geocoded address dataset for the whole of
Australia with quarterly updates. As quoted from
[data.gov.au](https://data.gov.au/data/dataset/geocoded-national-address-file-g-naf):

> Geoscape G-NAF is the geocoded address database for Australian businesses and
> governments. It’s the trusted source of geocoded address data for Australia
> with over 50 million contributed addresses distilled into 15.4 million G-NAF
> addresses.

The Geoscape Geocoded National Address File (G-NAF), not to be mistaken for
FNAF (Five Nights at Freddy's). Is basically a zip file with a bunch of PSV
files supplied by each state in Australia with physical addresses.

This type of dataset and use-case (address auto completion) is a perfect task
for SQLite3 and the 
[FTS5 extension](https://www.sqlite.org/fts5.html), which adds full-text search
functionality to the database engine.

### Shoutouts

Thankfully I am not the first person to ingest the G-NAF dataset into SQLite,
a huge thanks to Peter's blog post on
[Exploring G-NAF with SQLite](https://geocode.earth/blog/2021/exploring-gnaf-with-sqlite/)
and this [GitHub Repo](https://github.com/ggotti/g-naf-full-sqlite) by Gerard
which contains ingestion scripts, which saved me a lot of time ingesting the
data into SQLite.

The first couple of sections of this post is a quick rehash taken from the
people above, included here for completeness so you can easily follow along.
You can skip to my full-text search addition
[here](#full-text-search-with-sqlite3s-fts5-extension).

## Tools you'll need

You need SQLite3 installed with the FTS5 extension enabled. Some Linux
distributions come with it already enabled, you can check by running this command:
```
$ sqlite3 :memory: "PRAGMA compile_options;" | grep ENABLE_FTS5
```
If you're lucky you'll see `ENABLE_FTS5` as output from the above command. If
not, you may need to
[compile and install SQLite3](https://sqlite.org/howtocompile.html) from
source. Notably passing `--enable-fts5` to `./configure`.

You'll also need the following tools

* `unzip`
* `curl`
* `jq` (not required but helps with downloading the dataset)

## Ingesting the G-NAF into SQLite3

Alright let's get this data into SQLite!

### Download the Dataset

Use this command to get the latest G-NAF dataset URL from data.gov.au:

```
$ curl -s -o- https://data.gov.au/data/api/3/action/package_show?id=19432f89-dc3a-4ef3-b943-5326ef1dbecc | jq -r '.result.resources[].url | select(contains("gda2020"))'
```

Once we have that URL we can then download the dataset, it's almost 2GB:

```
$ curl -o gnaf.zip 'https://data.gov.au/data/dataset/19432f89-dc3a-4ef3-b943-5326ef1dbecc/resource/fa2f2db3-c7ad-4bc7-b93b-1f6415e180fa/download/g-naf_may25_allstates_gda2020_psv_1019.zip'
```

Finally we'll extract the dataset, it's about 8GB uncompressed:

```
$ unzip gnaf.zip -d gnaf
```

The structure of the dataset looks like this:

```
$ tree gnaf
G-NAF
├── Extras
│   ├── GNAF_TableCreation_Scripts
│   │   ├── add_fk_constraints.sql
│   │   ├── create_tables_ansi.sql
│   │   ├── create_tables_sqlserver.sql
│   │   └── README.txt
│   └── GNAF_View_Scripts
│       ├── address_view.sql
│       └── README.txt
└── G-NAF MAY 2025
    ├── Authority Code
    │   ├── Authority_Code_ADDRESS_ALIAS_TYPE_AUT_psv.psv
    │   ├── Authority_Code_ADDRESS_CHANGE_TYPE_AUT_psv.psv
    │   ├── Authority_Code_ADDRESS_TYPE_AUT_psv.psv
    │   ├── Authority_Code_FLAT_TYPE_AUT_psv.psv
    │   ├── Authority_Code_GEOCODED_LEVEL_TYPE_AUT_psv.psv
    │   ├── Authority_Code_GEOCODE_RELIABILITY_AUT_psv.psv
    │   ├── Authority_Code_GEOCODE_TYPE_AUT_psv.psv
    │   ├── Authority_Code_LEVEL_TYPE_AUT_psv.psv
    │   ├── Authority_Code_LOCALITY_ALIAS_TYPE_AUT_psv.psv
    │   ├── Authority_Code_LOCALITY_CLASS_AUT_psv.psv
    │   ├── Authority_Code_MB_MATCH_CODE_AUT_psv.psv
    │   ├── Authority_Code_PS_JOIN_TYPE_AUT_psv.psv
    │   ├── Authority_Code_STREET_CLASS_AUT_psv.psv
    │   ├── Authority_Code_STREET_LOCALITY_ALIAS_TYPE_AUT_psv.psv
    │   ├── Authority_Code_STREET_SUFFIX_AUT_psv.psv
    │   └── Authority_Code_STREET_TYPE_AUT_psv.psv
    └── Standard
        ├── ACT_ADDRESS_ALIAS_psv.psv
        ├── ACT_ADDRESS_DEFAULT_GEOCODE_psv.psv
        ├── ACT_ADDRESS_DETAIL_psv.psv
        ├── ACT_ADDRESS_FEATURE_psv.psv
        ├── ACT_ADDRESS_MESH_BLOCK_2016_psv.psv
        ├── ACT_ADDRESS_MESH_BLOCK_2021_psv.psv
        ├── ACT_ADDRESS_SITE_GEOCODE_psv.psv
        ├── ACT_ADDRESS_SITE_psv.psv
        ├── ACT_LOCALITY_ALIAS_psv.psv
        ├── ACT_LOCALITY_NEIGHBOUR_psv.psv
        ├── ACT_LOCALITY_POINT_psv.psv
        ├── ACT_LOCALITY_psv.psv
        ├── ACT_MB_2016_psv.psv
        ├── ACT_MB_2021_psv.psv
        ├── ACT_PRIMARY_SECONDARY_psv.psv
        ├── ACT_STATE_psv.psv
        ├── ACT_STREET_LOCALITY_ALIAS_psv.psv
        ├── ACT_STREET_LOCALITY_POINT_psv.psv
        ├── ACT_STREET_LOCALITY_psv.psv
        ...
```

### Create the SQLite Schema

Thankfully we don't need to create this ourselves, you may have spotted above
that the dataset actually includes an ANSI SQL schema which is compatible with
SQLite3, even better Gerard has gone through this schema by hand and added all
the appropriate constraints to all the tables. You can download that 
[here](https://raw.githubusercontent.com/ggotti/g-naf-full-sqlite/refs/heads/main/ddl.sql).

Once we have that schema, we'll then apply it to our new SQLite3 database
`gnaf.db`.

```
$ sqlite3 gnaf.db < gnaf_schema.sql
```

Once created you'll have something that looks like this
([credit](https://docs.geoscape.com.au/projects/gnaf_desc/en/stable/appendix_b.html)):

![G-NAF ORM](/posts/fast-gnaf-lookups-with-sqlite3-fts5/gnaf_orm.png)

Once last thing, the dataset also includes a stored query which makes it a
little easier to lookup addresses in the database. The stored query supplied by
the dataset doesn't work right out of the box with SQLite as the `OR REPLACE`
syntax is not supported. You can remove it or just download
[this](https://raw.githubusercontent.com/ggotti/g-naf-full-sqlite/refs/heads/main/addressView.sql).

Add it to the database:

```
$ sqlite3 gnaf.db < address_view.sql
```

### Importing the G-NAF dataset into SQLite3

First we'll import the Authority Code files into the database, these files 
contain constant definitions like the street types `STREET`, `COURT`, `ROAD`
and many others.

```
$ find "gnaf/G-NAF MAY 2025/Authority Code" -name "*.psv" -print0 | \
while IFS= read -r -d '' line; do
  fileName=`basename -- "$line"`
  tableName=`echo "$fileName" | sed -n 's/Authority_Code_\(.*\)_psv.psv/\1/p'`

  (
    echo ".mode csv"
    echo ".separator \"|\""
    echo ".import \"$line\" $tableName --skip 1"
  ) | sqlite3 gnaf.db 
done

```

This will go through all of the PSV files in the Authority Code directory and
generate SQLite3 compatible CSV import statements, feeding them directly to the
database.

We'll also do the same for the Standard directory, this holds the physical
address data for all the Australian states and will take some time to complete.

```
$ find "gnaf/G-NAF MAY 2025/Standard" -name "*.psv" -print0 | \
while IFS= read -r -d '' line; do
  fileName=`basename -- "$line"`
  tableName=`echo "$fileName" | sed -n 's/^\(WA\|NT\|VIC\|NSW\|SA\|TAS\|OT\|ACT\|QLD\)_\(.*\)_psv.psv$/\2/p'`

  (
    echo ".mode csv"
    echo ".separator \"|\""
    echo ".import \"$line\" $tableName --skip 1"
  ) | sqlite3 gnaf.db 
done
```

Finally let's do some cleanup of the database, we'll convert empty values in
the database tables to NULL values, this helps keep our queries clean. Grab
[this script](https://raw.githubusercontent.com/ggotti/g-naf-full-sqlite/refs/heads/main/nullCleanup.sql) and run it with:

```
sqlite3 gnaf.db < null_cleanup.sql
```

Once the command completes you'll now have the G-NAF dataset imported into your
SQLite3 database. At the moment it weighs in at around 8GB in size.

But there's a problem... It's slow, really slow, try this
([credit](https://geocode.earth/blog/2021/exploring-gnaf-with-sqlite/#querying-the-database)):
```
$ sqlite3 -cmd ".timer on" gnaf.db <<SQL

  SELECT COUNT(*)
  FROM ADDRESS_VIEW
  WHERE STREET_LOCALITY_PID = 'VIC1930629'
SQL
```
Eventually the query completes and we see how long it took, in my case it
took a minute to run...
```
Run Time: real 54.790 user 24.049264 sys 28.518008
```

That's not going to cut it. Let's create some index's to speed up data
retrieval, we'll create indices on anything ending in `_pid` or `_code`.
Run this command to generate the required SQL.

```
$ sqlite3 gnaf.db <<'EOF' > gnaf_indices.sql
    SELECT printf(
      'CREATE INDEX IF NOT EXISTS %s ON %s (%s);',
      LOWER(t.name) || '_' || LOWER(c.name),
      t.name,
      c.name
    )
    FROM sqlite_master t
    LEFT OUTER JOIN pragma_table_info(t.name) c
    WHERE t.type = 'table'
      AND (
        c.name LIKE '%_pid' ESCAPE '\' OR
        c.name LIKE '%_code' ESCAPE '\' OR
        c.name = 'code'
      );
EOF
```

We'll then add these indices to the database. This will take some time to
complete and double the databases size. As Peter mentioned in his blog post,
not all of the indices are required but using the script above is simple.

```
$ sqlite3 gnaf.db < gnaf_indices.sql
```

If we run the same select query from before we see it now only takes 62
milliseconds to complete, much better than 54 seconds:

```
Run Time: real 0.062 user 0.014118 sys 0.021947
```

## Full-Text Search with SQLite3's FTS5 Extension

Now that we have the G-NAF dataset ingested into our SQLite3 database, let's
get full-text search working using the SQLite3 FTS5 extension.

From the [FTS5 extension documentation](https://www.sqlite.org/fts5.html):
> FTS5 is an SQLite virtual table module that provides full-text search
> functionality to database applications. In their most elementary form,
> full-text search engines allow the user to efficiently search a large
> collection of documents for the subset that contain one or more instances of
> a search term.

### Create the Full-Text Search Table

Lets define the FTS5 virtual table for our full-text search queries:

```
$ sqlite3 gnaf.db <<SQL
    CREATE VIRTUAL TABLE ADDRESS_FTS USING fts5(                                    
      id UNINDEXED,                                                                 
      display,                                                                      
      tokenize='unicode61',                                                         
    );
SQL
```

In this FTS5 virtual table we define an `id` column that won't be indexed for
full-text search. Instead, we will be using this column to store a reference to the
`ADDRESS_DETAIL_PID` from the `ADDRESS_VIEW` stored query. This is so we can
easily gather more data about an address when we need to, such as retriving the
latitude and longitude of the address to pin on a map for example.

We also define a `display` column, this is what we will be conducting full-text
searches on, as well as what we display. Another common approach is having a
separate unindexed `formatted` column which stores a much prettier formatted 
address that is shown to users.

Finally we define our tokeniser, in this case __unicode61__.

FTS5 has numerous tokeniser implementations that are suitable for different
search workloads, from the docs:


> * The __porter__ tokenizer, which implements the
> [porter stemming algorithm](https://tartarus.org/martin/PorterStemmer/).
> * The __trigram__ tokenizer, which treats each contiguous sequence of three
> characters as a token, allowing FTS5 to support more general substring
> matching. 
> * The __unicode61__ tokenizer, based on the Unicode 6.1 standard. This is the
> default.
> * The __ascii__ tokenizer, which assumes all characters outside of the ASCII 
> codepoint range (0-127) are to be treated as token characters.

We don't need to use the __porter__ tokeniser
as there is no need to reduce words to there base form, for example _fishing_
→ _fish_ since our vocab is small compared to other datasets such as Wikipedia.

__trigram__ is pretty useless to us too as we don't want to create token's of
three characters in length.

For our use-case which is searching physical addresses, the __unicode61__ tokeniser
is the most appropriate for our needs, it tokenises on whitespace which is
perfect for our address format we will be using in our `display` column, so
lets define that, here's a simple one:

```
<NUMBER> <STREET_NAME> <STREET_TYPE_CODE> <LOCALITY> <STATE> <POST_CODE>
```

Using the __unicode61__ tokeniser the following address:
```
442 KINGSFORD SMITH DRIVE HAMILTON QLD 4007
```
will get tokenised as the following list:

1. `422`
1. `KINGSFORD`
1. `SMITH`
1. `DRIVE`
1. `HAMILTON`
1. `QLD`
1. `4007`

In my research this could be optimised, we actually shouldn't split on each
word, in the above example address the `STREET_NAME` is two words 
`KINGSFORD SMITH`. It would be more search efficient to merge the two words, so 
queries like `kingsford s*` quickly match to tokens like `KINGSFORD SMITH` or 
`KINGSFORD STREET`, instead of searching the entire table for tokens with
`KINGSFORD` AND a token starting with a `s`.

To do this you would be required to create a 
[custom SQLite3 FTS5 tokeniser](https://www.sqlite.org/fts5.html#custom_tokenizers),
I may do a part 2.

### Populating the Full-Text Search Table

Now we are ready to populate the table with addresses! To do this we need to
create a query that formats all of the addresses according to our address 
format above. We will use this command to insert all of the formatted addresses
into our full-text search table:

```
$ sqlite3 gnaf.db <<SQL
    INSERT INTO ADDRESS_FTS(id, display)
    SELECT
      ADDRESS_DETAIL_PID,
      TRIM(
        CASE
            WHEN FLAT_NUMBER IS NOT NULL AND LOT_NUMBER IS NOT NULL AND FLAT_NUMBER = LOT_NUMBER THEN
                COALESCE(FLAT_TYPE || ' ', '') || COALESCE(FLAT_NUMBER_PREFIX, '') || FLAT_NUMBER || COALESCE(FLAT_NUMBER_SUFFIX, '')
            WHEN FLAT_NUMBER IS NOT NULL AND LOT_NUMBER IS NOT NULL THEN
                COALESCE(FLAT_TYPE || ' ', '') || COALESCE(FLAT_NUMBER_PREFIX, '') || FLAT_NUMBER || COALESCE(FLAT_NUMBER_SUFFIX || ' ', '') || ' LOT ' || LOT_NUMBER || COALESCE(LOT_NUMBER_SUFFIX, '')
            WHEN FLAT_NUMBER IS NOT NULL THEN
                COALESCE(FLAT_TYPE || ' ', '') || COALESCE(FLAT_NUMBER_PREFIX, '') || FLAT_NUMBER || COALESCE(FLAT_NUMBER_SUFFIX, '')
            WHEN LOT_NUMBER IS NOT NULL AND NUMBER_FIRST IS NULL THEN
                'LOT ' || LOT_NUMBER || COALESCE(LOT_NUMBER_SUFFIX, '')
            ELSE
                ''
        END || ' ' ||
        COALESCE(NUMBER_FIRST_PREFIX, '') ||
        COALESCE(NUMBER_FIRST, '') ||
        COALESCE(NUMBER_FIRST_SUFFIX, '') ||
        CASE
            WHEN NUMBER_LAST IS NOT NULL THEN ' ' ||
                COALESCE(NUMBER_LAST_PREFIX, '') ||
                COALESCE(NUMBER_LAST, '') ||
                COALESCE(NUMBER_LAST_SUFFIX, '')
            ELSE ' '
        END ||

        COALESCE(STREET_NAME || ' ', '') ||
        COALESCE(STREET_TYPE_CODE || ' ', '') ||

        COALESCE(LOCALITY_NAME || ' ', '') ||
        COALESCE(STATE_ABBREVIATION || ' ', '') ||
        COALESCE(POSTCODE, '')
      ) AS display
    FROM ADDRESS_VIEW;
SQL
```

This query looks pretty complex, but all it is doing is formatting addresses
and handling some formatting edge cases for addresses with `FLAT_TYPES`,
`FLAT_NUMBERS` and `LOT_NUMBERS`.

Let's try it out! Run the following query to get all addresses that match 
`442 KINGSFORD`.

```
$ sqlite3 -cmd ".timer on" gnaf.db <<SQL
    SELECT id, display
    FROM ADDRESS_FTS 
    WHERE display MATCH '442 KINGSFORD*'
    LIMIT 5;
SQL
```
Nice we get the expected results in about 11 milliseconds on my hardware! 
```
Run Time: real 0.011 user 0.007617 sys 0.002808
```
What happens if we keep typing the address, such that we query for
`442 KINGSFORD S*`?

```
Run Time: real 1.049 user 0.960677 sys 0.086235
```
Woah it takes about a second to complete, what's going on here?

FTS5 can quickly lookup the complete tokens `442` and `KINGSFORD` easily as they
are automatically indexed by FTS5, this makes querying complete tokens fast.
However when we search for the prefix token `S*`, FTS5 has to
do a range scan on all tokens that match `S` and greater, such as `SMITH`, 
`STREET` and similar complete tokens.

So what can we do to speed this up? Luckily FTS5 has the ability to define
prefix indices, which speed up queries for prefix tokens. For example, 
optimising a query for prefix token `S*` requires a prefix index of just one
character index. However I also want token prefixes like `ST*` and `STR*` to be
fast, so we'll create a prefix index of three-character prefixes.

Let's add the prefix configuration option to our FTS5 virtual table, we'll need
to re-create the table, so lets drop it and create it again with the new prefix
configuration option included:

```
sqlite3 gnaf.db <<SQL
    DROP TABLE IF EXISTS ADDRESS_FTS;
    CREATE VIRTUAL TABLE ADDRESS_FTS USING fts5(                                    
      id UNINDEXED,                                                                 
      display,                                                                      
      tokenize='unicode61',                                                         
      prefix='1 2 3'
    );
SQL
```

We'll then run the same population query from
[earlier](#populating-the-full-text-search-table). Then run the select query
again:

```
sqlite3 -cmd ".timer on" gnaf.db <<SQL
    SELECT id, display
    FROM ADDRESS_FTS 
    WHERE display MATCH '442 KINGSFORD S*'
    LIMIT 5;
SQL
```
```
Run Time: real 0.028 user 0.014243 sys 0.002874
```
Nice! Our database is working pretty well and we could stop here, but we won't.

### Ranking Relavent Search Results

On testing this database I found that the ranking for some search results 
weren't the best, luckily again, FTS5 has us covered with a hidden built in
`rank` column. We can simply add `ORDER BY rank` to our search query to have
ranked search results sorted by relevance.
```
sqlite3 -cmd ".timer on" gnaf.db <<SQL
    SELECT id, display
    FROM ADDRESS_FTS 
    WHERE display MATCH '442 KINGSFORD S*'
    ORDER BY rank
    LIMIT 5;
SQL
```
We do take a little speed penalty for this however, but not by much. For me,
it's a trade-off I'm willing to make for better search results.

FTS5 also has the [bm25 auxillary function](https://sqlite.org/fts5.html#the_bm25_function)
which is a much more sophisticated ranking method producing more accurate
ranked results, with a bigger speed penalty of course.

# Wrapping up

Hopefully you can see how easy it is to get full-text search with simple tools.
Full-text search engines like Elastic Search and friends have their place don't
get me wrong, but do you really need them for simple use-cases like this? I've
only scratched the surface.

## Reducing Database Size

At the moment our `gnaf.db` is sitting at around 20GB which is pretty big. If
you just want to transform physical addresses to geographical coordinates
or similar, you can simply add latitude and longitude columns to the FTS5
virtual table, and insert the latitude and longitude from the `ADDRESS_VIEW`
when populating the virtual table. We can then copy the virtual table data to a
new database (with the same virtual table schema) leaving us with a database
that can solely transform physical addresses to geographical coordinates. Doing
this leaves us with a database around 3GB in size.

## Automatically Generate the Full-Text Database

It goes without saying that all of this can be put into a shell script and 
automated. I may create a GitHub repo for automating this process for the GNAF
dataset.
