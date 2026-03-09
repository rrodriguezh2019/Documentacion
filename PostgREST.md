# Pasos para crear el puente (PostgREST)

## Primer paso crear roles y y servicios.

En la bbdd que habíamos creado inicialmente, vamos a añidir unas cuantas cosas, pero sin eliminar lo que ya habíamos creado.

no metemose en la siguinte direción de nuestra carpeta:

    cd ~/tfg-bases-de-datos-en-vr
    touch db/init/003_postgrest_roles.sql
    gedit db/init/003_postgrest_roles.sql


Una vez que hemos creado el archivo sql, dentro escribimos el siguiente código:

    -- Rol anónimo (solo lectura)
    do $$
    begin
      if not exists (select 1 from pg_roles where rolname = 'web_anon') then
        create role web_anon nologin;
      end if;
    end
    $$;
    
    -- Usuario con el que PostgREST se conecta a Postgres
    do $$
    begin
      if not exists (select 1 from pg_roles where rolname = 'authenticator') then
        create role authenticator noinherit login password 'authpass';
      end if;
    end
    $$;
    
    grant web_anon to authenticator;
    
    -- Permisos de lectura
    grant usage on schema vr to web_anon;
    grant select on all tables in schema vr to web_anon;
    
    -- Que futuras tablas en vr también tengan SELECT por defecto
    alter default privileges in schema vr grant select on tables to web_anon;

Con todo esto creado lo que tenemos que hacer es comprobar que tengamos levantado docker:

    docker ps -a

En nuestro caso como no lo teniamos, lo que tenemos que hacer el levantarlo:

    docker-compose up -d


Ahora ejecutamos el sql dentro del contenedor de docker:

    docker exec -i tfg_vr_postgres psql -U tfg -d tfg_vr < db/init/003_postgrest_roles.sql


Comprobamos que existen los roles:

    docker exec -it tfg_vr_postgres psql -U tfg -d tfg_vr -c "\du"



## Segundo paso añadir postgREST como puente en docker compose:

Tenemos que editar docker-compose.yml


    services:
      postgres:
        image: postgres:16
        container_name: tfg_vr_postgres
        command: ["postgres", "-c", "listen_addresses=0.0.0.0"]
        environment:
          POSTGRES_USER: tfg
          POSTGRES_PASSWORD: tfgpass
          POSTGRES_DB: tfg_vr
        ports:
          - "5432:5432"
        volumes:
          - pgdata:/var/lib/postgresql/data
          - ./db/init:/docker-entrypoint-initdb.d:ro
        healthcheck:
          test: ["CMD-SHELL", "pg_isready -U tfg -d tfg_vr"]
          interval: 5s
          timeout: 3s
          retries: 20
    
      postgrest:
        image: postgrest/postgrest:latest
        container_name: tfg_vr_postgrest
        depends_on:
          - postgres
        environment:
          PGRST_DB_URI: postgres://authenticator:authpass@tfg_vr_postgres:5432/tfg_vr
          PGRST_DB_SCHEMA: vr
          PGRST_DB_ANON_ROLE: web_anon
          PGRST_SERVER_PORT: 3000
        ports:
          - "3000:3000"
    
    volumes:
      pgdata:


## Tercer paso levantar PostREST:

    docker-compose up -d
    docker ps

## Cuarto paso probar endpoint y filtros



Probar los otros endpoints

    curl -s http://localhost:3000/experiences | head -c 400; echo
    curl -s http://localhost:3000/sessions | head -c 400; echo


Probar filtros 

    curl -s "http://localhost:3000/users?select=id,username&order=id.asc" | head -c 400; echo
    curl -s "http://localhost:3000/sessions?select=id,started_at&order=started_at.desc" | head -c 400; echo


## Quinto paso crear una view para ver los datos de manera mas sencilla


1) Crearla view


        docker exec -it tfg_vr_postgres psql -U tfg -d tfg_vr -c "create or replace view vr.session_detail_view as select s.id as session_id, u.username, u.full_name, e.title as experience_title, e.category, s.started_at, s.ended_at from vr.sessions s join vr.users u on u.id = s.user_id join vr.experiences e on e.id = s.experience_id;"

2) Dar permiso de lectura al rol anónimo (PostgREST)


        docker exec -it tfg_vr_postgres psql -U tfg -d tfg_vr -c "grant select on vr.session_detail_view to web_anon;"

3) Probar el endpoint REST


        curl -s http://localhost:3000/session_detail_view | head -c 600; echo


Si da algun fallo porque no se ha actualizado el docker probar a reiniciarlo:


    docker restart tfg_vr_postgrest
    docker logs --tail=100 tfg_vr_postgrest





