# HaProxy-Mysql-Replication
Docker ve HaProxy ile Çoklu Php ve Mysql Server
Bu sistem 3 loadbalancer, 1 master 3 slave mysql,3 php server oluşturur ve sistem hatalarına karşı yüksek güvenlik sağlar. 

Tüm dosyaları linux işletim sistemi klasörlerinden birine kaydedin.
Docker ve docker-compose un yüklü olduğundn emin olduktan sonra dosyaları kaydettiğiniz dizine gelin ve build.sh ( çalışmaz ise tırnaklar içerisindeki kodu çalıştırdıktan sonra "chmod +x  build.sh" tekrar deneyin.) dosyasını çalıştırın.
# Sistem Doğru olarak çalışmaz ise alttaki aşamaları sıralı takip ederek sistemsel hatanın nereden kaynaklandığını bulabilirsiniz #

# eski sistemi kaldır
docker-compose down -v

# eski sistem datalarını sil ( veritabanı yok olur )
rm -rf ./mysql_1_master/data/*
rm -rf ./mysql_2_slave/data/*
rm -rf ./mysql_3_slave/data/*
rm -rf ./mysql_4_slave/data/*

# sistem imajlarını oluştur
docker-compose build

# sistemi ayağa kaldır
docker-compose up -d

# php pdo kütüphanesi
docker exec web1 bash -c "docker-php-ext-install pdo pdo_mysql"
docker exec web2 bash -c "docker-php-ext-install pdo pdo_mysql"
docker exec web3 bash -c "docker-php-ext-install pdo pdo_mysql"

# haproxy container bash bağlan mysql client yükle
docker exec HaProxy1 bash -c "apt update"
docker exec HaProxy1 bash -c "apt install mysql-client -y"

docker exec HaProxy2 bash -c "apt update"
docker exec HaProxy2 bash -c "apt install mysql-client -y"

docker exec HaProxy3 bash -c "apt update"
docker exec HaProxy3 bash -c "apt install mysql-client -y"

# master mysql içinde replication userini oluştur ve yetki ver
docker exec mysql_1_master mysql -uroot -p111 -e 'CREATE USER "mydb_slave_user"@"%" IDENTIFIED WITH mysql_native_password BY "mydb_slave_pwd"; GRANT REPLICATION SLAVE ON *.* TO "mydb_slave_user"@"%"; FLUSH PRIVILEGES;'

#master verisi değişkenlere aktarılıyor ve slave MASTER_LOG_FILE ve MASTER_LOG_POS otomatik özelleştiriliyor
MS_STATUS=`docker exec mysql_1_master mysql -uroot -p111 -e 'SHOW MASTER STATUS';`
CURRENT_LOG=`echo $MS_STATUS | awk '{print $6}'`
CURRENT_POS=`echo $MS_STATUS | awk '{print $7}'`

#			MYSQL SLAVE		MASTER_LOG_FILE ve MASTER_LOG_POS özelleştir
docker exec mysql_2_slave mysql -uroot -p111 -e "CHANGE MASTER TO MASTER_HOST='mysql_1_master',MASTER_USER='mydb_slave_user',MASTER_PASSWORD='mydb_slave_pwd',MASTER_LOG_FILE='$CURRENT_LOG',MASTER_LOG_POS=$CURRENT_POS;start slave;";

docker exec mysql_3_slave mysql -uroot -p111 -e "CHANGE MASTER TO MASTER_HOST='mysql_1_master',MASTER_USER='mydb_slave_user',MASTER_PASSWORD='mydb_slave_pwd',MASTER_LOG_FILE='$CURRENT_LOG',MASTER_LOG_POS=$CURRENT_POS;start slave;";

docker exec mysql_4_slave mysql -uroot -p111 -e "CHANGE MASTER TO MASTER_HOST='mysql_1_master',MASTER_USER='mydb_slave_user',MASTER_PASSWORD='mydb_slave_pwd',MASTER_LOG_FILE='$CURRENT_LOG',MASTER_LOG_POS=$CURRENT_POS;start slave;";

# HaProxy user oluştur
docker exec mysql_1_master mysql -uroot -p111 -e "CREATE USER 'haproxy_user'@'%';"
docker exec mysql_1_master mysql -uroot -p111 -e "CREATE USER 'haproxy_root'@'%' IDENTIFIED WITH mysql_native_password BY 'Haproxymysql123';"

docker-compose restart web1 web2 web3 HaProxy1 HaProxy2 HaProxy3

#			Slave mysql replication durumunu gör
docker exec mysql_2_slave mysql -uroot -p111 -e 'SHOW SLAVE STATUS \G';
docker exec mysql_3_slave mysql -uroot -p111 -e 'SHOW SLAVE STATUS \G';
docker exec mysql_4_slave mysql -uroot -p111 -e 'SHOW SLAVE STATUS \G';

#			HaProxy user slave kontrol
docker exec mysql_2_slave mysql -uroot -p111 -e 'select DISTINCT User from mysql.user;';
docker exec mysql_3_slave mysql -uroot -p111 -e 'select DISTINCT User from mysql.user;';
docker exec mysql_4_slave mysql -uroot -p111 -e 'select DISTINCT User from mysql.user;';

# haproxy container bash bağlan mysql balancer kontrol yap
docker exec HaProxy1 bash -c "mysql -uroot -p111 -h 127.0.0.1 -e \"show variables like 'server_id'\""
docker exec HaProxy2 bash -c "mysql -uroot -p111 -h 127.0.0.1 -e \"show variables like 'server_id'\""
docker exec HaProxy3 bash -c "mysql -uroot -p111 -h 127.0.0.1 -e \"show variables like 'server_id'\""

#  !! herhangi bir kullanıcı ismi veya şifre değiştirildiğinde bunlara bağlı klasör isimleri, dosya isimleri ve .env dosya içeriklerininde değiştirilmesi gerektiğini unutmayın !!
