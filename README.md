<h1 id="ppdb_nba">ppdb_nba</h1>

Dit is de NBA preprocessing database class.

Hierin zitten alle functies en database afhankelijkheden waarmee import data
kan worden gefilterd alvorens een import in de NBA documentstore plaatsvind.

## installatie

Installeren kan het beste via pip. Dit is een python3 module.

`pip install -e git+https://github.com/naturalis/ppdb_nba.git#egg=ppdb_nba`

Vervolgens wordt de module `ppdb_nba` en het commando `ppdb_nba` toegevoegd
aan je executable path. De aanroep is meestal:

```
ppdb_nba --source [bronnaam] /volledigepad/naar/jsonlinesfile.txt
```

**LET OP: Het pad naar de jsonlines bestand moet exact hetzelfde zijn als die voor
postgres instance. En bij voorkeur niet relatief!**

Meer opties zijn te vinden bij aanroep met --help

```
usage: ppdb_nba --source sourcename /path/file1

Preprocessing data to create incremental updates

positional arguments:
  files            One json data file

optional arguments:
  -h, --help       show this help message and exit
  --source SOURCE  Name of the data source
  --config CONFIG  Config file
  --current        Import data directly to current table (default is normal
                   incremental import)
  --delete         Handle permanent deletes (default is normal incremental
                   import)
  --force          Ignore lockfiles, to force the import
  --createtables   Generate database tables needed for importing
  --debug          Set debugging level logging
```

## Cron

Standaard zal het ppdb_nba script draaien door steeds opnieuw te starten via
crontab. 

```
 * * * * * cd /shared-data && ppdb_nba
```

Periodiek scant `ppdb_nba` de jobs directory. De files die hier worden 
aangetroffen worden op volgorde van timestamp (oplopend) verwerkt. Er wordt 
maar één job per keer verwerkt. Op het moment dat een job wordt behandelt 
wordt er een lock file gezet, zolang die lock file er staat wordt er geen 
import proces gestart. In de lock file wordt de naam van het
job bestand gezet.

Na het succesvol afhandelen van de import file(s) in een job worden de 
jsonlines bestanden in `./imported/` gezet. De job file gaat naar done. 
Als een job niet succesvol wordt afgehandeld wordt hij in failed gezet, 
de import data blijft in dit geval staan.

Nadat een job file is afgehandeld wordt de .lock file weer verwijderd, 
waarna de volgende job wordt opgepakt.

## Logging

ppdb_nba logt naar een elastic search server volgens de wijze beschreven
door Atze. Alle relevante acties worden weggeschreven:

 - start
 - finish
 - fail
 - new
 - update
 - delete
 
Dit gebeurt met de functie [log_change](https://github.com/naturalis/ppdb_nba/blob/c499b29875254045e0093006d8655731973a9129/ppdb_nba/ppdb_nba.py#L316).

## Directory structuur

De onderstaande mappen structuur is volledig instelbaar via `config.yml`, die in
de root van de shared data directory. Op de huidige productie machine is dat in
`/data/shared-data`, de mappenstructuur staat alsvolgt gedefinieerd:

```
paths:
    incoming: /shared-data/incoming
    processed: /shared-data/processed
    jobs: /shared-data/jobs
    failed: /shared-data/failed
    done: /shared-data/done
    delta: /shared-data/incremental 
```

**LET OP: De paden moeten in alle docker containers te vinden zijn op dezelfde 
locatie anders gaat het mis.**

### /shared-data/incoming

Hier komen alle json lines bestanden die (nog) moeten worden opgepakt.

### /shared-data/jobs

De json files in deze directory bevatten de meta informatie over
te importeren data. Job files die in behandeling zijn worden
gelocked in de `.lock` file. 

### /shared-data/processed

Hier komen de data files terecht die succesvol zijn afgehandeld.

### /shared-data/done

Hier komen de job files terecht die zijn afgehandeld.

### /shared-data/failed

Hier komen de job files terecht die niet succesvol zijn afgehandeld.

### /shared-data/incremental

Hier komen alle `delta` veranderingen op de nba database. Dit zijn de records die
een-voor-een zullen worden ingelezen en gepost naar de elastic search database.

### Import proces

Periodiek scant `ppdb_nba` de jobs directory. De files die hier worden 
aangetroffen worden op volgorde van timestamp (oplopend) verwerkt. Er
wordt maar een job per keer verwerkt. Op het moment dat een job wordt
behandelt wordt er een lock file gezet, zolang die lock file wordt
er geen import proces gestart. In de lock file wordt de naam van het
job bestand gezet.

Na het succesvol afhandelen van de import file(s) in een job worden de 
jsonlines bestanden in `./imported/` gezet. De job file gaat naar done.
Als een job niet succesvol wordt afgehandeld wordt hij in failed gezet,
de import data blijft in dit geval staan.

Nadat een job file is afgehandeld wordt de .lock file weer verwijderd, 
waarna de volgende job wordt opgepakt.


## Class

Om de class te gebruiken:

```python
from ppdb_nba import ppdbNBA
pp = ppdbNBA(config='config.yml',source='naam-van-de-bron')
```

<h2 id="ppdb_nba.ppdb_nba.open_deltafile">ppdbNBA.open_deltafile</h2>

```python
pp.open_deltafile(action='new', index='unknown')
```

Open een delta bestand met records of id's om weg te schrijven.

<h2 id="ppdb_nba.ppdb_nba.clear_data">ppdbNBA.clear_data</h2>

```python
pp.clear_data(table='')
```
Verwijder data uit tabel.
<h2 id="ppdb_nba.ppdb_nba.import_data">ppdbNBA.import_data</h2>

```python
pp.import_data(table='', datafile='')
```

Importeert data direct in de postgres database. En laat zoveel mogelijk over aan postgres zelf.

<h2 id="ppdb_nba.ppdb_nba.remove_doubles">ppdbNBA.remove_doubles</h2>

```python
pp.remove_doubles()
```
Bepaalde bronnen bevatte dubbele records, deze moeten eerst worden verwijderd, voordat de hash vergelijking wordt uitgevoerd.
<h2 id="ppdb_nba.ppdb_nba.list_changes">ppdbNBA.list_changes</h2>

```python
pp.list_changes()
```

Identificeert de verschillen tussen de huidige database en de nieuwe data, op basis van hash.

Als een hash ontbreekt in de bestaande data, maar aanwezig is in de nieuwe data. Dan kan het gaan
om een nieuw (new) record of een update.

Een hash die aanwezig is in de bestaande data, maar ontbreekt in de nieuwe data kan gaan om een
verwijderd record. Maar dit is alleen te bepalen bij analyse van complete datasets. Een changes
dictionary ziet er over het algemeen zo uit.

    ```
        changes = {
            'new': {
                '3732672@BRAHMS' : [12345],
                '1369617@BRAHMS' : [45678],
                '2455323@BRAHMS' : [99999]
            },
            'update': {
                '3732673@BRAHMS' : [12345,67676]
            },
            'delete': {
                '3732674@BRAHMS' : [55555]
            }
        }
    ```


<h2 id="ppdb_nba.ppdb_nba.handle_new">ppdbNBA.handle_new</h2>

```python
pp.handle_new()
```

Afhandelen van alle nieuwe records.


<h2 id="ppdb_nba.ppdb_nba.handle_updates">ppdbNBA.handle_updates</h2>

```python
pp.handle_updates()
```

Afhandelen van alle updates.

<h2 id="ppdb_nba.ppdb_nba.handle_deletes">ppdbNBA.handle_deletes</h2>

```python
pp.handle_deletes()
```

Afhandelen van alle deletes.

<h2 id="ppdb_nba.ppdb_nba.handle_changes">ppdbNBA.handle_changes</h2>

```python
pp.handle_changes()
```

Afhandelen van alle veranderingen.

## ppdbNBA.import_deleted(filename='')

```python
pp.import_deleted('/path/to/listofdeleteids.txt')
```

Handelt alle deletes af, dit zijn de geforceerde deletes of van bronnen die
geen volledige dumps leveren.


## Voorbeelden

Voorbeeld van een script dat een import doet.

```python
from ppdb_nba import *
logger.setLevel(logging.DEBUG)
# Zet de logging level op DEBUG

pp = ppdbNBA(config='config.yml',source='brahms-specimen')
# configureer de preprocessor
pp.clear_data(table=pp.source_config.get('table') + "_import")
# verwijder de data uit de import tabel
pp.import_data(table=pp.source_config.get('table') + "_import", datafile='/data/brahms-specimen/1-base.json')
# importeer de basis data
pp.remove_doubles()
# verwijder de dubbele
pp.list_changes()
# bepaal de veranderingen (allemaal nieuwe records)
pp.handle_changes()
# handel de nieuwe af
```

