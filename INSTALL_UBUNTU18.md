# devstats installation on Ubuntu

Prerequisites:
- Ubuntu 18.04.
- [golang](https://golang.org).
    - `apt-get update`
    - `sudo locale-gen "en_US.UTF-8"`
    - Add to `/etc/environment`:
    ```
    LC_ALL=en_US.UTF-8
    LANG=en_US.UTF-8
    ```
    - Check if language/locale settings are ok: `perl -e exit`.
    - `apt install golang` - this installs Go 1.10.1.
    - `apt install git psmisc jsonlint yamllint gcc`
    - `mkdir /data; mkdir /data/dev`
1. Configure Go:
    - For example add to `~/.bash_profile`:
     ```
     GOPATH=/data/dev; export GOPATH
     PATH=$PATH:$GOPATH/bin; export PATH
     ```
    - Logout and login again.
    - [golint](https://github.com/golang/lint): `go get -u golang.org/x/lint/golint`
    - [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports): `go get -u golang.org/x/tools/cmd/goimports`
    - [goconst](https://github.com/jgautheron/goconst): `go get -u github.com/jgautheron/goconst/cmd/goconst`
    - [usedexports](https://github.com/jgautheron/usedexports): `go get -u github.com/jgautheron/usedexports`
    - [errcheck](https://github.com/kisielk/errcheck): `go get -u github.com/kisielk/errcheck`
    - [cors](https://github.com/rs/cors): `go get -u github.com/rs/cors`
    - [jsoniter](https://github.com/json-iterator/go): `go get -u github.com/json-iterator/go`
2. Go to `$GOPATH/src/github.com/cncf` and clone devstats and devstatscode there:
    - `git clone https://github.com/cncf/devstats.git`.
    - `git clone https://github.com/cncf/devstatscode.git`.
    - cd `devstats`.
    - Set reuse TCP connections: `./cron/sysctl_config.sh`
3. If you want to make changes and PRs, please clone `devstats` and `devstatscode` from GitHub UI, and clone your forked version instead, like this:
    - `git clone https://github.com/your_github_username/devstats.git`
    - `git clone https://github.com/your_github_username/devstatscode.git`
4. Go to `devstatscode` directory, so you are in `~/dev/go/src/github.com/cncf/devstatscode` directory and compile binaries:
    - `make`
5. If compiled sucessfully then execute test coverage that doesn't need databases:
    - `make test`
    - Tests should pass.
6. Install binaries:
    - `sudo mkdir /etc/gha2db`
    - `sudo chmod 777 /etc/gha2db`
    - `sudo make install`
7. Go to `devstats` directory, so you are in `~/dev/go/src/github.com/cncf/devstats` and install scripts, metrics and other config/data files:
    - `make install`
8. Install Postgres database ([link](https://gist.github.com/sgnl/609557ebacd3378f3b72)):
    - Make sure you database uses `en_US.UTF-8` locale and collation.
    - `apt install postgresql` (you can use specific version, for example `postgresql-9.6`)
    - `devstats` repo directory must be available for postgres user. `chmod -R ugo+r /data/`.
    - `sudo -i -u postgres`, `psql` and as root `sudo -u postgres psql` to test installation.
    - Postgres only allows local connections by default so it is secure, we don't need to disable external connections:
    - Config file is: `/etc/postgresql/10/main/pg_hba.conf`, instructions to enable external connections (not recommended): `http://www.thegeekstuff.com/2014/02/enable-remote-postgresql-connection/?utm_source=tuicool`
    - You need to set something like `host all all 1.2.3.4/32 md5` in this file and also `listen_addresses = '*'` in `postgresql.conf`.
    - Set bigger maximum number of connections, at least 4x number of your CPU cores or more: `/etc/postgresql/10/main/postgresql.conf`. Default is 100. `max_connections = 300`.
    - You can also set `shared_buffers = ...` to something like 25% of your RAM. This is optional.
    - `service postgresql restart` or `systemctl restart postgresql@10-main`.
    - `./devel/set_psql_password.sh` to set postgres user password.
    - `chown -R postgres /var/log/postgresql`.
9. Setup GitHub OAuth
    - You need to have GitHub OAuth token, either put this token in `/etc/github/oauth` file or specify token value via `GHA2DB_GITHUB_OAUTH=deadbeef654...10a0` (here you token value)
    - If you want to use multiple tokens, create `/etc/github/oauths` file that contain list of comma separated OAuth keys or specify token values via `GHA2DB_GITHUB_OAUTH=key1,key2,...,keyN`
    - If you really don't want to use GitHub OAuth2 token, specify `GHA2DB_GITHUB_OAUTH=-` - this will force tokenless operation (via public API), it is a lot more rate limited than OAuth2 which gives 5000 API points/h
    - If you set `GHA2DB_GHAPISKIP=1` all GitHub API calls will be skipped. You can set `GHA2DB_GHAPISKIP=1` then, because artificial events cleanup is not needed when GitHub API is not needed. If both those variables are set, `ghapi2db` won't be called at all.
    - If your project(s) use icons (some of them has value other than `ICON="-"`, then you need to clone CNCF artwork repo into `~/dev/cncf/artwork`: `cd ~/dev/cncf/`, `git clone https://github.com/cncf/artwork.git`.
    - You can have artwork elsewhere, then you must use `ARTWORK=/path/to/artwork/repo`. If all projects use `ICON="-"` artwork is not needed.
    - You also need ImageMagick's convert utility: `apt install imagemagick`.
    - You need to have `/var/www/html/img` directory available for deploy user.
10. Install Grafana.
    - Go to: `https://grafana.com/grafana/download`.
    - `wget https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana_5.x.x_amd64.deb`.
    - `sudo dpkg -i grafana_5.x.x_amd64.deb`
11. Install Apache & SSL (You need to have a working DNS name for this). This is optional.
    - `apt-get install apache2`.
    - Create "web" directory: `mkdir /var/www/html/` (it will hold gha databases dumps and other static info on the main domain.)
    - `chmod -R ugo+rwx /var/www`.
    - Enable mod proxy and mod rewrite:
    - `ln /etc/apache2/mods-available/proxy.load /etc/apache2/mods-enabled/`
    - `ln /etc/apache2/mods-available/proxy.conf /etc/apache2/mods-enabled/`
    - `ln /etc/apache2/mods-available/proxy_http.load /etc/apache2/mods-enabled/`
    - `ln /etc/apache2/mods-available/rewrite.load /etc/apache2/mods-enabled/`
    - `ln /etc/apache2/mods-available/headers.load /etc/apache2/mods-enabled/`
    - You can enable SSL, to do so You need to follow SSL instruction in [SSL](https://github.com/cncf/devstats/blob/master/SSL.md) (that requires domain name).
    - `service apache2 restart`
    - Test via: `apt install links`, `links 127.0.0.1`
    - If you don't use Apache/SSL proxy, you should add EXTERNAL=1 when deploying - this will expose grafana to outside world by binding to `0.0.0.0` instead of local only `127.0.0.1`. See `devel/create_grafana.sh`.
12. Automatic deploy of first database (few examples)
    - You should probably use last two command examples or tweak any of commands listed below.
    - Top deploy one of CNCF projects see `projects.yaml` chose one `projectname` and specify to deploy only this project: `ONLY=projectname`.
    - Use `INIT=1` to specify that you want to initialize logs database `devstats`, you need to provide passowrd for admin `PG_PASS`, `ro_user` (`PG_PASS_RO` used by Grafana for read-only access) and `devstats_team` (`PG_PASS_TEAM` - this is just another role to allow readonly access from the bastion ssh server).
    - You must use `INIT=1` when running for the first time, this creates shared logs database and postgres users.
    - This will generate all data, without fetching anything from backups: `INIT=1 ONLY=projectname PG_PASS=... PG_PASS_RO=... PG_PASS_TEAM=... ./devel/deploy_all.sh`
    - You shoud add `SETPASS=1` on the first run to set main postgres database user password. Thsi requires user interaction, so it shouldn't be used from automatic scripts.
    - If you do not specify `ONLY="project1 project2 ... projectN"` it will deploy all projects defined in `projects.yaml`.
    - Use `SKIPWWW=1` to skip Apache/SSL config (which requires DNS name for a server), the final result will be Grafana via HTTP on port 3xxx on accessible by server IP or name.
    - Example with skipping GitHub API and custom ARTWORK: `ARTWORK=/data/artwork SKIPWWW=1 GHA2DB_GITHUB_OAUTH=- GHA2DB_GHAPISKIP=1 IGET=1 GET=1 ONLY=someproj PG_PASS=... ./devel/deploy_all.sh`
    - Other possible env variables used for automatic deploy are in the comments section in a first lines of deploy files: `devel/deploy_all.sh devel/deploy_proj.sh devel/create_databases.sh devel/init_database.sh devel/create_grafana.sh devel/create_www.sh all/add_project.sh`.
    - Example command that worked on a clean Ubuntu 18 without domain name: `EXTERNAL=1 SKIPTEMP=1 ARTWORK=/data/artwork INIT=1 PG_PASS=p PG_PASS_RO=p PG_PASS_TEAM=p ONLY=cncf GET=1 SKIPWWW=1 GHA2DB_GITHUB_OAUTH=- GHA2DB_GHAPISKIP=1 GHA2DB_PROJECTS_OVERRIDE="+cncf" ./devel/deploy_all.sh`.
    - Assuming you have GitHub OAuth file and artwork is in the default localtion and there is no domain name: `EXTERNAL=1 SKIPTEMP=1 INIT=1 PG_PASS=p PG_PASS_RO=p PG_PASS_TEAM=p ONLY=cncf GET=1 SKIPWWW=1 GHA2DB_PROJECTS_OVERRIDE="+cncf" ./devel/deploy_all.sh`.
    - You can also take a look at `ADDING_NEW_PROJECT.md` file for more info about setting up new projects.
    - Assuming that you deployed cncf project (like in the last example), you should see response using `links 127.0.0.1:3255`. You should albo be able to access your Grafana from outside world via: `http://ip:3255`
    - Now when postgres users are created, you test all stuff that require databases (on `cncf/devstats` repo): `PG_PASS=p make`.
    - Tests should pass.
13. To enable Continuous deployment using Travis, please follow instructions [here](https://github.com/cncf/devstats/blob/master/CONTINUOUS_DEPLOYMENT.md).
14. We need to setup cron job that will call sync every hour (10 minutes after 1:00, 2:00, ...)
    - Setup `devstats`, `webhook` (to handle CI/CD deployments) and `cron_db_backup_all.sh` cron jobs, similar to this:
    ```
    7 * * * * PATH=$PATH:/path/to/GOPATH/bin PG_PASS="..." devstats 2>> /tmp/gha2db_sync.err 1>> /tmp/gha2db_sync.log
    30 3 * * * PATH=$PATH:/path/to/GOPATH/bin cron_db_backup_all.sh 2>> /tmp/gha2db_backup.err 1>> /tmp/gha2db_backup.log
    */5 * * * * PATH=$PATH:/path/to/GOPATH/bin GOPATH=/go/path GHA2DB_CMDDEBUG=1 GHA2DB_PROJECT_ROOT=/path/to/repo GHA2DB_DEPLOY_BRANCHES="production,master" PG_PASS=... GHA2DB_SKIP_FULL_DEPLOY=1 webhook 2>> /tmp/gha2db_webhook.err 1>> /tmp/gha2db_webhook.log
    ```
    - First crontab entry is for automatic GHA sync.
    - Second crontab entry is for automatic daily backup of postgres databases.
    - Third crontab entry is for Continuous Deployment - this a Travis Web Hook listener server, it deploys project when specific conditions are met, details [here](https://github.com/cncf/devstats/blob/master/CONTINUOUS_DEPLOYMENT.md).
    - You need to change "..." `PG_PASS` to the real postgres password value and copy this line.
    - You need to change "/path/to/your/GOPATH/bin" to the value of "$GOPATH/bin", you cannot use $GOPATH in crontab directly.
    - Run `crontab -e` and put this line at the end of file and save.
    - Cron job will update Postgres database at 0:08, 1:08, ... 23:08 every day.
    - It outputs logs to `/tmp/gha2db_sync.log` and `/tmp/gha2db_sync.err` and also to gha Postgres `devstats` database: into table `gha_logs`.
    - Check database values and logs about 25 minutes after full hours, like 14:25:
    - Check max event created date: `select max(created_at) from gha_events` and logs `select * from gha_logs order by dt desc limit 20`.

# More details
- [Local Development](https://github.com/cncf/devstats/blob/master/DEVELOPMENT.md).
- [README](https://github.com/cncf/devstats/blob/master/README.md)
- [USAGE](https://github.com/cncf/devstats/blob/master/USAGE.md)
