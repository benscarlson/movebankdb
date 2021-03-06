## This is old, should be revisited, and is possibly not that helpful anyway.

### Populate the database

This is a short demonstration of how to use [rmoveapi](https://github.com/benscarlson/rmoveapi) to extract data from the movebank database and load it into your movebankdb sqlite file. To keep things simple there is no filtering or de-duplication, so the data will contain a lot of errors. The movebank database is pretty messy and requires quite a lot of processing to get clean data. In particular, raw api data can have the following errors.

* Basic errors - Missing lon/lat/timestamp etc.
* Unwanted 'animals' - These are often humans carrying around a tag for testing
* Undeployed locations - Locations from testing the tag, before placing on an animal, or even locations from animals not associated with your study can find their way into your study.
* Partial duplicates - Movebank can contain semi-duplicated records in which both records are the same but one has some additional information

For analysis ready data, the data need to be processed to remove these errors. This is what [get_study_data.r](https://github.com/benscarlson/movebankdb/blob/master/db/get_study_data.r) does. That is a complex script so for illustration purposes the code below is a simplified version. Each chunk just 1) uses the api to extract data and 2) insert this data to the database. Most of the code is just specifying, ordering, and formatting the various fields.

#### Initialize

````{r}
library(DBI)
library(dplyr)
library(getPass)
library(rmoveapi)
library(RSQLite)

.wd <- '~/projects/mycoolproject'
.dbPF <- file.path(.wd,'analysis/data/movebank.db')
.evtrawP <- file.path(.wd,'analysis/data/event.csv')

db <- DBI::dbConnect(RSQLite::SQLite(), .dbPF)

invisible(assert_that(length(dbListTables(db))>0)) # Ensure that you have loaded the database correctly

setAuth(getPass('Movebank user:'),getPass('Movebank password:'))

dbBegin(db)

````

#### Tag entity

````{r}

attstr <- 'beacon_frequency,comments,id,local_identifier,manufacturer_name,model,processing_type,serial_no,tag_failure_comments,tag_production_date,weight'
attributes <- trimws(str_split(attstr,',')[[1]])

tag <- getTag(.studyid,params=list(attributes=attributes)) %>% 
  select(tag_id=id,local_identifier,manufacturer_name,everything()) %>%
  select(-comments,everything()) #This puts comment at the end

rows <- tag %>% 
  dbAppendTable(db, "tag", .)
  
message(glue('Inserted {rows} rows into the tag table'))

````

#### Study entity

````{r}

attributes <- dbListFields(db,'study')
#Need to rename request attributes since local database has different names than movebank
attributes[attributes=='study_id'] <- 'id'
attributes[attributes=='study_name'] <- 'name'

study <- getStudy(.studyid,params=list(attributes=attributes)) %>% 
  rename(study_id=id,study_name=name)

rows <- study %>% 
  mutate_if(is.POSIXct,strftime,format='%Y-%m-%dT%TZ',tz='UTC') %>%
  dbAppendTable(db, "study", .)
  
message(glue('Inserted {rows} row into the study table'))

````

#### Sensor entity

````{r}

sensor <- getSensor(.studyid) %>% rename(sensor_id=id)

rows <- sensor %>% 
  dbAppendTable(db, "sensor", .)
  
message(glue('Inserted {rows} rows into the sensor table'))

````

#### Individual entity

````{r}

attstr <- 'id, local_identifier, nick_name, study_id, ring_id, sex, taxon_id, taxon_canonical_name, access_profile_id, 
default_profile_eventdata_id, earliest_date_born, latest_date_born, exact_date_of_birth, external_id, external_id_namespace_id, 
i_am_owner, death_comments, comments'

attributes <- trimws(str_split(attstr,',')[[1]])

ind <- getIndividual(.studyid,params=list(attributes=attributes)) 

rows <- ind %>% 
  mutate_if(is.POSIXct,strftime,format='%Y-%m-%dT%TZ',tz='UTC') %>%
  dbAppendTable(db, "individual", .)

message(glue('Inserted {rows} rows into the individual table'))

````

#### Deployment entity

````{r}

attstr <- 'id, local_identifier, individual_id, tag_id, deploy_on_timestamp, deploy_off_timestamp'
attributes <- trimws(str_split(attstr,',')[[1]])

dep <- getDeployment(.studyid,params=list(attributes=attributes)) %>%
  rename(deployment_id=id)

rows <- dep %>% 
  mutate_if(is.POSIXct,strftime,format='%Y-%m-%dT%TZ',tz='UTC') %>%
  dbAppendTable(db, "deployment", .)

message(glue('Inserted {rows} rows into the deployment table'))

````

#### Event entity

The event entity is often very large and can't be loaded directly into memory. To get around this issue, use  `getEvent(...,save_as=<myeventdata.csv>` to directly save the data to a csv file. Then, load the csv file into R and insert into the database.

The code below gets just the gps (sensor_type_id=653) data

````{r}

attributes <- c('event_id','individual_id','location_long','location_lat','timestamp',
                'sensor_type_id','tag_id',
                'ground_speed','gps_speed_accuracy_estimate',
                'visible',
                'gps_dop','gps_hdop','gps_vdop',
                'eobs_horizontal_accuracy_estimate','gps_horizontal_accuracy_estimate',
                'location_error_numerical', 'location_error_percentile', 'location_error_text',
                'gps_fix_type', 'gps_satellite_count', 'gps_time_to_fix', 'eobs_type_of_fix', 
                'eobs_used_time_to_get_fix', 'eobs_status')
                

t1 <- Sys.time()
evt0 <- getEvent(.studyid,attributes,sensor_type_id=653,save_as=.evtrawP)
t2 <- Sys.time()
message(glue('Complete in {diffmin(t1,t2)} minutes'))

evt <- read_csv(.evtrawP)

rows <- evt %>% 
  mutate_if(is.POSIXct,strftime,format='%Y-%m-%dT%TZ',tz='UTC') %>%
  dbAppendTable(db, "event", .)
  
message(glue('Inserted {format(rows,big.mark=",")} rows into the event table'))
  
````

#### Finalize

````{r}
dbCommit(db)
dbDisconnect(db)
````
