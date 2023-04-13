- [Setup on Ubuntu Linux](#setup-on-ubuntu-linux)
  - [Python](#python)
  - [PostgreSQL Database](#postgresql-database)
  - [Setup Mapnik and tirex](#setup-mapnik-and-tirex)
  - [Setup apache2](#setup-apache2)
  - [Setup mod_tile](#setup-mod_tile)
  - [Django & Python](#django--python)
  - [Graphhopper](#graphhopper)
  - [Initial database population](#initial-database-population)
  - [Run the importer](#run-the-importer)
  - [Rendering Map Tiles](#rendering-map-tiles)
- [List of known issues](#list-of-known-issues)

# Setup on Ubuntu Linux

Latest update: 13.06.2021

This guide describes the project setup for Ubuntu 18.04. If you have a clean setup OS you can also try running the `util/ubuntu-setup.sh` script which should do all the steps described here automatically.

Make sure all your packages are up to date with `sudo apt update`.

## Python

Install Python 3.8 and python development headers: `sudo apt install python3.8 python3.8-dev`

`python3.8 --version` should return `Python 3.8.10` now.

Now, change the `python` command to point towards `python3.8` instead of python 2:

```
update-alternatives --remove python /usr/bin/python2
sudo update-alternatives --install /usr/bin/python python /usr/bin/python3.8 10
```

Now, to install pip for python 3.8, first execute `sudo apt remove python-pip` to remove existing pip versions. Then check, whether pip is no longer installed (`pip --version`).

Next, install pip for Python 3.6 via `sudo apt install python3-pip` (`pip3 --version`) and then install pip for Python 3.8 by using pip: `python -m pip install pip`. Last, set your `~/.local/bin/` directory in the `PATH` variable. To do so, open `~./bashrc` (or your respective shell configuration file) and paste the following at the end of the file:

```
# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi
```

Now restart the terminal or execute `source ~/.bashrc` and check whether the installation was successful with `pip --version` which should return XYZ.

(Steps taken from [this](https://stackoverflow.com/questions/63207385/how-do-i-install-pip-for-python-3-8-on-ubuntu-without-changing-any-defaults) tutorial.)

Lastly, start django by executing `python manage.py runserver` in the `api/` directory.

Python packages which have to be installed after (TODO: Add to requirements.txt):

- `rdp`
- `geopy`
- `gpxpy`

## PostgreSQL Database

Install the database via `sudo apt install postgresql postgresql-contrib`. `psql --version`. should return `10.17`.

The following lines create a new database user `simra` and a database of the same name. To do so we use the default psql user `postgres` who has superadmin access to the entire PostgreSQL instance.

```
sudo -u postgres createuser simra
sudo -u postgres createdb simra
```

Next, enter the psql command line interface as the user `postgres` to allocate a password to the new `simra` user:

```
sudo -u postgres psql
psql=# alter user simra with encrypted password 'simra12345simra';
psql=# grant all privileges on database simra to simra;
psql=# alter role simra superuser;
```

The CLI should return `ALTER ROLE` and `GRANT` as visual feedback when both operations where successful. You can also check whether the database was created by typing `\list` into the psql CLI which will list all databases.

To quit the CLI, type `\q` and press enter.

(The steps above where taken from [this](https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e) tutorial.)

Install postgis via `sudo apt install postgis`. Next, you need to log into the database as the `postgres` user and activate PostGIS by executing `CREATE EXTENSION postgis;`. The `SELECT PostGIS_full_version();` should yield version 2.4.3.Log out again.

We recommend the tool pgAdmin4 for managing the PostgreSQL database because it allows for geo data visualization. It can be installed, following [this tutorial](https://www.pgadmin.org/download/pgadmin-4-apt/).

## Setup Mapnik and tirex

Clone the mapnik repository via `git clone https://github.com/mapnik/mapnik` and enter the `mapnik/` directory.

```
# you might have to update your outdated clang
sudo add-apt-repository -y ppa:ubuntu-toolchain-r/test
sudo apt-get update -y

# https://askubuntu.com/questions/509218/how-to-install-clang
# Grab the key that LLVM use to GPG-sign binary distributions
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
sudo apt-add-repository "deb http://apt.llvm.org/bionic/ llvm-toolchain-bionic-6.0 main"
sudo apt-get install -y gcc-6 g++-6 clang-6.0 lld-6.0
export CXX="clang++-6.0" && export CC="clang-6.0"


# Install mapnik
git clone https://github.com/mapnik/mapnik mapnik
cd mapnik
git checkout v3.0.x
git submodule update --init
sudo apt-get install zlib1g-dev clang make pkg-config curl
source bootstrap.sh
./configure CUSTOM_CXXFLAGS="-D_GLIBCXX_USE_CXX11_ABI=0" CXX=${CXX} CC=${CC}
make -j 3 # Uses 3 CPU cores
make test # This yields 6 errors if your OS username is not 'simra' but you can ignore them.
sudo make install
```

Install `python-mapnik` by using apt: `sudo apt install python-mapnik`. _Attention!_ This could yield errors, as this is probably not meant for mapnik 3.0.x!

For tirex we first need a new user:

```
sudo useradd -M tirex
sudo usermod -L tirex
```

Install Tirex by executing the following lines:

```
# Meet missing dependencies
sudo apt install devscripts

# Install Perl modules
sudo cpan IPC::ShareLite JSON GD LWP
sudo cpan CPAN # Updates to newer cpan

# Install, build and install Tirex
git clone git@github.com:openstreetmap/tirex.git
make
sudo make install
```

Check whether `/etc/tirex/` and `/usr/lib/tirex/` exist. If not, something went wrong and you should restart the Tirex setup.

Tirex needs some additional directories to run, so create them now: `sudo mkdir /var/lib/tirex/ /var/run/tirex/ /var/log/tirex/`

Next, run `chown tirex:tirex -R /var/lib/tirex/ /var/run/tirex/ /usr/lib/tirex/ /var/log/tirex/ /etc/tirex/` to allocated respective rights for the user `tirex`.

Now, remove the initial Mapnik directory of Tirex via `sudo rmdir /etc/tirex/renderer/mapnik`. Create a symlink to this repositories folder `tileserver/mapnik_config/` instead, e.g. `sudo ln -s /home/sfuehr/Documents/TUB-WI/S7_BA_SimRa/simra-visualizations-server/tileserver/mapnik_config /etc/tirex/renderer/mapnik`. **Be sure that the path to `mapnik_config/` does not contain spaces!**

Next, remove all content from the `openseamap/` directory via `sudo rm -r /etc/tirex/renderer/openseamap/*`.

Execute `sudo mkdir /var/lib/tirex/tiles` to create the Tirex tiles folder and for each config file in `tileserver/mapnik_config/` create a folder of the same name in `/var/lib/tirex/tiles/`: `sudo mkdir incident-combined popularity-combined relative-speed-aggregated relative-speed-single rides-density_all rides-density_rushhourevening rides-density_rushhourmorning rides-density_weekend rides-original rides-quality-segments stoptimes surface-quality-aggregated surface-quality-single popularity-score popularity-original_avoided popularity-original_chosen popularity_w-incidents_combined popularity_w-incidents_score popularity-original_w-incidents_avoided popularity-original_w-incidents_chosen`. Grant permission via `sudo chown tirex:tirex -R /var/lib/tirex`.

Then copy the service files `tirex-backend-manager.service` and `tirex-master.service` into the systems service folder. Both service files can be found inside the directory `tileserver/config/` inside this repository.

```
sudo cp ./tileserver/config/tirex-master.service /etc/systemd/system
sudo cp ./tileserver/config/tirex-backend-manager.service /etc/systemd/system
```

Grant this permission: `sudo chown tirex:tirex -R /var/run/tirex`.

Finally, update `/etc/tirex/renderer/mapnik.conf`: `plugindir` should be `/usr/local/lib/mapnik/input` and `fontdir` should be `/usr/local/lib/mapnik/fonts`.

In your `mapnik_config/` directory change the `mapfile` variable to point towards the map files inside `mapnik_maps/`.

Now you can start the Tirex services: `sudo systemctl start tirex-master` and `sudo systemctl start tirex-backend-manager`.

## Setup apache2

Install the web server by executing `sudo apt install apache2 apache2-dev`. Checking the status of the server with `systemctl status apache2` should yield `active (running)`. If not, start the web server manually via `sudo systemctl start apache2`.

Now copy `simra-ubuntu.conf` into `/etc/apache2/mods-available`: `sudo cp ./tileserver/config/simra-ubuntu.conf /etc/apache2/mods-available`. Next we system link the config file to `/etc/apache2/mods-enabled`: `sudo ln -s /etc/apache2/mods-available/simra-ubuntu.conf /etc/apache2/mods-enabled`. Apache2 will automatically look into these directories and read those configurations. Lastly restart the service via `sudo systemctl restart apache2`.

## Setup mod_tile

This package is a module for the Apache web server. If not done yet, add the OSM PPA to the OS: `sudo add-apt-repository -y ppa:osmadmins/ppa`. Now the installation is possible straight forward by executing `sudo apt-get install -y libapache2-mod-tile`. Lastly execute `sudo ln -s /var/lib/tirex/tiles /var/lib/mod_tile` to create a system link.

## Django & Python

Now that the database is set up, we will create database models which will define the form of the database tables which will be filled later. For that, the project uses the django framework which can be installed with pip: `pip install django`.

Then, execute `pip install -r requirements.txt`. This will run in an error which can be resolved by downgrading the `libpq5` package and installing some more dependencies:

```
sudo apt install libgpgme-dev python-apt python3-testresources
sudo apt-get install libpq-dev python-dev pkg-config libcairo2-dev
sudo apt-get install libpq5=10.17-0ubuntu0.18.04.1
sudo apt-get install libpq-dev python-dev
pip install psycopg2
```

TODO: Can the apt-get commands be executed together?

Now run `pip install -r requirements.txt` again.

_Attention!_ `mapnik==3.0.23` was removed from `requirements.txt`. TODO: Put back in there?

Navigate into the `api/` directory (if necessary, remove `SimRaAPI/migrations/`) and run:

```
python manage.py makemigrations SimRaAPI
python manage.py migrate
```

## Graphhopper

This component is used to determine the shortest routes between two GPS coordinates as well as to map match the raw GPS data onto a map.

```
# Download the Graphhopper web server
wget https://graphhopper.com/public/releases/graphhopper-web-3.0.jar -P ./graphhopper/

# Download street map data for DACH (Germany, Austria, Switzerland)
sudo mkdir /var/simra /var/simra/pbf
sudo wget http://download.geofabrik.de/europe/dach-latest.osm.pbf -P /var/simra/pbf/
```

If not done yet, install Java on your system: `sudo apt install default-jdk`.

Now start the web server by executing `java -jar ./graphhopper/graphhopper-web-3.0.jar server ./graphhopper/config.yml`. **Make sure this service is running before importing any GPS data.** TODO: Make `start.sh` executable: You can also use the startup script provided in the same folder. To do so, give it execution permission by calling `chmod +x ./graphhopper/start.sh`. Now you can start the Graphhopper web server via `./graphhopper/start.sh`.

_Notice: Starting the service can take a while as it will create a graph inside `graph-cache/`._

## Initial database population

Now the PostgreSQL database gets filled with basic map data, retrieved by the OSM service. To do so, we use the tool [Imposm](https://imposm.org/docs/imposm3/latest/).

Navigate into the `osm importer` directory and execute `sudo ./imposm-0.10.0-linux-x86-64/imposm import -mapping mapping.yml -read "berlin-latest.osm.pbf" -overwritecache -write -connection postgis://simra:simra12345simra@localhost/simra`. This creates the schema `import` with the table `osm_ways` which is used in the next step by the `create_legs.py` script.

## Run the importer

Next, the existing map data is separated into legs which represent single street segments of the street network. To do that, execute `python importer/importer/create_legs.py`. This will fill the database tables `OsmWaysLegs`, `OsmWaysJunctions` and `OsmWaysLargeJunctions` inside the schema `public` of the `simra` database.

Execute `python importer/importer/import.py`. This will import the CSV data into the database.

Attention, depending on the amount of data which is imported, this process can take a while. To test if your setup is working we highly recommend to test with a small amount of CSV files.

## Rendering Map Tiles

Now that everything is set up, the data in the database can be rendered into PNG images which are used by the Tirex tile server. To do so execute `tirex-batch map=rides-original bbox=11.642761,51.894292,15.135040,53.006521 z=0-10`. You can monitor this process by calling `tirex-status` in another terminal instance.

Attention, depending on the amount of data which is imported, this process can take a while.

# List of known issues

For detailed feedback, the following commands may be useful:

- `tirex-status`
- `tirex-status --once --extended`
- `journalctl -xe`
- For development and easier use of Tirex related task, check out the `tirex` bash script in `util/`. To view available commands and options execute `./util/tirex.sh -h`.

For some of the components mentioned above, detailed setup instructions can be found [here](https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/).

The directory `/var/run/tirex/` is deleted on every shutdown of the machine. Thus, `sudo mkdir /var/run/tirex/` and `chown tirex:tirex -R /var/run/tirex/` have to be repeated. You can provide a shell script to do so in your autostart directory.
