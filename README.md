
## Requisitos previos

Para poder utilizar esta configuración es necesario contar con:

- [Docker](https://www.docker.com/) instalado en tu máquina.
- Una terminal de wsl para windows o una máquina linux

## Creación de clúster de HDFS con Hive e inicialización

Para crear el clúster, nos vamos a ayudar de un proyecto de github:
- https://github.com/big-data-europe/docker-hive/blob/master/

Ahora, vamos a hacer un git clone de este repositorio para tener todos los archivos

```bash
git clone https://github.com/big-data-europe/docker-hive/blob/master/
```

Para añadir más detalle, vamos a modificar el docker-compose para añadir yarn y map-reduce en nuestro clúster


```bash
version: "3"

services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop2.7.4-java8
    volumes:
      - namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop-hive.env
    ports:
      - "50070:50070"

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop2.7.4-java8
    volumes:
      - datanode:/hadoop/dfs/data
    env_file:
      - ./hadoop-hive.env
    environment:
      SERVICE_PRECONDITION: "namenode:50070"
    ports:
      - "50075:50075"

  resourcemanager:
    image: bde2020/hadoop-resourcemanager:2.0.0-hadoop2.7.4-java8
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
      - YARN_RESOURCEMANAGER_HOSTNAME=resourcemanager
    env_file:
      - ./hadoop-hive.env
    depends_on:
      - namenode
    ports:
      - "8088:8088" # Puerto para la interfaz web de YARN ResourceManager

  nodemanager:
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop2.7.4-java8
    environment:
      - CORE_CONF_fs_defaultFS=hdfs://namenode:8020
      - YARN_NODEMANAGER_HOSTNAME=nodemanager
    env_file:
      - ./hadoop-hive.env
    depends_on:
      - resourcemanager

  hive-server:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    environment:
      HIVE_CORE_CONF_javax_jdo_option_ConnectionURL: "jdbc:postgresql://hive-metastore/metastore"
      SERVICE_PRECONDITION: "hive-metastore:9083"
    ports:
      - "10000:10000"

  hive-metastore:
    image: bde2020/hive:2.3.2-postgresql-metastore
    env_file:
      - ./hadoop-hive.env
    command: /opt/hive/bin/hive --service metastore
    environment:
      SERVICE_PRECONDITION: "namenode:50070 datanode:50075 hive-metastore-postgresql:5432"
    ports:
      - "9083:9083"

  hive-metastore-postgresql:
    image: bde2020/hive-metastore-postgresql:2.3.0

volumes:
  namenode:
  datanode:


```

# Creando una tabla con un archivo .csv

Para crear la tabla, primero vamos a descargar un archivo csv para importar los datos con hdfs a hive. En mi caso, he escogido el archivo CatalogV2 de este enlace

- https://www.ibm.com/docs/es/scis?topic=samples-sample-csv-files

Ahora, vamos a añadir nuestro archivo csv a hdfs

```bash
docker cp catalogo.csv hdfs-powerbi-hive-server-1:/catalogo.csv
docker exec -it hdfs-powerbi-hive-server-1 bash
hdfs dfs -mkdir -p /user/hive/data
hdfs dfs -put /catalogo.csv /user/hive/data/
```

Podemos comprobar que catalogo.csv está en hdfs con el siguiente comando

```bash
hdfs dfs -ls /user/hive/data
```

Una vez tenemos nuestro archivo, ejecutamos hive

```bash
hive
```

Ahora, vamos a crear una tabla que consuma el archivo catalogo.csv en la base de datos default.

Para ello, debemos de seguir estos pasos:

1.- Crear la tabla catálogo, especificando sus campos, pero primero debemos de especificar que queremos usar la base de datos default

```bash
USE default;
CREATE TABLE catalogo (
    levelType STRING,
    code STRING,
    catalogType STRING,
    name STRING,
    description STRING,
    sourceLink STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/data';
```

3.- Hacemos una consulta simple a la tabla para ver que se han insertado los datos correctamente
```bash
SELECT * FROM test_table;
```

# Conexión de Hive con Power bi

Una vez tenemos nuestra base de datos, vamos a conectarla con Power Bi, y,  para ello, necesitaremos seguir lo siguientes pasos:

1.- Instalar Power Bi Desktop en la máquina en la que esté corriendo Hive

2.- Una vez instalado, iniciar sesión en la aplicación

3.- En la página principal, seleccionamos  obtener datos de otros orígenes y buscamos Hive LLAP para conectar nuestra base de datos

4.- En la configuración de conexión pondremos lo siguiente: 
- En el campo servidor ponemos localhost:<puerto> siendo el puerto en el cuál está escuchando hive (en nuestro caso es 1000)
- En base de datos ponemos default 
- En protocolo de transporte Thrift le damos estádar
- En modo conectividad lo dejamos en Importar.

Una vez conectado nuestro Hive, desplegará las bases de datos accesibles, en nuestro caso seleccionamos catalogo, le damos a cargar y ya tenemos la base de datos en Power Bi

1