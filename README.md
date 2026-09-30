# Práctica 1 — Conceptos fundamentales de PostgreSQL
 
**Asignatura:** Administración y Diseño de Bases de Datos  
**Grado:** Ingeniería Informática — ULL
 
---
 
## 1. Creación de la base de datos
 
```sql
CREATE DATABASE biblioteca;
\c biblioteca
```
 
---
 
## 2. Creación de usuarios y roles
 
```sql
-- Crear usuario administrador
CREATE USER admin_biblio WITH PASSWORD 'admin1234' CREATEDB CREATEROLE;
GRANT ALL PRIVILEGES ON DATABASE biblioteca TO admin_biblio;
 
-- Crear usuario de solo lectura
CREATE USER usuario_biblio WITH PASSWORD 'user1234';
 
-- Crear rol lectores con permisos de consulta
CREATE ROLE lectores;
GRANT CONNECT ON DATABASE biblioteca TO lectores;
 
-- Asignar usuario_biblio al rol lectores
GRANT lectores TO usuario_biblio;
 
-- Consultar todos los usuarios y roles creados
SELECT rolname, rolsuper, rolcanlogin FROM pg_roles;
 
-- Cambiar la contraseña de usuario_biblio
ALTER USER usuario_biblio WITH PASSWORD 'nuevapassword456';
 
-- Revocar permiso de borrado a usuario_biblio
REVOKE DELETE ON ALL TABLES IN SCHEMA public FROM usuario_biblio;
```
 
---
 
## 3. Creación de tablas
 
```sql
-- Tabla autores
CREATE TABLE autores (
    id_autor      SERIAL PRIMARY KEY,
    nombre        VARCHAR(100),
    nacionalidad  VARCHAR(100)
);
 
-- Tabla libros
CREATE TABLE libros (
    id_libro         SERIAL PRIMARY KEY,
    titulo           VARCHAR(200),
    año_publicacion  INTEGER,
    id_autor         INTEGER NOT NULL,
    FOREIGN KEY (id_autor) REFERENCES autores(id_autor) ON DELETE CASCADE
);
 
-- Tabla prestamos
CREATE TABLE prestamos (
    id_prestamo          SERIAL PRIMARY KEY,
    id_libro             INTEGER NOT NULL,
    fecha_prestamo       DATE,
    fecha_devolucion     DATE,
    usuario_prestatario  VARCHAR(100),
    FOREIGN KEY (id_libro) REFERENCES libros(id_libro) ON DELETE CASCADE
);
 
-- Dar permisos de lectura al rol lectores sobre todas las tablas
GRANT SELECT ON ALL TABLES IN SCHEMA public TO lectores;
```
 
---
 
## 4. Inserción de datos
 
```sql
-- 5 autores
INSERT INTO autores (nombre, nacionalidad) VALUES
    ('Gabriel Garcia Marquez', 'Colombiano'),
    ('Isabel Allende',         'Chilena'),
    ('Jorge Luis Borges',      'Argentino'),
    ('Haruki Murakami',        'Japones'),
    ('George Orwell',          'Britanico');
 
-- 8 libros
INSERT INTO libros (titulo, año_publicacion, id_autor) VALUES
    ('Cien anos de soledad',              1967, 1),
    ('El amor en los tiempos del colera', 1985, 1),
    ('La casa de los espiritus',          1982, 2),
    ('Eva Luna',                          1987, 2),
    ('Ficciones',                         1944, 3),
    ('El Aleph',                          1949, 3),
    ('Norwegian Wood',                    1987, 4),
    ('1984',                              1949, 5);
 
-- 5 préstamos (algunos sin fecha de devolución = no devueltos)
INSERT INTO prestamos (id_libro, fecha_prestamo, fecha_devolucion, usuario_prestatario) VALUES
    (1, '2024-01-10', '2024-01-24', 'ana_garcia'),
    (2, '2024-02-05', NULL,         'carlos_lopez'),
    (3, '2024-02-15', '2024-03-01', 'ana_garcia'),
    (5, '2024-03-01', NULL,         'maria_perez'),
    (8, '2024-03-10', NULL,         'carlos_lopez');
```
 
---
 
## 5. Consultas básicas
 
### a) Listar todos los libros con su autor
 
```sql
SELECT titulo, nombre
FROM autores NATURAL JOIN libros;
```
 
### b) Préstamos sin fecha de devolución
 
```sql
SELECT id_prestamo, id_libro, fecha_prestamo, usuario_prestatario
FROM prestamos
WHERE fecha_devolucion IS NULL;
```
 
### c) Autores con más de un libro
 
```sql
SELECT nombre, COUNT(*) AS total_libros
FROM autores
JOIN libros ON autores.id_autor = libros.id_autor
GROUP BY nombre
HAVING COUNT(*) > 1;
```
 
---
 
## 6. Consultas con agregación
 
### a) Número total de préstamos
 
```sql
SELECT COUNT(*) AS prestamos_realizados
FROM prestamos;
```
 
### b) Número de libros prestados por cada usuario
 
```sql
SELECT usuario_prestatario, COUNT(*) AS numero_prestamos
FROM prestamos
GROUP BY usuario_prestatario
ORDER BY numero_prestamos DESC;
```
 
---
 
## 7. Modificación de datos
 
### a) Actualizar fecha de devolución de un préstamo pendiente
 
```sql
UPDATE prestamos
SET fecha_devolucion = '2026-03-28'
WHERE id_prestamo = 12;
```
 
### b) Eliminar un libro y comprobar el efecto en préstamos
 
```sql
-- Ver los préstamos del libro antes de borrarlo
SELECT * FROM prestamos WHERE id_libro = 2;
 
-- Eliminar el libro
DELETE FROM libros WHERE id_libro = 2;
 
-- Comprobar que sus préstamos se han borrado en cascada
SELECT * FROM prestamos WHERE id_libro = 2;
```
 
> Gracias a `ON DELETE CASCADE` definido en la tabla `prestamos`, al eliminar un libro se eliminan automáticamente todos sus registros de préstamos asociados.
 
---
 
## 8. Creación de vistas
 
### a) Crear la vista
 
```sql
CREATE VIEW vista_libros_prestados AS
SELECT titulo, nombre, usuario_prestatario
FROM autores NATURAL JOIN libros NATURAL JOIN prestamos;
```
 
### b) Conceder permisos sobre la vista a usuario_biblio
 
```sql
GRANT SELECT ON vista_libros_prestados TO usuario_biblio;
```
 
### Consultar la vista
 
```sql
SELECT * FROM vista_libros_prestados;
```
 
---
 
## 9. Funciones y consultas avanzadas
 
### a) Función que devuelve los libros de un autor
 
```sql
CREATE OR REPLACE FUNCTION autor_libros(autor VARCHAR(120))
RETURNS TABLE (titulo_libro VARCHAR(120))
AS $$
BEGIN
    RETURN QUERY
        SELECT titulo
        FROM autores NATURAL JOIN libros
        WHERE nombre = autor;
END;
$$ LANGUAGE plpgsql;
```
 
**Uso:**
 
```sql
SELECT * FROM autor_libros('Gabriel Garcia Marquez');
```
 
### b) Los tres libros más prestados
 
```sql
SELECT titulo, COUNT(*) AS veces_prestado
FROM libros NATURAL JOIN prestamos
GROUP BY titulo
ORDER BY veces_prestado DESC
LIMIT 3;
```
 
---
 
## 10. Exportación e importación de datos
 
### a) Exportar la tabla libros a CSV
 
```sql
\copy libros TO '/var/lib/postgresql/libros.csv' CSV HEADER;
```
 
### b) Importar autores desde CSV externo
 
Primero crear el archivo CSV en el sistema:
 
```bash
cat > /tmp/autores_nuevos.csv << EOF
nombre,nacionalidad
Fyodor Dostoievski,Ruso
Virginia Woolf,Britanica
Franz Kafka,Checo
EOF
 
sudo cp /tmp/autores_nuevos.csv /var/lib/postgresql/
```
 
Luego importarlo desde psql:
 
```sql
\copy autores (nombre, nacionalidad) FROM '/var/lib/postgresql/autores_nuevos.csv' CSV HEADER;
```
 
