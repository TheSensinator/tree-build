# Introduction

Creating a bespoke OneZoom tree involves a number of steps, as documented below. These take an initial tree, map taxa onto Open Tree identifiers, add subtrees from the OpenTree of Life, resolve polytomies and delete subspecies, and calculate mappings to other databases together with creating wikipedia popularity metrics for all taxa. Finally, the resulting tree and database files are converted to a format usable by the OneZoom viewer. Mapping and popularity calculations require various large files to be downloaded e.g. from wikipedia, as [documented here](../data/README.markdown).

The instructions below are primarily intended for creating a full tree of all life on the main OneZoom site. If you are making a bespoke tree, you may need to tweak them slightly.

The output files created by the tree building process (database files and files to feed to the js, and which can be loaded into the database and for the tree viewer) are saved in `data/output_files`.

## Using DVC (recommended)

The entire build is defined as a [DVC](https://dvc.org/) pipeline in `dvc.yaml`, with parameters in `params.yaml`. This means you can reproduce the full build with a single command:

```bash
source .venv/bin/activate
dvc repro
```

If the pipeline has already been run by someone else and the results pushed to the DVC remote, you can pull cached outputs without downloading any of the large source files:

```bash
dvc repro --pull --allow-missing
```

To run only up to a specific stage (e.g. just the JS generation):

```bash
dvc repro make_js
```

To visualize the pipeline graph:

```bash
dvc dag
```

After running the pipeline, copy the JS output from `data/js_output/` to the OZtree repo:

```bash
cp data/js_output/* ../OZtree/static/FinalOutputs/data/
```

Then see the section titled "Upload data to the server and check it" below.

### Updating parameters

Edit `params.yaml` to change the OpenTree version, taxonomy version, build version, etc. DVC will detect the parameter changes and re-run only the affected stages.

### Upload data to the server and check it

8. If you are running the tree building scripts on a different computer to the one running the web server, you will need to push the `completetree_XXXXXX.js`, `completetree_XXXXXX.js.gz`, `cut_position_map_XXXXXX.js`, `cut_position_map_XXXXXX.js.gz`, `dates_XXXXXX.js`, `dates_XXXXXX.js.gz` files onto your server, e.g. by pushing to your local Github repo then pulling the latest github changes to the server.
1. (15 mins) load the CSV tables into the DB, using the SQL commands printed in step 6 (at the end of the `data/output_files/ordered_output.log` file: the lines that start something like `TRUNCATE TABLE ordered_leaves; LOAD DATA LOCAL INFILE ...;` `TRUNCATE TABLE ordered_nodes; LOAD DATA LOCAL INFILE ...;`). Either do so via a GUI utility, or copy the `.csv.mySQL` files to a local directory on the machine running your SQL server (e.g. using `scp -C` for compression) and run your `LOAD DATA LOCAL INFILE` commands on the mysql command line (this may require you to start the command line utility using `mysql --local-infile`, e.g.:

   ```
   mysql --local-infile --host db.MYSERVER.net --user onezoom --password --database onezoom_dev
   ```

1. Check for dups, and if any sponsors are no longer on the tree, using something like the following SQL command:

   ```
   select * from reservations left outer join ordered_leaves on reservations.OTT_ID = ordered_leaves.ott where ordered_leaves.ott is null and reservations.verified_name IS NOT NULL;
   select group_concat(id), group_concat(parent), group_concat(name), count(ott) from ordered_leaves group by ott having(count(ott) > 1)
   ```

### Fill in additional server fields

11. (15 mins) create example pictures for each node by percolating up. This requires the most recent `images_by_ott` table, so either do this on the main server, or (if you are doing it locally) update your `images_by_ott` to the most recent server version.

    ```
    ${OZ_DIR}/OZprivate/ServerScripts/Utilities/picProcess.py -v
    ```

1. (5 mins) percolate the IUCN data up using

   ```
   ${OZ_DIR}/OZprivate/ServerScripts/Utilities/IUCNquery.py -v
   ```

   (note that this both updates the IUCN data in the DB and percolates up interior node info)

1. (10 mins) If this is a site with sponsorship (only the main OZ site), set the pricing structure using SET_PRICES.html (accessible from the management pages).
1. (5 mins - this does seem to be necessary for ordered nodes & ordered leaves). Make sure indexes are reset. Look at `OZprivate/ServerScripts/SQL/create_db_indexes.sql` for the SQL to do this - this may involve logging in to the SQL server (e.g. via Sequel Pro on Mac) and pasting all the drop index and create index commands.

### At last

15. Have a well deserved cup of tea
