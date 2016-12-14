# Obecný
### Klávesnice
```bash
setxkbmap cz
```
### Připojení k databázi sqlplus / rman
* (RMAN je nutné restartovat pokud restartujeme DB)
```bash
unset TWO_TASK
sqlplus / as sysdba
rman target /
```

### Oracle home
```bash
ORACLE_HOME = /u01/app/oracle/product/12.1.0.2/db_1
```
### Kroky startování DB
* startup nomount
* alter database mount
* alter database open
* startup force - přeskočení kroků

***

# Zálohování
### Archivelog mód v SQLPLUS
```bash
SQL> select log_mode from v$database; #(zjistíme NOARCHIVELOG / ARCHIVELOG)
SQL> shutdown immediate;
SQL> startup mount;
SQL> alter database archivelog;
SQL> alter database open;
```

### Multiplexace řídících souborů
* zapnuto by default, není třeba dělat pokud není výslovně řečeno
```bash
SQL> show parameter control; #(zobrazí cesty k řídicím souborům, jeden zkopírujem do /home/oracle/control03.ctl)
SQL> shutdown immediate;
SQL> create pfile from spfile; #(vytvoří inicializační soubor pro čtení, DB nesmí běžet)
```

* cesta k pfile souboru: **/u01/app/oracle/product/12.1.0.2/db_1/dbs/initorcl12c.ora**
* upravíme řádek  \*.control_files='/u01/app/oracle/oradata/orcl12c/control01.ctl','/u01/app/oracle/fast_recovery_area/orcl12c/control02.ctl', **'/home/oracle/control03.ctl'**
* uložíme pfile soubor

```bash
BASH> cp /u01/app/oracle/oradata/orcl12c/control01.ctl /home/oracle/control03.ctl
SQL> create spfile from pfile;
SQL> startup;
```
### Další způsob zálohy controlfile
```bash
SQL> ALTER DATABASE BACKUP CONTROLFILE TO TRACE; #(další způsob jak zálohovat control file)
SQL> ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/manual/control.ctl'; #(manuální záloha řídícího souboru)
```
```bash
RMAN> show all; #(mělo by být CONFIGURE CONTROLFILE AUTOBACKUP ON)
RMAN> CONFIGURE CONTROLFILE AUTOBACKUP ON; #(kdyby náhodou nebylo)
```

### Nastavení maximální velikosti pro zálohu (asi není nutné pokud budeme dělat pouze jednu zálohu)
```bash
SQL> show parameter reco; #(ukáže aktuální nastavení záloh)
```
* nalezneme zde i cestu kam se ukládají zálohy /u01/app/oracle/fast_recovery_area
```bash
SQL> alter system set db_recovery_file_dest_size=10G;
```

### Multiplexace redologů (nejspíš nutná pokud bychom o nějaký přišli)
```bash
SQL> column member format a50 #(úpravy formátování)
SQL> set linesize 180
SQL> select * from v$logfile; #(ukáže nám cesty k redologům, zde později můžeme hledat problémy)
SQL> select * from v$log; #(ukáže aktivní redology
SQL> alter database add logfile member '/home/oracle/redo01.log' to group 1; #(duplikace redologu 1, 2, 3)
SQL> alter database add logfile member '/home/oracle/redo02.log' to group 2;
SQL> alter database add logfile member '/home/oracle/redo03.log' to group 3;
```
* duplikované redology jsou ve stavu invalid
```bash
SQL> alter system switch logfile; #(spustíme 3x abychom je dali do funkčního stavu)
```
- pokud smažeme jeden z šesti redologů poznáme to po restartu DB, že některá bude invalid

### Záloha inicializačních souborů
* ručně zálohujeme spfile /u01/app/oracle/product/12.1.0.2/db_1/dbs/spfileorcl12c.ora (lze použít scritp níže)
```bash
BASH> mkdir /home/oracle/spfile_backup ; cp /u01/app/oracle/product/12.1.0.2/db_1/dbs/spfileorcl12c.ora /home/oracle/spfile_backup/
```

### Zálohování samotné DB
```bash
SQL> CREATE TABLE persons (id int, name varchar(255)); #(přidáme nějaká ukázková data)SQL> 
SQL> INSERT INTO persons VALUES (1, 'pepa');
SQL> INSERT INTO persons VALUES (2, 'karel');
SQL> commit;

RMAN> backup database plus archivelog; #//zeptat se na přednášce

RMAN> backup as backupset database
RMAN> backup as compressed backupset database
RMAN> backup as copy database
```

# Skriptíky z toníkovo a filípkovo mikulášské dílny - na vlastní nebezpečí
* ruční zálohování (před rozbitím, doporučuji provádět v průběhu kompletní zálohy)
* heslo pro sudo je **oracle**

```bash
sudo yum install tree

mkdir /home/oracle/ora_backup
cp /u01/app/oracle/oradata/orcl12c/control01.ctl /home/oracle/ora_backup/control01.ctl
cp /u01/app/oracle/oradata/orcl12c/redo01.log /home/oracle/ora_backup/redo01.log
cp /u01/app/oracle/oradata/orcl12c/redo02.log /home/oracle/ora_backup/redo02.log
cp /u01/app/oracle/oradata/orcl12c/redo03.log /home/oracle/ora_backup/redo03.log
cp /u01/app/oracle/product/12.1.0.2/db_1/dbs/spfileorcl12c.ora /home/oracle/ora_backup/spfileorcl12c.ora
mkdir /home/oracle/ora_backup/tree/
tree /u01/app/oracle/oradata/orcl12c/ > /home/oracle/ora_backup/tree/oradata_tree.txt
tree /u01/app/oracle/product/12.1.0.2/db_1/dbs/ >> /home/oracle/ora_backup/tree/oradata_tree.txt
```

* diff souborové struktury oraclu (po rozbití, doporučuji provádět jako první úkon řešení problému)
  - zobrazí, který soubory chybí
```bash
tree /u01/app/oracle/oradata/orcl12c/ > /home/oracle/ora_backup/tree/oradata_tree1.txt
tree /u01/app/oracle/product/12.1.0.2/db_1/dbs/ >> /home/oracle/ora_backup/tree/oradata_tree1.txt
diff -y -W80 /home/oracle/ora_backup/tree/oradata_tree.txt /home/oracle/ora_backup/tree/oradata_tree1.txt
```
#Obnova DB:

### Kompletní obnova ze zálohy (je potřeba mít vyplou DB, bude fungovat jen pokud máme spfile)
```bash
RMAN> startup nomount;
RMAN> restore controlfile from autobackup;
RMAN> alter database mount;
RMAN> restore database;
RMAN> recover database;
RMAN> alter database open resetlogs;
```

### Ruční kontrola souborů v /u01/app/oracle/oradata/orcl12c/
* může zde chybět control file (control01.ctl)
* měli by tu být 3 redology (redo123.log)

### Použití RMAN pro identifikaci problému (z vypnuté DB)
```bash
RMAN> startup; #(pokud umře na nějaké fázi) nebo postupně - startup nomount; atd...
RMAN> list failure; #(ukáže nalezené chyby a nekonzistence v souborech)
RMAN> advise failure; #(ukáže možná řešení chyb)
RMAN> repair failure preview; #(vytiskne pomocný skript, který by měl opravit problém, pak už jen kopírovat příkazy odsud)
RMAN> repair failure; #(automaticky spustí vygenerovaný skript s opravou)
```
* když jsou smazaný všechny tři redology Rman opraví jenom redolog02.log a redolog03.log, pak už ani nenabídne pomoc, že máme opravit     manuálně -> nakopírujem ručně ze zálohy
```bash
BASH> cp /home/oracle/redo01.log /u01/app/oracle/oradata/orcl12c
```

### Zkontrolování alert logů 
* ve složce **/u01/app/oracle/diag/rdbms/orcl12c/orcl12c/trace/alert_orcl12c.log**

### Chybný spfile
* pokud při spuštění DB dostáváme **"could not open parameter file '/u01/app/oracle/product/12.1.0.2/db_1/dbs/initorcl12c.ora'"**
  - je potřeba nakopírovat do cesty zálohovaný spfile (lze použít příkaz níže)

```bash
BASH> cp /home/oracle/spfile_backup/*.ora /u01/app/oracle/product/12.1.0.2/db_1/dbs/
```

### Flashback 
* u příkladu kde chtěl vrátit tabulku o jeden krok zpět (kdy je v tabulce řádek cislo=1):
```bash
SQL> create table test(cislo number(3));
SQL> insert into test values (1);
SQL> commit;

BASH> date #- zapamotovat si čas, např Wed Dec 14 09:20:05 EST 2016

SQL> alter system switch logfile;
SQL> update test set cislo = 100;
SQL> commit;
```
* pak nám rozbije DB, pomocí RMANU a věcí tady nahoře obnovim databázi, nastartuju 

```bash
SQL> SELECT * FROM test AS OF TIMESTAMP TO_TIMESTAMP('2016-12-14 09:20:05', 'YYYY-MM-DD hh24:MI:SS');
```
* dám tam náš čas, kterej jsme si vytisknuli v konzoli před upravováním řádku 
* měl by se nám vypsat řádek CISLO
                             -----
                                 1
* když se vypíše správnej řádek můžu pokračovat - jinak asi zkoušet upravovat čas než nám vrátí správnou hodnotu
```bash
SQL> UPDATE test SET cislo = (SELECT * FROM test AS OF TIMESTAMP TO_TIMESTAMP('2016-12-14 09:20:05', 'YYYY-MM-DD hh24:MI:SS'));
```
