# Plex-Meta Manager scripts

Misc scripts and tools. Undocumented scripts probably do what I need them to but aren't finished yet.

## Setup

See the top-level [README](../README.md) for setup instructions.

All these scripts use the same `.env` and requirements.

### `.env` contents

```
TMDB_KEY=TMDB_API_KEY                        # https://developers.themoviedb.org/3/getting-started/introduction
TVDB_KEY=TVDB_V4_API_KEY                     # currently not used; https://thetvdb.com/api-information
PLEX_URL=https://plex.domain.tld             # URL for Plex; can be a domain or IP:PORT
PLEX_TOKEN=PLEX-TOKEN
PLEX_OWNER=yournamehere                      # account name of the server owner
LIBRARY_NAMES=Movies,TV Shows,Movies 4K      # comma-separated list of libraries to act on
CAST_DEPTH=20                                # how deep to go into the cast for actor collections
TOP_COUNT=10                                 # how many actors to export
TARGET_LABELS=this label, that label         # comma-separated list of labels to remove posters from
REMOVE_LABELS=True                           # attempt to remove the TARGET_LABELs from items after resetting the poster
DELAY=1                                      # optional delay between items
POSTER_DIR=extracted_posters                 # put downloaded posters here
POSTER_DEPTH=20                              # grab this many posters [0 grabs all]
POSTER_DOWNLOAD=0                            # if set to 0, generate a script rather than downloading
POSTER_CONSOLIDATE=1                         # if set to 0, posters are separated into folders by library
PMM_CONFIG_DIR=/opt/pmm/Plex-Meta-Manager/config/ # path to Plex-Meta-Manager config directory

# ORIGINAL TO ASSETS
USE_ASSET_FOLDERS=1                          # should the asset directory use asset folders?
ASSETS_BY_LIBRARIES=1                        # should those asset folders be sorted into library folders?
ASSET_DIR=assets                             # top-level directory for those assets
```

## Scripts:
1. [clean-overlay-backup.py](#clean-overlay-backuppy) - clean out leftover overlay backup art
1. [extract-collections.py](#extract-collectionspy) - extract collections from a library
1. [overlay-default-posters.py](#overlay-default-posterspy) - apply overlays to default collection posters
1. [pmm-trakt-auth.py](#pmm-trakt-authpy) - generate trakt auth block for PMM config.yml
1. [pmm-mal-auth.py](#pmm-mal-authpy) - generate mal auth block for PMM config.yml
1. [original-to-assets.py](#original-to-assetspy) - Copy image files from an "Original Posters" directory to an asset directory

### OBSOLETE
1. [top-n-actor-coll.py](#top-n-actor-collpy) - generate collections for the top *n* actors in a library

## clean-overlay-backup.py

You've deleted stuff from Plex and want to clean up the leftover backup art that PMM saved when it applied overlays

### Settings

The script uses these settings from the `.env`:
```
PLEX_URL=https://plex.domain.tld             # URL for Plex; can be a domain or IP:PORT
PLEX_TOKEN=PLEX-TOKEN
LIBRARY_NAMES=Movies,TV Shows,Movies 4K      # comma-separated list of libraries to act on
DELAY=1                                      # optional delay between items
PMM_CONFIG_DIR=/opt/pmm/config/              # path to Plex-Meta-Manager config directory
```

### Usage
1. setup as above
2. Run with `python clean-overlay-backup.py`

The script will catalog the backup files and current Plex contents for each library listed in the `.env`.

It then compares the two lists, and any files in the backup dir that do not correspond to current items in Plkex are deleted.

```
Starting clean-overlay-backup 0.1.0 at 2024-02-13 17:52:31
connecting to https://plex.bing.bang...
6686 images in the Movies overlay backup directory ...
Loading Movies ...
Loading movies from Movies  ...
Completed loading 6965 of 6965 movie(s) from Movies
Clean Overlay Backup Movies |████████████████████████████████████████| 6965/6965 [100%] in 11.0s (633.23/s) 
Processed 6965 of 6965
0 items to delete
279 items in Plex with no backup art
These might be items added to Plex since the last overlay run
They might be items that are not intended to have overlays
...
```

## extract-collections.py

You're getting started with Plex-Meta-Manager and you want to export your existing collections

Here is a quick and dirty [emphasis on "quick" and "dirty"] way to do that.

### Usage
1. setup as above
2. Run with `python extract-collections.py`

The script will grab some details from each collection and write a metadata file that you could use with PMM.  It also grabs artwork and background.

This is extremely naive; it doesn't recreate filters, just grabs a list of everything in each collection.

For example, you'll end up with something like this for each collection:

```yaml
collections:
  ABC:
    sort_title: +++_ABC
    url_poster: ./config/TV Shows - 4K-artwork/ABC.png
    summary: A collection of ABC content
    collection_order: release
    plex_search:
      any:
        title:
          - Twin Peaks
          - Strange World
          - Designated Survivor
          - The Good Doctor
```

## overlay-default-posters.py

You want to apply an overlay to the default collection posters; perhaps for branding, perhaps you don't like the separators, whatever

Here is a basic script to do that.

### Usage
1. setup as above
1. create overlay images in `default_collection_overlays` [see README in that folder for notes.]
1. Run with `overlay-default-posters.py`

The script will clone or update the `Plex-Meta-Manager-Images` repo, then iterate through it applying overlays to each image and storing them in a parallel file system rooted at `Plex-Meta-Manager-Images-Overlaid`, ready for you to use with the PMM Asset Directory [after moving them to that directory] or via template variables.

It chooses the overlay by name based on the "group" that each collection is part of: 
```
Plex-Meta-Manager-Images
├── aspect
├── audio_language
├── award
├── based
├── chart
├── content_rating
├── country
├── decade
├── franchise
├── genre
├── network
├── playlist
├── resolution
├── seasonal
├── separators
├── streaming
├── studio
├── subtitle_language
├── universe
└── year
```

For example, all the `audio_language` collection posters will be overlaid by `Plex-Meta-Manager/default_collection_overlays/audio_language.png`.

If there isn't a specific image for a "group", then `Plex-Meta-Manager/default_collection_overlays/overlay-template.png` will be used.

```
Fetch/Pull on Plex-Meta-Manager-Images
Using default_collection_overlays/overlay-template.png as global overlay
building list of targets
Applying overlays |████████████████████████████▎           | ▇▅▃ 5027/7119 [71%] in 3:53 (21.6/s, eta: 1:37) 
Plex-Meta-Manager-Images/genre/Sword & Sandal.jpg
```

## pmm-trakt-auth.py

Perhaps you're running PMM in a docker or something where getting it into interactive mode to authentication trakt is a hassle.

This little script will generate the trakt section for your PMM config file.  Most of this code is pulled from PMM's own trakt authentication; it's just been simplified to do the one thing.

You can run this on a completely separate machine to where PMM is running.

There is an online version available [here](https://metamanager.wiki/en/latest/config/auth/).

### Usage
1. Run with `python pmm-trakt-auth.py`


You'll be asked for your trakt Client ID and Client Secret then taken to a trakt web page.

Copy the PIN and paste it at the prompt.

Some yaml will be printed, ready to copy-paste into your PMM config.yml.

```
Let's authenticate against Trakt!


Trakt Client ID: JOHNNYJOEYDEEDEE
Trakt Client Secret: PETEROGERJOHNKEITH
Taking you to: https://trakt.tv/oauth/authorize?response_type=code&client_id=JOHNNYJOEYDEEDEE&redirect_uri=urn%3Aietf%3Awg%3Aoauth%3A2.0%3Aoob

If you get an OAuth error your Client ID or Client Secret is invalid

If a browser window doesn't open go to that URL manually.


Enter the Trakt pin from that web page: 9CE4045E
Copy the following into your PMM config.yml:
############################################
trakt:
  client_id: JOHNNYJOEYDEEDEE
  client_secret: PETEROGERJOHNKEITH
  authorization:
    access_token: OZZYTONYGEEZERBILL
    token_type: Bearer
    expires_in: 7889237
    refresh_token: JOHNPAULGEORGERINGO
    scope: public
    created_at: 1644589166
############################################
```

## pmm-mal-auth.py

This little script will generate the `mal` section for your PMM config file.  Most of this code is pulled from PMM's own MAL authentication; it's just been simplified to do the one thing.

You can run this on a completely separate machine to where PMM is running.

There is an online version available [here](https://metamanager.wiki/en/latest/config/auth/).

### Usage
1. `python3 -m pip install pyopenssl`
1. `python3 -m pip install requests secrets`
1. Run it with `python3 pmm-mal-auth.py`.

You'll be asked for your MyAnimeList Client ID and Client Secret then taken to a MyAnimeList web page.

Log in and click "Allow"; you'll be redirected to a localhost page that won't load.

Copy that localhost URL and paste it at the prompt.

Some yaml will be printed, ready to copy-paste into your PMM config.yml.

```
Let's authenticate against MyAnimeList!


MyAnimeList Client ID: JOHNNYJOEYDEEDEE
MyAnimeList Client Secret: PETEROGERJOHNKEITH
We're going to open https://myanimelist.net/v1/oauth2/authorize?response_type=code&client_id=JOHNNYJOEYDEEDEE&code_challenge=STINGANDYSTEWART


Log in and click the Allow option.

You will be redirected to a localhost url that probably won't load.

That's fine.  Copy that localhost URL and paste it below.

Hit enter when ready:
URL: http://localhost/?code=TuomasEmppuTroyFloorKaiJukka


Copy the following into your PMM config.yml:
############################################
mal:
  client_id: JOHNNYJOEYDEEDEE
  client_secret: PETEROGERJOHNKEITH
  authorization:
    access_token: OZZYTONYGEEZERBILL
    token_type: Bearer
    expires_in: 2415600
    refresh_token: JOHNPAULGEORGERINGO
############################################

```

## original-to-assets.py

You've applied overlays to a library and want to seed an asset directory with the "Original Posters".

Here is a basic script to do that.

### Usage
1. setup as above
1. Run with `original-to-assets.py`

The script will copy the contents of the "Original Posters" directory to an asset directory.

```
# ORIGINAL TO ASSETS
USE_ASSET_FOLDERS=1                          # should the asset directory use asset folders?
ASSET_DIR=assets                             # top-level directory for those assets
```
The asset file system will be rooted at the directory in the `ASSET_DIR` setting, and `USE_ASSET_FOLDERS` controls whether the images are stored as:

`USE_ASSET_FOLDERS=1`
```
Media-Scripts/Plex/assets/All That Jazz (1979) {imdb-tt0078754} {tmdb-16858}.jpg
```
or `USE_ASSET_FOLDERS=0`
```
Media-Scripts/Plex/assets/All That Jazz (1979) {imdb-tt0078754} {tmdb-16858}/poster.jpg
```

example output:
```
Starting originals-to-assets 0.0.1 at 2024-04-01 17:13:30
connecting to https://test-plex.DOMAIN.TLD...
Loading Test-Movies ...
Loading movies  ...
Completed loading 35 of 35 movie(s) from Test-Movies
Grab all posters Test-Movies |████████████████████████████████████████| 35/35 [100%] in 0.2s (190.63/s) 
Processed 35 of 35
Complete!
```

## OBSOLETE SCRIPTS

## top-n-actor-coll.py

This has been obsoleted by "Dynamic Collections" in PMM; it's left here for historical reference.

You should use Dynamic Collections instead.

Connects to a plex library, grabs all the movies.

For each movie, gets the cast from TMDB; keeps track across all movies how many times it sees each actor.  You can specify a TV library, but I haven't tested that a lot.  My one attempt showed a list of 10 actors who had each been in 1 series, which doesn't seem right.

At the end, builds a basic Plex-Meta-Manager metadata file for the top N actors.

Script-specific variables in .env:
```
CAST_DEPTH=20                   ### HOW DEEP TO GO INTO EACH MOVIE CAST
TOP_COUNT=10                    ### PUT THIS MANY INTO THE FILE AT THE END
```

`CAST_DEPTH` is meant to prevent some journeyman character actor from showing up in the top ten; I'm thinking of someone like Clint Howard who's been in the cast of many movies, but I'm guessing when you think of the top ten actors in your library you're not thinking about Clint.  Maybe you are, though, in which case set that higher.

`TOP_COUNT` is the number of actors to dump into the metadata file at the end.

`template.tmpl` - this is the beginning of the target metadata file; change it if you like, but you're on your own there.

`collection.tmpl` - this is the collection definition inserted for each actor [`%%NAME%%%` and `%%ID%%` are placeholders that get substituted for each actor]; change it if you like, but this script only knows about those two data field substitutions.

### Usage
1. setup as above
1. Run with `python top-n-actor-coll.py`

```
connecting...
getting movies...
looping over 2819 movies...
[==--------------------------------------] 5.4% ... Annihilation
```

It will go through all your movies, and then at the end print out however many actors you specified in TOP_COUNT and produce a yml file.

```
38      2231-Samuel L. Jackson
26      500-Tom Cruise
25      64-Gary Oldman
25      192-Morgan Freeman
23      884-Steve Buscemi
22      62-Bruce Willis
22      31-Tom Hanks
21      19278-Bill Hader
21      2888-Will Smith
21      16483-Sylvester Stallone
```

In my 4K movie library, the script produces:

```
######################################################
#               People Collections                   #
######################################################
templates:
  Person:
    smart_filter:
      any:
        actor: tmdb
        director: tmdb
        writer: tmdb
        producer: tmdb
      sort_by: year.asc
      validate: false
    tmdb_person: <<person>>
    sort_title: +9_<<collection_name>>
    schedule: weekly(monday)
    collection_order: release
    collection_mode: hide

collections:
  Samuel L. Jackson:
    template: {name: Person, person: 2231}
  Tom Cruise:
    template: {name: Person, person: 500}
  Gary Oldman:
    template: {name: Person, person: 64}
  Morgan Freeman:
    template: {name: Person, person: 192}
  Steve Buscemi:
    template: {name: Person, person: 884}
  Bruce Willis:
    template: {name: Person, person: 62}
  Tom Hanks:
    template: {name: Person, person: 31}
  Bill Hader:
    template: {name: Person, person: 19278}
  Will Smith:
    template: {name: Person, person: 2888}
  Sylvester Stallone:
    template: {name: Person, person: 16483}
```
