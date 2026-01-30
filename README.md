# ExamenParcial2BIM2

**Nombre:** David Israel González Hurtado
**Fecha:** 30/01/2026

---

## Codigo solucion en scala: 
 ```scala
import cats.effect.{IO, IOApp}
import doobie._
import doobie.implicits._
import fs2.io.file.{Files, Path}
import fs2.text

import model.equipos

object ExamenParcial2BIM2 extends IOApp.Simple {

  // ===============================
  // Conexión a Oracle
  // ===============================
  val xa: Transactor[IO] =
    Transactor.fromDriverManager[IO](
      "oracle.jdbc.OracleDriver",
      "jdbc:oracle:thin:@localhost:1521/XEPDB1",
      "equipo",
      "equipo",
      None
    )

  // ===============================
  // INSERT Doobie
  // ===============================
  def insertar(e: equipos): ConnectionIO[Int] =
    sql"""
      INSERT INTO equipos (
        id,
        equipo_local,
        equipo_visitante,
        fecha_partido,
        estadio,
        goles_local,
        goles_visitante,
        partido_jugado
      )
      VALUES (
        ${e.ID},
        ${e.Equipo_Local},
        ${e.Equipo_Visitante},
        TO_DATE(${e.Fecha_Partido}, 'YYYY-MM-DD'),
        ${e.Estadio},
        ${e.Goles_Local},
        ${e.Goles_Visitante},
        ${if (e.Partido_Jugado) 1 else 0}
      )
    """.update.run

  // ===============================
  // Programa principal
  // ===============================
  override def run: IO[Unit] = {

    val rutaCSV = Path(
      "C:\\Users\\Usuario\\OneDrive - Universidad Técnica Particular de Loja - UTPL\\Escritorio\\Programar\\Programacion\\ciclo_3_BIM1\\ExamenParcial2BIM2\\src\\main\\resources\\futbol.csv"
    )

    Files[IO]
      .readAll(rutaCSV)
      .through(text.utf8.decode)
      .through(text.lines)
      .drop(1) // Saltar cabecera
      .filter(_.nonEmpty)
      .evalMap { linea =>
        val columnas = linea.split(",")

        if (columnas.length != 8) {
          IO.println(s"Línea inválida: $linea")
        } else {

          val equip = equipos(
            columnas(0).toInt,
            columnas(1),
            columnas(2),
            columnas(3),
            columnas(4),
            columnas(5).toInt,
            columnas(6).toInt,
            columnas(7).toBoolean
          )

          insertar(equip)
            .transact(xa)
            .attempt
            .flatMap {
              case Right(_) =>
                IO.println(
                  s"Insertado: ${equip.Equipo_Local} vs ${equip.Equipo_Visitante}"
                )
              case Left(e) =>
                IO.println(s"Error BD: ${e.getMessage}")
            }
        }
      }
      .compile
      .drain
  }
}


 ```

---
## Script: 

### Creacion del usuario: 
```sql
-- Cambiar al contenedor correcto
ALTER SESSION SET CONTAINER = XEPDB1;

-- Crear el usuario
CREATE USER equipo IDENTIFIED BY equipo;

-- Darle cuota de 100 MB
ALTER USER equipo QUOTA 100M ON USERS;

-- Dar permisos
GRANT CONNECT, RESOURCE TO equipo;
```


### Creacion de la tabla: 
```sql

CREATE TABLE equipos (
    id               NUMBER(10) PRIMARY KEY,
    equipo_local     VARCHAR2(100) NOT NULL,
    equipo_visitante VARCHAR2(100) NOT NULL,
    fecha_partido    DATE NOT NULL,
    estadio          VARCHAR2(100),
    goles_local      NUMBER(2),
    goles_visitante  NUMBER(2),
    partido_jugado   NUMBER(1) NOT NULL
        CHECK (partido_jugado IN (0, 1))
);

```

### Consulta general: 

```sql
SELECT * FROM equipos

```

### Resultados de la consulta

<img width="1192" height="683" alt="image" src="https://github.com/user-attachments/assets/999c1983-3998-4c31-98d7-118e4b0b1576" />

---


