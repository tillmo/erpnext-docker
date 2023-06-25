# ansible playbook for deployment of a dockerised ERPNext
following https://github.com/frappe/frappe_docker

## Deploy ERPNext

```
ansible-playbook -i hosts -l erpnext -v setUpERPNext.yml
```

## Create ERPNext site

```
cd ~/.cache/<deployment>
source .env
## the following command starts with a space to prevent it from ending up in the bash history
## (because of the admin password which is here "commented out")
 docker run --rm \
    -e "SITE_NAME=${SITES}" \
    -e "DB_ROOT_USER=root" \
    -e "MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}" \
    -e "ADMIN_PASSWORD=*****" \
    -e "INSTALL_APPS=erpnext" \
    -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites \
    --network ${PROJECT_NAME}_default \
    frappe/erpnext-worker:${ERPNEXT_VERSION} new
```

## Restore backup

The following command automatically restores the most recent backup that can be found in
`/home/docker/erpnext/backups`.

```
cd ~/.cache/<deployment>
source .env
docker run --rm -e "MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}" -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites -v /home/docker/erpnext/backups/${SITES}:/home/frappe/backups/${SITES} --network ${PROJECT_NAME}_default frappe/erpnext-worker:${ERPNEXT_VERSION} restore-backup
```

The script looks for backups matching the installed site(s).
For example, for the site `myserver.de` the command looks for "timestamp" folders in
`/home/docker/erpnext/backups/myserver.de`, e.g.
`/home/docker/erpnext/backups/myserver.de/20210407_000607`. Inside this folder it expects to
find the following files:
```
20210407_000607-myserver_de-database.sql.gz
20210407_000607-myserver_de-files.tar
20210407_000607-myserver_de-private-files.tar
```

## Remove missing apps after restoring backup

Note: The bench command `remove-from-installed-apps` doesn't work because it ignores apps that are not installed.

```
cd ~/.cache/<deployment>
source .env
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} uninstall-app --yes --force --no-backup journeys
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} uninstall-app --yes --force --no-backup erpnext_support
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} uninstall-app --yes --force --no-backup pibiapp
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} uninstall-app --yes --force --no-backup erpnextfints
```

## Migrate (e.g. after restoring older backup)

```
cd ~/.cache/<deployment>
source .env
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} migrate
```

## Maintenance mode

```
cd ~/.cache/<deployment>
source .env
docker run --rm -v ${PROJECT_NAME}_sites-vol:/home/frappe/frappe-bench/sites --network ${PROJECT_NAME}_default --user frappe frappe/erpnext-worker:${ERPNEXT_VERSION} bench --site ${SITES} set-maintenance-mode off
```

## Tear down ERPNext

```
cd ~/.cache/<deployment>
source .env
cd /home/docker/.cache/${PROJECT_NAME}
docker-compose --project-name ${PROJECT_NAME} -f installation/docker-compose-common.yml -f installation/docker-compose-erpnext.yml -f installation/docker-compose-networks.yml down
```

## Also remove all persistent volumes (WARNING! Destroys all data!)

```
cd ~/.cache/<deployment>
source .env
docker-compose --project-name ${PROJECT_NAME} -f installation/docker-compose-common.yml -f installation/docker-compose-erpnext.yml -f installation/docker-compose-networks.yml down --volumes
```

If you get the error message
```
ERROR: remove erpnext[...]_sites-vol: volume is in use - [...]
```
then there are still some left-over erpnext-worker containers from `docker run` commands. They
can be removed with
```
docker container prune
```
WARNING! This will remove _all_ stopped containers. You should check with `docker ps -a` if there are
stopped containers you want to keep. The alternative is to use `docker rm <container> ...` where
`<container> ...` are either the ids or the names of the containers to remove.

After removing the left-over erpnext-worker containers you can simply run the above `docker-compose`
command again.

## Restore files

```
tar xvf *de-private-files.tar
cd myserver.de/private/files; for f in *; do docker cp "$f" ${PROJECT_NAME}-erpnext-python-1:/home/frappe/frappe-bench/sites/${PROJECT_NAME}.myserver.de/private/files/ ; done; cd -
tar xvf *de-files.tar
cd myserver.de/public/files; for f in *; do docker cp "$f" ${PROJECT_NAME}-erpnext-python-1:/home/frappe/frappe-bench/sites/${PROJECT_NAME}.myserver.de/public/files/ ; done; cd -
```
