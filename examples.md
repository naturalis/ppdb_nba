Vullen van de current tabel van crs-specimen.

```bash
percolator --current --debug --source crs-specimen /data/nsr-taxa/1\ crs-specimen-meerkoeten.json 
```

Vullen van de current tabel van brahms-specimen.

```bash
percolator --current --debug --source brahms-specimen /data/nsr-taxa/3\ brahms-specimen-madeliefjes.json 
```

Vullen van de current tabel van nsr-taxa.

```bash
percolator --current --debug --source nsr-taxa /data/nsr-taxa/2\ nsr-taxa.json 
```
   
Incrementele import van nsr-taxa.

```bash
percolator --force --debug --source nsr-taxa /data/nsr-taxa/4\ nsr-taxa-changed.json
```

Geforceerd deleten van bepaalde nsr taxa records.

```bash
percolator --delete --debug --source nsr-taxa /data/nsr-taxa/delete-these-records.txt
```
