# ansible playbook for deployment of a dockerised ERPNext
following https://github.com/frappe/frappe_docker

## Deploy ERPNext

```
ansible-playbook -i hosts -l erpnext -v setUpERPNext.yml
```

## Create ERPNext site

```
cd ~/.cache/<deployment>
source .env # todo: handle backquotes
## the following command starts with a space to prevent it from ending up in the bash history
## (because of the admin password which is here "commented out")
 docker exec ${PROJECT_NAME}-backend-1 bench new-site ${SITES} --db-root-password ${MYSQL_ROOT_PASSWORD} --admin-password <admin-password>
```

## Restore backup

The following script restores the most recent backup that can be found in
`/home/docker/erpnext/backups`.

```
### set variables
cd ~/.cache/<deployment>
source .env # todo: handle backquotes
DIR=~/erpnext/backups/${SITES}
SITES_U=${SITES//./_}
DATE=$(ls ${DIR} |sort |tail -n 1 |cut -d "-" -f 1)
DIR=${DIR}/${DATE}

### rename files to new site (only needed if restore happens across sites)
OLD_SITE=erpnext.cafesunshine.de
OLD_SITE_U=${OLD_SITE//./_}
pushd $DIR
tar xf ${DATE}-${OLD_SITE_U}-files.tar
mv $OLD_SITE $SITES
tar cf ${DATE}-${SITES_U}-files.tar $SITES
rm -rf $SITES
tar xf ${DATE}-${OLD_SITE_U}-private-files.tar
mv $OLD_SITE $SITES
tar cf ${DATE}-${SITES_U}-private-files.tar $SITES
rm -rf $SITES
popd

### copy files into docker volume and restore db+files
docker cp ${DIR}/${DATE}-${SITES_U}-database.sql.gz ${PROJECT_NAME}-backend-1:/home/frappe/frappe-bench/sites/
docker cp ${DIR}/${DATE}-${SITES_U}-files.tar ${PROJECT_NAME}-backend-1:/home/frappe/frappe-bench/sites/
docker cp ${DIR}/${DATE}-${SITES_U}-private-files.tar ${PROJECT_NAME}-backend-1:/home/frappe/frappe-bench/sites/
docker exec ${PROJECT_NAME}-backend-1 bench --site ${SITES} restore /home/frappe/frappe-bench/sites/${DATE}-${SITES_U}-database.sql.gz --db-root-password ${MYSQL_ROOT_PASSWORD} --with-public-files /home/frappe/frappe-bench/sites/${DATE}-${SITES_U}-files.tar --with-private-files /home/frappe/frappe-bench/sites/${DATE}-${SITES_U}-private-files.tar
docker cp ${DIR}/${DATE}-${SITES_U}-site_config_backup.json ${PROJECT_NAME}-backend-1:/home/frappe/frappe-bench/sites/${SITES}/site_config.json

### possibly needed
docker exec -it ${PROJECT_NAME}-db-1 mysql -u root -p${MYSQL_ROOT_PASSWORD} -e"GRANT ALL PRIVILEGES ON *.* TO '_b2b773e646de5d9f'@'%';"


## Migrate (e.g. after restoring older backup)

```
cd ~/.cache/<deployment>
source .env
docker exec ${PROJECT_NAME}-backend-1 bench --site ${SITES} migrate
```

## Maintenance mode

```
cd ~/.cache/<deployment>
source .env
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext:${ERPNEXT_VERSION} bench --site ${SITES} set-maintenance-mode off
```

## Start and tear down ERPNext

```
cd ~/.cache/<deployment>
source .env
cd /home/docker/.cache/${PROJECT_NAME}
# start
docker compose --project-name ${PROJECT_NAME} -f compose.yaml -f installation/compose.https.yaml -f overrides/compose.redis.yaml -f overrides/compose.mariadb.yaml up -d
# tear down
docker compose --project-name ${PROJECT_NAME} -f compose.yaml -f installation/compose.https.yaml -f overrides/compose.redis.yaml -f overrides/compose.mariadb.yaml down
```

## Also remove all persistent volumes (WARNING! Destroys all data!)

```
cd ~/.cache/<deployment>
source .env
docker compose --project-name ${PROJECT_NAME} -f compose.yaml -f installation/compose.https.yaml -f overrides/compose.redis.yaml -f overrides/compose.mariadb.yaml down --volumes
```

If you get the error message
```
ERROR: remove erpnext[...]_sites-vol: volume is in use - [...]
```
then there are still some left-over erpnext containers from `docker run` commands. They
can be removed with
```
docker container prune
```
WARNING! This will remove _all_ stopped containers. You should check with `docker ps -a` if there are
stopped containers you want to keep. The alternative is to use `docker rm <container> ...` where
`<container> ...` are either the ids or the names of the containers to remove.

After removing the left-over erpnext containers you can simply run the above `docker-compose`
command again.


## list docker processes

docker ps --format "table {{.ID}}\t {{.Image}}\t {{.Names}}\t {{.Command}}\t {{.Status}}"


## Restore files

```
tar xvf *de-private-files.tar
cd myserver.de/private/files; for f in *; do docker cp "$f" ${PROJECT_NAME}-erpnext-python-1:/home/frappe/frappe-bench/sites/${PROJECT_NAME}.myserver.de/private/files/ ; done; cd -
tar xvf *de-files.tar
cd myserver.de/public/files; for f in *; do docker cp "$f" ${PROJECT_NAME}-erpnext-python-1:/home/frappe/frappe-bench/sites/${PROJECT_NAME}.myserver.de/public/files/ ; done; cd -
```
