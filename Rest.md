Primero modificamos el archivo .yml para que tambien funcione con postgREST(esta subido al repo)


## 1. Preparación del Entorno (Limpieza Total)
Para garantizar un despliegue libre de conflictos de persistencia o errores de versiones previas, elimino los datos que habian 

    docker compose down -v
    docker compose up -d


## 2.Pgadmin

Abre tu navegador en http://localhost:5050.

Login: pgadmin4@pgadmin.org / Pass: admin (o lo que tengas en tu yaml).

Registrar Servidor:

    Clic derecho en Servers -> Register -> Server.

    En General: Nombre MiProyecto.

    En Connection:

        Host name: postgres_container (importante, usa el nombre del servicio de Docker).

        Username: postgres.

        Password: changeme.

    Clic en Save.

## 3.Crear la base de datos Universidad

Por defecto estarás dentro de la base de datos llamada postgres. Necesitamos crear la específica para tu TFG:

    Clic derecho sobre Databases -> Create -> Database...

    Nombre: Universidad

    Clic en Save.

## 4.Metemos los datos que tenemos como sql mediante la query toll que nos facilita pgadmin

## 5.Configuración de roles y permisos 

hora vamos a preparar a Postgres para que acepte las conexiones de la API.

Despliega el árbol de la izquierda y haz clic izquierdo sobre la base de datos "Universidad" para seleccionarla.

Abre el Query Tool
   
    -- Creamos el rol que usará PostgREST para entrar
    CREATE ROLE authenticator WITH LOGIN PASSWORD 'authenticator_password';
    
    -- Creamos el rol para usuarios que NO están logueados (lectura pública)
    CREATE ROLE web_anon NOLOGIN;
    
    -- Le decimos a authenticator que puede actuar como web_anon
    GRANT web_anon TO authenticator;
    
    -- Damos permiso para usar el esquema "public"
    GRANT USAGE ON SCHEMA public TO web_anon;


## 6. Dar permiso de lectura

    -- Esto le da permiso a la API para leer todas tus tablas
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO web_anon;

## 7. Reiniciamos postgres y probamos 

      docker restart postgrest_container



Ahora tenemos:

Listado de Estudiantes: http://localhost:3000/estudiantes

Listado de Profesores: http://localhost:3000/profesores



