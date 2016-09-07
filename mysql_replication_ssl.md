# Sécurisation SSL

## Serveur maitre

Connexion à la BDD

	mysql -u root -p

Création de l'utilisateur pour la réplication

	> CREATE USER 'replica_user'@'IP_SLAVE' IDENTIFIED BY 'password';
	> GRANT SELECT, SHOW DATABASES ON *.* TO 'replica_user'@'IP_SLAVE';
	> FLUSH PRIVILEGES;
	> exit

Génération des certificats SSL

	mkdir /etc/mysql/ssl/
	cd /etc/mysql/ssl/
	
	# Generate a CA key and certificate with SHA1 digest
	openssl genrsa 2048 > ca-key.pem
	openssl req -sha1 -new -x509 -nodes -days 3650 -key ca-key.pem > ca-cert.pem

	# Create server key and certficate with SHA1 digest, sign it and convert
	# the RSA key from PKCS #8 (OpenSSL 1.0 and newer) to the old PKCS #1 format
	openssl req -sha1 -newkey rsa:2048 -days 730 -nodes -keyout server-key.pem > server-req.pem # password : 2BfWMs3d282v4FpL
	openssl x509 -sha1 -req -in server-req.pem -days 730  -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > server-cert.pem
	openssl rsa -in server-key.pem -out server-key.pem

	# Create client key and certificate with SHA digest, sign it and convert
	# the RSA key from PKCS #8 (OpenSSL 1.0 and newer) to the old PKCS #1 format
	openssl req -sha1 -newkey rsa:2048 -days 730 -nodes -keyout client-key.pem > client-req.pem # password : 2BfWMs3d282v4FpL
	openssl x509 -sha1 -req -in client-req.pem -days 730 -CA ca-cert.pem -CAkey ca-key.pem -set_serial 01 > client-cert.pem
	openssl rsa -in client-key.pem -out client-key.pem

Modification de la configuration de MySQL pour la prise en compte des certificats

	nano /etc/mysql/my.cnf

Dans la partie **[mysqld]** ajouter :

	ssl-ca=/etc/mysql/ssl/ca-cert.pem
	ssl-cert=/etc/mysql/ssl/server-cert.pem
	ssl-key=/etc/mysql/ssl/server-key.pem

On redémarre le service MySQL

	service mysql restart

Connexion à la BDD

	mysql -u root -p

On vérifie que la configuration SSL est activée

	> show variables LIKE "%ssl%";

On devrait avoir 

> **have_openssl** : YES
> **have_ssl** : YES
> **ssl_ca** : /etc/mysql/ssl/ca-cert.pem
> **ssl_capath** :
> **ssl_cert** : /etc/mysql/ssl/server-cert.pem
> **ssl_cipher** :
> **ssl_key** : /etc/mysql/ssl/server-key.pem


## Serveur esclave

On récupère les fichiers de certificats que l'on a généré sur le serveur maitre :

> /etc/mysql/ssl/ca-cert.pem
> /etc/mysql/ssl/client-cert.pem
> /etc/mysql/ssl/client-key.pem

Et on les met dans le même répertoire
On redémarre le service MySQL

    service mysql restart

# Réplication

## Serveur maitre

Modification de la configuration de MySQL pour la réplication

	nano /etc/mysql/my.cnf

Dans la partie **[mysqld]** :

    bind-address = IP_MASTER
	server-id = 1
	log_bin = /var/log/mysql/mysql-bin.log
	binlog_do_db = BDD_A_REPLIQUER

On redémarre le service MySQL

    service mysql restart

Connexion à la BDD

	mysql -u root -p

Mise à jour de l'utilisateur créé pour la réplication

    > GRANT REPLICATION SLAVE ON *.* TO 'replica_user'@'IP_SLAVE' IDENTIFIED BY 'password';
	> FLUSH PRIVILEGES;

On met la base de donnée a répliquer en lecture seule

	> USE BDD_A_REPLIQUER;
	> FLUSH TABLES WITH READ LOCK;

On affiche le statut MASTER et on note les valeurs de **File** et **Position** qui nous serviront plus tard

    > SHOW MASTER STATUS;
	> exit

On fait un dump de la base à répliquer

    mysqldump -u root -p BDD_A_REPLIQUER > BDD_A_REPLIQUER.sql

Connexion à la BDD

	mysql -u root -p

On dévérouille la base de donnée a répliquer

	> USE BDD_A_REPLIQUER;
	> UNLOCK TABLES;

## Serveur esclave

Connexion à la BDD

	mysql -u root -p

On crée la base répliquée

	> CREATE DATABASE BDD_A_REPLIQUER;
	> exit

On importe le dump

	mysql -u root -p BDD_A_REPLIQUER < BDD_A_REPLIQUER.sql

Modification de la configuration de MySQL pour la réplication

	nano /etc/mysql/my.cnf

Dans la partie **[mysqld]** :

	server-id = 2    # il faut mettre un id différent du master
	log_bin = /var/log/mysql/mysql-bin.log
	replicate-do-db = BDD_A_REPLIQUER

On redémarre le service MySQL

    service mysql restart

Connexion à la BDD

	mysql -u root -p

On paramètre la réplication

    > CHANGE MASTER TO MASTER_HOST='IP_MASTER',MASTER_USER='replica_user', MASTER_PASSWORD='password', MASTER_SSL_CAPATH='/etc/mysql/ssl/', MASTER_LOG_FILE='VALEUR_FILE_MASTER', MASTER_LOG_POS=VALEUR_POSITION_MASTER, MASTER_SSL=1, MASTER_SSL_CA='/etc/mysql/ssl/ca-cert.pem', MASTER_SSL_CERT='/etc/mysql/ssl/client-cert.pem', MASTER_SSL_KEY='/etc/mysql/ssl/client-key.pem';
	> START SLAVE;

On affiche le statut SLAVE pour vérifier si la replication est bien activée

	> SHOW SLAVE STATUS\G
	> exit

Si vous avez Slave_IO_State: Waiting for master to send event cela veut dire que tout est OK

# Test de la réplication

Sur le serveur maitre on fait une modification de la BDD *(insert / update / delete)* et on vérifie sur le serveur esclave que cette modification est bien répercutée
