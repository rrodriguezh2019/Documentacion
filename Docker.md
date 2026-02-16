## Pasos levantar PostgreSQL
- El primer paso es Levantar PostgreSQL como si fuera un servidor real

  - Limpiamos por si queda algo:
  
        docker rm -f tfg_postgres 2>/dev/null

  - Crear red Docker:
    
        docker network create tfg-net

  
  - Una vez limpio, levantamos esto:
    
        docker run -d \
        --name tfg_postgres \
        --network tfg-net \
        -e POSTGRES_USER=admin \
        -e POSTGRES_PASSWORD=admin \
        -e POSTGRES_DB=vrdb \
        -v pgdata:/var/lib/postgresql/data \
        -p 5555:5432 \
        postgres:15


  - Para conectarnos:

        psql -h localhost -p 5555 -U admin -d vrdb

  - Una vez conectados creamos las tablas y llenamos de datos:

        CREATE TABLE nodos (
          id SERIAL PRIMARY KEY,
          nombre TEXT,
          tipo TEXT,
          x FLOAT,
          y FLOAT,
          z FLOAT
        );


        INSERT INTO nodos (nombre, tipo, x, y, z)
        VALUES
        ('Usuarios', 'tabla', 0, 1, -3),
        ('Pedidos', 'tabla', 2, 0, -4),
        ('Productos', 'tabla', -2, 0, -4),
        ('VistaVentas', 'vista', 0, 2, -2);

- Una vez creado, hacemos un entorno virtual:

      python3 -m venv venv
      source venv/bin/activate
      psycopg2-binary

- Crear carpeta python y dockerfile

  - Dockerfile:

        FROM python:3.11-slim
        WORKDIR /app
        RUN pip install --no-cache-dir psycopg2-binary
        COPY test.py .
        CMD ["python", "test.py"]

  - test.py:
    
        import psycopg2
        import time
        
        # Esperar un poco para asegurarnos que PostgreSQL está listo
        time.sleep(5)
        
        conn = psycopg2.connect(
            host="tfg_postgres",
            port=5432,
            database="vrdb",
            user="admin",
            password="admin"
        )
        cur = conn.cursor()
        cur.execute("SELECT * FROM nodos;")
        rows = cur.fetchall()
        
        for row in rows:
            print(row)
        
        cur.close()
        conn.close()

- Construir imagen python:

      docker build -t python-backend .

- Ejecutar contenedor python en la red:
  
      docker run --rm --network tfg-net --name tfg_python python-backend

  
  


