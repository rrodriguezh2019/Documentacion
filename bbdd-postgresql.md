# Pasos para levantar base de datos basica postgreSQL

## Primer paso crear una carpeta para nuestro proyecto

He decidido crearla dentro de la carpeta personal, usando los siguientes comando:
    
    mkdir -p ~/tfg-bases-de-datos-en-vr/db/init
    cd ~/tfg-bases-de-datos-en-vr


## Segundo paso crear el docker-compose.yml

Lo vamos a crear dentro de la carpeta de nuestro proyecto, y le vamos a meter la siguiente informacion:
    
    services:
      postgres:
        image: postgres:16
        container_name: tfg_vr_postgres
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
    
    volumes:
      pgdata:


## Tercer paso crear el esquema sql de la bbd:

Para ello lo vamos a hacer dentro de nuestra carpeta creamos un directorio que se db/init y dentro creamos el archivo 001_schema.sql

    create schema if not exists vr;
    
    create table if not exists vr.users (
      id            bigserial primary key,
      username      text not null unique,
      full_name     text not null,
      created_at    timestamptz not null default now()
    );
    
    create table if not exists vr.experiences (
      id            bigserial primary key,
      title         text not null,
      category      text not null,
      difficulty    int not null check (difficulty between 1 and 5),
      created_at    timestamptz not null default now()
    );
    
    create table if not exists vr.sessions (
      id             bigserial primary key,
      user_id        bigint not null references vr.users(id) on delete restrict,
      experience_id  bigint not null references vr.experiences(id) on delete restrict,
      started_at     timestamptz not null,
      ended_at       timestamptz,
      check (ended_at is null or ended_at > started_at)
    );
    
    create index if not exists idx_sessions_user_id on vr.sessions(user_id);
    create index if not exists idx_sessions_experience_id on vr.sessions(experience_id);
    create index if not exists idx_sessions_started_at on vr.sessions(started_at);



## Cuarto paso meter algunos datos de ejemplo:

Estos son unos pocos datos de ejemplo más adelante introduciremos cambios.

    insert into vr.users (username, full_name)
    values
      ('ana', 'Ana Pérez'),
      ('luis', 'Luis García'),
      ('marta', 'Marta López')
    on conflict (username) do nothing;
    
    insert into vr.experiences (title, category, difficulty)
    values
      ('Museo Virtual', 'cultura', 2),
      ('Simulador de Alturas', 'exposicion', 4),
      ('Entrenamiento Quirurgico', 'formacion', 5);
    
    insert into vr.sessions (user_id, experience_id, started_at, ended_at)
    values
      (
        (select id from vr.users where username = 'ana'),
        (select id from vr.experiences where title = 'Museo Virtual'),
        now() - interval '2 days',
        now() - interval '2 days' + interval '25 minutes'
      ),
      (
        (select id from vr.users where username = 'luis'),
        (select id from vr.experiences where title = 'Simulador de Alturas'),
        now() - interval '1 day',
        now() - interval '1 day' + interval '10 minutes'
      ),
      (
        (select id from vr.users where username = 'marta'),
        (select id from vr.experiences where title = 'Entrenamiento Quirurgico'),
        now() - interval '3 hours',
        null
      );
    SQL



##   Quinto paso levantar postgreSQL:

Nos metemos en la carpeta general que hemos creado no en ninguna de lass de detro sino en la general y lanzamos el siguiente codigo:

    docker-compose up -d
    docker ps

comprovamos que no haya fallos en los archivos creados mediante la revision de los logs:

    docker logs --tail=200 tfg_vr_postgres


## Sexto paso probamos a entrar a postgres:

Si lo logramos es que hemos levantado la bbdd de manera correcta.

    docker exec -it tfg_vr_postgres psql -U tfg -d tfg_vr

Una vez estemos dentro lo que vamos a hacer es comprobar que existen las tablas e intentar imprimir alguna:

    \dt vr.*

Después:

    select count(*) as users from vr.users;
    select count(*) as experiences from vr.experiences;
    select count(*) as sessions from vr.sessions;

Y por último para visualizar usamos unos joins:
    
    select
      s.id,
      u.username,
      e.title,
      s.started_at,
      s.ended_at
    from vr.sessions s
    join vr.users u on u.id = s.user_id
    join vr.experiences e on e.id = s.experience_id
    order by s.started_at desc;


Para poder salir \q






