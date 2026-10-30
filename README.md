# Moldes Spring Boot para el parcial

Guía de consulta rápida: API REST con **Spring Boot + Spring Data JPA + PostgreSQL**, con CRUD, relaciones, validaciones, excepciones y **transacciones**.

Basada en las guías del profesor:
- [Guía Spring Data JPA + Supabase](https://github.com/carlosreneas/guide_web/blob/master/Guia_Spring_Boot_Spring_Data_JPA_Supabase_v1.md)
- [Guía Transacciones (Semilleros)](https://github.com/carlosreneas/guide_web/blob/master/Guia_Spring_Boot_Transacciones_Semilleros.md)

---

## Cómo usar este README

| Marca | Significado |
|---|---|
| `// 🔧` | **Esto lo cambias** según el enunciado (nombres, campos, mensajes, reglas) |
| Sin marca | **Se copia tal cual**, no se toca |

Regla de oro: **cambia los nombres, no la estructura.** Cada clase sigue siempre el mismo molde.

## Índice

1. [Receta para atacar un enunciado](#1-receta-para-atacar-un-enunciado)
2. [Estructura de paquetes y qué se conecta con qué](#2-estructura-de-paquetes-y-qué-se-conecta-con-qué)
3. [Configuración (pom.xml y application.properties)](#3-configuración)
4. [Parte A: CRUD de una tabla (molde base)](#4-parte-a-crud-de-una-tabla-molde-base)
5. [Parte B: Relaciones y transacción (caso Semilleros completo)](#5-parte-b-relaciones-y-transacción-caso-semilleros-completo)
6. [Parte C: Excepciones y códigos HTTP](#6-parte-c-excepciones-y-códigos-http)
7. [Consultas derivadas (el "idioma" de los repository)](#7-consultas-derivadas)
8. [Diccionario de anotaciones](#8-diccionario-de-anotaciones)
9. [Transacciones: reglas y trampas](#9-transacciones-reglas-y-trampas)
10. [Errores comunes y cómo arreglarlos](#10-errores-comunes-y-cómo-arreglarlos)
11. [Pruebas en Postman](#11-pruebas-en-postman)
12. [Checklist final del parcial](#12-checklist-final-del-parcial)

---

## 1. Receta para atacar un enunciado

### Paso 1. Subraya el enunciado con dos colores

- **Sustantivos** (semillero, director, integrante, libro, préstamo...) → serán **entidades / tablas**.
- **Reglas** ("la sigla es única", "debe haber un solo líder", "mínimo un integrante") → serán **validaciones**.

### Paso 2. Dibuja las tablas y sus relaciones

Pregunta clave: **"¿uno a muchos?"**

- Un semillero tiene **muchos** integrantes; cada integrante pertenece a **un** semillero.
- La tabla del lado "muchos" es la que lleva la **llave foránea** (`semillero_id` en `integrantes`).
- En Java, la clase que tiene la llave foránea lleva `@ManyToOne`.

### Paso 3. Decide dónde va cada regla

| Tipo de regla | Dónde se revisa | Cómo |
|---|---|---|
| Campo vacío, formato de correo, tamaño | **DTO** (o Entity en CRUD simple) | `@NotBlank`, `@Email`, `@Size` |
| Lista no puede estar vacía | **DTO** | `@NotEmpty` |
| "Ya existe X" (sigla, correo repetido) | **Service** + `UNIQUE` en la BD | `repo.existsByX(...)` y `throw` |
| Comparar varios elementos del mismo JSON (un solo líder, códigos repetidos) | **Service** | `stream().filter().count()`, `HashSet` |
| "Debe existir X con ese id" | **Service** | `repo.findById(id).orElseThrow(...)` |

### Paso 4. Escribe las clases en este orden

1. SQL de las tablas (o Flyway)
2. **Entity** (una por tabla)
3. **Repository** (una por entity)
4. **DTO** (request y response) si el enunciado lo pide
5. **Excepciones** + `ApiExceptionHandler`
6. **Service** (validar → guardar → responder)
7. **Controller**

### Paso 5. Prueba con Postman

Primero el caso feliz (201/200), luego un caso por cada regla (400/404/409).

---

## 2. Estructura de paquetes y qué se conecta con qué

```
src/main/java/com/ejemplo/proyecto/          🔧 tu paquete base
├── ProyectoApplication.java                 (la crea Spring Initializr; NO moverla)
├── controller/
│   └── SemilleroController.java
├── dto/
│   ├── request/
│   │   ├── SemilleroRequest.java
│   │   ├── DirectorRequest.java
│   │   └── IntegranteRequest.java
│   └── response/
│       └── SemilleroResponse.java
├── service/
│   └── SemilleroService.java
├── repository/
│   ├── SemilleroRepository.java
│   ├── DirectorRepository.java
│   └── IntegranteRepository.java
├── model/                                   (o "entity": es lo mismo, depende del profe)
│   ├── Semillero.java
│   ├── Director.java
│   └── Integrante.java
└── exception/
    ├── ReglaNegocioException.java
    ├── ConflictoException.java
    ├── RecursoNoEncontradoException.java
    └── ApiExceptionHandler.java

src/main/resources/
├── application.properties
└── db/migration/                            (solo si usas Flyway)
    └── V1__crear_estructura.sql
```

> **Importante:** `ProyectoApplication` (la clase con `@SpringBootApplication`) debe estar en el paquete raíz. Los demás paquetes van **dentro** de él, si no Spring no los encuentra.

### Flujo de una petición

```
Postman ──JSON──▶ Controller ──DTO──▶ Service ──Entity──▶ Repository ──SQL──▶ Base de datos
                     │                  │
                     │                  └─ valida reglas + @Transactional
                     └─ @Valid revisa anotaciones del DTO
Postman ◀──JSON── Controller ◀──DTO Response── Service
```

### Quién "tiene" a quién (cada flecha es un atributo `private final` + constructor)

```
SemilleroController  ──tiene──▶  SemilleroService
SemilleroService     ──tiene──▶  SemilleroRepository, DirectorRepository
SemilleroRepository  ──maneja──▶ Semillero (Entity)
Semillero (Entity)   ──tiene──▶  Director (@ManyToOne) y lista de Integrante (@OneToMany)
Integrante (Entity)  ──tiene──▶  Semillero (@ManyToOne)
```

---

## 3. Configuración

### 3.1 Spring Initializr (https://start.spring.io)

- Project: **Maven** · Language: **Java** · Packaging: **Jar** · Java: **17 o 21**
- Dependencias a marcar:
  - **Spring Web**
  - **Spring Data JPA**
  - **Validation**
  - **PostgreSQL Driver**
  - **Flyway Migration** (solo si el enunciado lo pide)

### 3.2 Dependencias en `pom.xml`

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 3.3 `src/main/resources/application.properties`

```properties
spring.application.name=proyecto                          # 🔧
server.port=8080

# Conexión a la BD (con variables de entorno, recomendado)
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# Pool pequeño (Supabase gratis tiene pocas conexiones)
spring.datasource.hikari.maximum-pool-size=3
spring.datasource.hikari.minimum-idle=1

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false
```

**Valores posibles de `ddl-auto`:**

| Valor | Qué hace |
|---|---|
| `validate` | Solo **revisa** que las entities coincidan con las tablas. No cambia nada. (el de las guías) |
| `update` | Hibernate crea/modifica tablas solo. Cómodo para practicar, no para producción |
| `create` | Borra y recrea todo cada vez que arranca |
| `none` | No hace nada |

> Si el enunciado no te da tablas creadas, puedes usar `update` para que Hibernate las cree, o crearlas tú con SQL.

**Si no quieres variables de entorno (solo local):**

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mi_bd     # 🔧
spring.datasource.username=postgres                              # 🔧
spring.datasource.password=mi_clave                              # 🔧
```

**URL de Supabase (Session Pooler):**
```
jdbc:postgresql://aws-0-XXXX.pooler.supabase.com:5432/postgres?sslmode=require
usuario: postgres.[PROJECT_REF]   (no solo "postgres")
```

> Nunca subas contraseñas a GitHub. Agrega `.env` al `.gitignore`.

---

## 4. Parte A: CRUD de una tabla (molde base)

Ejemplo con `Producto`. **Reemplaza `Producto` por tu entidad** (y `productos` por tu tabla).

### 4.1 SQL

```sql
CREATE TABLE IF NOT EXISTS public.productos (          -- 🔧 nombre de la tabla
    id        BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    nombre    VARCHAR(100) NOT NULL,                   -- 🔧 columnas
    codigo    VARCHAR(30)  NOT NULL UNIQUE,            -- 🔧 UNIQUE = no se repite
    precio    NUMERIC(12,2) NOT NULL
);
```

### 4.2 Entity (`model/Producto.java`)

```java
package com.ejemplo.proyecto.model;                      // 🔧 tu paquete

import jakarta.persistence.*;
import jakarta.validation.constraints.*;

@Entity
@Table(name = "productos")                               // 🔧 nombre EXACTO de la tabla
public class Producto {                                  // 🔧 nombre de la clase (singular)

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "El nombre es obligatorio")      // 🔧 mensaje
    @Size(max = 100)
    @Column(name = "nombre", nullable = false, length = 100)
    private String nombre;                               // 🔧 atributo

    @NotBlank(message = "El código es obligatorio")
    @Column(name = "codigo", nullable = false, unique = true, length = 30)
    private String codigo;                               // 🔧

    @NotNull(message = "El precio es obligatorio")
    @Positive(message = "El precio debe ser mayor a 0")
    @Column(name = "precio", nullable = false)
    private Double precio;                               // 🔧

    // OBLIGATORIO: constructor vacío (JPA lo necesita)
    public Producto() {
    }

    // Getters y setters de TODOS los atributos 🔧
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }

    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }

    public Double getPrecio() { return precio; }
    public void setPrecio(Double precio) { this.precio = precio; }
}
```

> **Atajo:** en el IDE (VS Code / Antigravity / IntelliJ) → clic derecho → *Source Action → Generate Getters and Setters*.
>
> **Nombres de columnas:** Spring convierte `grupoInvestigacion` en `grupo_investigacion` automáticamente. Si tu columna se llama distinto, usa `@Column(name = "nombre_exacto")`.

### 4.3 Repository (`repository/ProductoRepository.java`)

```java
package com.ejemplo.proyecto.repository;                 // 🔧

import com.ejemplo.proyecto.model.Producto;              // 🔧
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface ProductoRepository extends JpaRepository<Producto, Long> {   // 🔧 <Entity, TipoDelId>

    // Spring escribe la consulta por ti leyendo el nombre del método (ver sección 7)
    boolean existsByCodigoIgnoreCase(String codigo);                          // 🔧
    List<Producto> findByNombreContainingIgnoreCase(String nombre);           // 🔧
}
```

Con solo `extends JpaRepository` ya tienes: `save`, `saveAndFlush`, `findById`, `findAll`, `existsById`, `count`, `delete`, `deleteById`.

### 4.4 Service CRUD (`service/ProductoService.java`)

```java
package com.ejemplo.proyecto.service;                    // 🔧

import com.ejemplo.proyecto.exception.ConflictoException;
import com.ejemplo.proyecto.exception.RecursoNoEncontradoException;
import com.ejemplo.proyecto.model.Producto;              // 🔧
import com.ejemplo.proyecto.repository.ProductoRepository;   // 🔧
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;   // ⚠ este import, NO el de jakarta

import java.util.List;

@Service
public class ProductoService {                           // 🔧

    private final ProductoRepository repo;               // 🔧 (la casilla)

    // Constructor: Spring te "entrega" el repository y tú lo guardas en la casilla
    public ProductoService(ProductoRepository repo) {    // 🔧
        this.repo = repo;
    }

    @Transactional(readOnly = true)
    public List<Producto> listar() {
        return repo.findAll();
    }

    @Transactional(readOnly = true)
    public Producto consultar(Long id) {
        return repo.findById(id)
                .orElseThrow(() -> new RecursoNoEncontradoException(
                        "Producto no encontrado con id: " + id));         // 🔧 mensaje
    }

    @Transactional
    public Producto crear(Producto nuevo) {
        nuevo.setId(null);                               // que la BD asigne el id
        if (repo.existsByCodigoIgnoreCase(nuevo.getCodigo())) {           // 🔧 regla
            throw new ConflictoException(
                    "Ya existe un producto con el código: " + nuevo.getCodigo());   // 🔧
        }
        return repo.saveAndFlush(nuevo);
    }

    @Transactional
    public Producto actualizar(Long id, Producto datos) {
        Producto actual = consultar(id);                 // si no existe, lanza 404

        // Si el código cambió, verificar que el nuevo no esté en uso   🔧
        if (!actual.getCodigo().equalsIgnoreCase(datos.getCodigo())
                && repo.existsByCodigoIgnoreCase(datos.getCodigo())) {
            throw new ConflictoException("El código ya está en uso");
        }

        actual.setNombre(datos.getNombre());             // 🔧 un setter por cada campo
        actual.setCodigo(datos.getCodigo());
        actual.setPrecio(datos.getPrecio());

        return repo.saveAndFlush(actual);
    }

    @Transactional
    public void eliminar(Long id) {
        Producto actual = consultar(id);
        repo.delete(actual);
    }

    @Transactional(readOnly = true)
    public List<Producto> buscarPorNombre(String nombre) {                // 🔧
        return repo.findByNombreContainingIgnoreCase(nombre);
    }
}
```

### 4.5 Controller CRUD (`controller/ProductoController.java`)

```java
package com.ejemplo.proyecto.controller;                 // 🔧

import com.ejemplo.proyecto.model.Producto;              // 🔧
import com.ejemplo.proyecto.service.ProductoService;     // 🔧
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/productos")                        // 🔧 ruta base
public class ProductoController {                        // 🔧

    private final ProductoService service;               // 🔧

    public ProductoController(ProductoService service) { // 🔧
        this.service = service;
    }

    // GET /api/productos
    @GetMapping
    public List<Producto> listar() {
        return service.listar();
    }

    // GET /api/productos/5
    @GetMapping("/{id}")
    public Producto consultar(@PathVariable Long id) {
        return service.consultar(id);
    }

    // POST /api/productos   (body JSON)
    @PostMapping
    public ResponseEntity<Producto> crear(@Valid @RequestBody Producto producto) {
        Producto nuevo = service.crear(producto);
        return ResponseEntity.status(HttpStatus.CREATED).body(nuevo);     // 201
    }

    // PUT /api/productos/5   (body JSON)
    @PutMapping("/{id}")
    public Producto actualizar(@PathVariable Long id, @Valid @RequestBody Producto producto) {
        return service.actualizar(id, producto);
    }

    // DELETE /api/productos/5
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)               // 204
    public void eliminar(@PathVariable Long id) {
        service.eliminar(id);
    }

    // GET /api/productos/buscar?nombre=lap
    @GetMapping("/buscar")
    public List<Producto> buscar(@RequestParam String nombre) {           // 🔧
        return service.buscarPorNombre(nombre);
    }
}
```

### 4.6 Qué significa cada línea del Controller y el Service

| Línea | Qué hace |
|---|---|
| `private final ProductoService service;` | Crea una "casilla" para guardar el service. `private` = solo esta clase la usa; `final` = una vez asignada no cambia |
| `public ProductoController(ProductoService service)` | Constructor. Spring entrega el service automáticamente (no haces `new`) |
| `this.service = service;` | Guarda lo recibido en la casilla. `this.service` es el atributo; `service` es el parámetro |
| `@RequestBody` | Convierte el JSON del cuerpo en un objeto Java |
| `@Valid` | Revisa las anotaciones (`@NotBlank`, etc.) antes de entrar al método. Si falla → 400 |
| `@PathVariable Long id` | Toma el número de la URL: `/api/productos/`**5** |
| `@RequestParam String nombre` | Toma el valor de `?nombre=...` en la URL |
| `ResponseEntity<Producto>` | La respuesta completa: **código de estado + cuerpo** |
| `ResponseEntity.status(HttpStatus.CREATED).body(x)` | "Responde con código 201 y mete `x` en el cuerpo" |
| `@ResponseStatus(HttpStatus.NO_CONTENT)` | Fija el código de la respuesta cuando el método devuelve `void` |

---

## 5. Parte B: Relaciones y transacción (caso Semilleros completo)

**Caso:** registrar en **una sola petición** un semillero con su director, líneas de investigación e integrantes. Si algo falla, **no debe quedar nada guardado**.

**Reglas:**
- R1. La sigla del semillero es única.
- R2. El correo del director debe ser válido.
- R3. Al menos una línea de investigación.
- R4. Al menos un integrante.
- R5. Los códigos de estudiantes no se repiten dentro del semillero.
- R6. Exactamente un líder.
- R7. Ningún error deja escrituras parciales.

### 5.1 SQL

```sql
CREATE TABLE directores (
    id     BIGSERIAL    PRIMARY KEY,
    nombre VARCHAR(160) NOT NULL,
    correo VARCHAR(180) NOT NULL UNIQUE
);

CREATE TABLE semilleros (
    id                  BIGSERIAL    PRIMARY KEY,
    nombre              VARCHAR(180) NOT NULL,
    sigla               VARCHAR(30)  NOT NULL UNIQUE,
    grupo_investigacion VARCHAR(180) NOT NULL,
    mision              TEXT,
    vision              TEXT,
    objetivo_general    TEXT,
    director_id         BIGINT       NOT NULL REFERENCES directores(id)
);

CREATE TABLE integrantes (
    id          BIGSERIAL    PRIMARY KEY,
    semillero_id BIGINT      NOT NULL REFERENCES semilleros(id),
    codigo      VARCHAR(30)  NOT NULL,
    nombre      VARCHAR(160) NOT NULL,
    programa    VARCHAR(160) NOT NULL,
    es_lider    BOOLEAN      NOT NULL DEFAULT FALSE,
    UNIQUE (semillero_id, codigo)
);

-- Versión simple de las líneas: una tabla con el texto de cada línea
CREATE TABLE semillero_lineas (
    semillero_id BIGINT       NOT NULL REFERENCES semilleros(id),
    linea        VARCHAR(160) NOT NULL
);
```

> Si usas Flyway: guarda esto en `src/main/resources/db/migration/V1__crear_estructura.sql`. **Nunca edites una migración ya ejecutada**; crea `V2__...`, `V3__...`.

### 5.2 Entities con relaciones

**Cómo pensar las relaciones:**

| Relación | Anotación | Quién la lleva |
|---|---|---|
| Muchos integrantes → un semillero | `@ManyToOne` + `@JoinColumn` | El lado "muchos" (el que tiene la FK) |
| Un semillero → muchos integrantes | `@OneToMany(mappedBy = "...")` | El lado "uno". `mappedBy` = nombre del atributo en la otra clase |
| Uno a uno | `@OneToOne` | Cualquiera, el que tiene la FK usa `@JoinColumn` |
| Muchos a muchos | `@ManyToMany` + `@JoinTable` | Cualquiera (ver nota abajo) |

**`Director.java`**

```java
package com.ejemplo.proyecto.model;

import jakarta.persistence.*;

@Entity
@Table(name = "directores")
public class Director {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nombre", nullable = false, length = 160)
    private String nombre;

    @Column(name = "correo", nullable = false, unique = true, length = 180)
    private String correo;

    public Director() {
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getCorreo() { return correo; }
    public void setCorreo(String correo) { this.correo = correo; }
}
```

**`Semillero.java`** (tiene un director, muchos integrantes y líneas)

```java
package com.ejemplo.proyecto.model;

import jakarta.persistence.*;

import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "semilleros")
public class Semillero {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "nombre", nullable = false, length = 180)
    private String nombre;

    @Column(name = "sigla", nullable = false, unique = true, length = 30)
    private String sigla;

    @Column(name = "grupo_investigacion", nullable = false, length = 180)
    private String grupoInvestigacion;

    @Column(name = "mision", columnDefinition = "TEXT")
    private String mision;

    @Column(name = "vision", columnDefinition = "TEXT")
    private String vision;

    @Column(name = "objetivo_general", columnDefinition = "TEXT")
    private String objetivoGeneral;

    // MUCHOS semilleros -> UN director (esta clase tiene la FK director_id)
    @ManyToOne(optional = false)
    @JoinColumn(name = "director_id")
    private Director director;

    // UN semillero -> MUCHOS integrantes
    // mappedBy = "semillero" -> se llama así el atributo dentro de Integrante
    // cascade ALL -> al guardar el semillero, se guardan también sus integrantes
    @OneToMany(mappedBy = "semillero", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Integrante> integrantes = new ArrayList<>();

    // Lista simple de textos guardada en otra tabla
    @ElementCollection
    @CollectionTable(name = "semillero_lineas", joinColumns = @JoinColumn(name = "semillero_id"))
    @Column(name = "linea", nullable = false, length = 160)
    private List<String> lineas = new ArrayList<>();

    public Semillero() {
    }

    // Método de ayuda: conecta AMBOS lados de la relación
    public void agregarIntegrante(Integrante integrante) {
        integrantes.add(integrante);
        integrante.setSemillero(this);
    }

    // Getters y setters de todos los atributos
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getSigla() { return sigla; }
    public void setSigla(String sigla) { this.sigla = sigla; }
    public String getGrupoInvestigacion() { return grupoInvestigacion; }
    public void setGrupoInvestigacion(String g) { this.grupoInvestigacion = g; }
    public String getMision() { return mision; }
    public void setMision(String mision) { this.mision = mision; }
    public String getVision() { return vision; }
    public void setVision(String vision) { this.vision = vision; }
    public String getObjetivoGeneral() { return objetivoGeneral; }
    public void setObjetivoGeneral(String o) { this.objetivoGeneral = o; }
    public Director getDirector() { return director; }
    public void setDirector(Director director) { this.director = director; }
    public List<Integrante> getIntegrantes() { return integrantes; }
    public void setIntegrantes(List<Integrante> integrantes) { this.integrantes = integrantes; }
    public List<String> getLineas() { return lineas; }
    public void setLineas(List<String> lineas) { this.lineas = lineas; }
}
```

**`Integrante.java`**

```java
package com.ejemplo.proyecto.model;

import jakarta.persistence.*;

@Entity
@Table(name = "integrantes")
public class Integrante {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "codigo", nullable = false, length = 30)
    private String codigo;

    @Column(name = "nombre", nullable = false, length = 160)
    private String nombre;

    @Column(name = "programa", nullable = false, length = 160)
    private String programa;

    @Column(name = "es_lider", nullable = false)
    private boolean esLider;

    // MUCHOS integrantes -> UN semillero (aquí está la FK semillero_id)
    @ManyToOne(optional = false)
    @JoinColumn(name = "semillero_id")
    private Semillero semillero;

    public Integrante() {
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getCodigo() { return codigo; }
    public void setCodigo(String codigo) { this.codigo = codigo; }
    public String getNombre() { return nombre; }
    public void setNombre(String nombre) { this.nombre = nombre; }
    public String getPrograma() { return programa; }
    public void setPrograma(String programa) { this.programa = programa; }
    public boolean isEsLider() { return esLider; }
    public void setEsLider(boolean esLider) { this.esLider = esLider; }
    public Semillero getSemillero() { return semillero; }
    public void setSemillero(Semillero semillero) { this.semillero = semillero; }
}
```

> **Nota `@ManyToMany`:** la guía usa una tabla `lineas` aparte y una tabla intermedia `semillero_lineas`. Si el enunciado lo exige:
> ```java
> @ManyToMany
> @JoinTable(name = "semillero_lineas",
>            joinColumns = @JoinColumn(name = "semillero_id"),
>            inverseJoinColumns = @JoinColumn(name = "linea_id"))
> private List<Linea> lineas = new ArrayList<>();
> ```

### 5.3 Repositories

```java
// SemilleroRepository.java
public interface SemilleroRepository extends JpaRepository<Semillero, Long> {
    boolean existsBySiglaIgnoreCase(String sigla);
}

// DirectorRepository.java
public interface DirectorRepository extends JpaRepository<Director, Long> {
    boolean existsByCorreoIgnoreCase(String correo);
}

// IntegranteRepository.java (opcional, para consultas)
public interface IntegranteRepository extends JpaRepository<Integrante, Long> {
    List<Integrante> findBySemilleroId(Long semilleroId);
}
```

Cada uno lleva `@Repository`, los imports de `JpaRepository`, `Repository` y de su entity.

### 5.4 DTOs (records)

**¿Por qué DTOs y no recibir la Entity?** Porque la Entity es la tabla (incluye ids, relaciones, listas que se referencian entre sí). El DTO es solo "lo que el cliente envía/recibe". Además evita errores de bucles infinitos al convertir a JSON.

`dto/request/DirectorRequest.java`
```java
package com.ejemplo.proyecto.dto.request;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record DirectorRequest(
        @NotBlank(message = "El nombre del director es obligatorio") String nombre,
        @NotBlank(message = "El correo es obligatorio")
        @Email(message = "Correo inválido") String correo) {
}
```

`dto/request/IntegranteRequest.java`
```java
package com.ejemplo.proyecto.dto.request;

import jakarta.validation.constraints.NotBlank;

public record IntegranteRequest(
        @NotBlank(message = "El código es obligatorio") String codigo,
        @NotBlank(message = "El nombre es obligatorio") String nombre,
        @NotBlank(message = "El programa es obligatorio") String programa,
        boolean esLider) {
}
```

`dto/request/SemilleroRequest.java`
```java
package com.ejemplo.proyecto.dto.request;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.util.List;

public record SemilleroRequest(
        @NotBlank String nombre,
        @NotBlank @Size(max = 30) String sigla,
        @NotBlank String grupoInvestigacion,
        String mision,
        String vision,
        String objetivoGeneral,
        @NotNull @Valid DirectorRequest director,                 // @Valid = revisa también el objeto interno
        @NotEmpty(message = "Debe haber al menos una línea") List<String> lineas,
        @NotEmpty(message = "Debe haber al menos un integrante")
        @Valid List<IntegranteRequest> integrantes) {             // @Valid = revisa cada elemento de la lista
}
```

`dto/response/SemilleroResponse.java`
```java
package com.ejemplo.proyecto.dto.response;

public record SemilleroResponse(Long id, String sigla, int totalIntegrantes, String estado) {
}
```

> Los nombres de los componentes del record deben coincidir con las claves del JSON (`grupoInvestigacion`, `esLider`...).

### 5.5 Service transaccional (el corazón del ejercicio)

```java
package com.ejemplo.proyecto.service;

import com.ejemplo.proyecto.dto.request.IntegranteRequest;
import com.ejemplo.proyecto.dto.request.SemilleroRequest;
import com.ejemplo.proyecto.dto.response.SemilleroResponse;
import com.ejemplo.proyecto.exception.ConflictoException;
import com.ejemplo.proyecto.exception.RecursoNoEncontradoException;
import com.ejemplo.proyecto.exception.ReglaNegocioException;
import com.ejemplo.proyecto.model.Director;
import com.ejemplo.proyecto.model.Integrante;
import com.ejemplo.proyecto.model.Semillero;
import com.ejemplo.proyecto.repository.DirectorRepository;
import com.ejemplo.proyecto.repository.SemilleroRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;   // ⚠ de Spring

import java.util.HashSet;
import java.util.Set;

@Service
public class SemilleroService {

    private final SemilleroRepository semilleroRepo;
    private final DirectorRepository directorRepo;

    public SemilleroService(SemilleroRepository semilleroRepo, DirectorRepository directorRepo) {
        this.semilleroRepo = semilleroRepo;
        this.directorRepo = directorRepo;
    }

    @Transactional      // <-- TODO el método es un paquete: todo o nada
    public SemilleroResponse crear(SemilleroRequest req) {

        // ============ 1. VALIDAR (antes de guardar nada) ============

        // R1: sigla única
        if (semilleroRepo.existsBySiglaIgnoreCase(req.sigla())) {
            throw new ConflictoException("Ya existe un semillero con la sigla: " + req.sigla());   // 409
        }

        // R6: exactamente un líder
        long lideres = req.integrantes().stream()
                .filter(IntegranteRequest::esLider)
                .count();
        if (lideres != 1) {
            throw new ReglaNegocioException(
                    "Debe haber exactamente un líder y se recibieron: " + lideres);                // 400
        }

        // R5: códigos de estudiantes no repetidos
        Set<String> codigos = new HashSet<>();
        for (IntegranteRequest i : req.integrantes()) {
            if (!codigos.add(i.codigo())) {          // add() devuelve false si ya estaba
                throw new ReglaNegocioException("Código de estudiante repetido: " + i.codigo());   // 400
            }
        }

        // ============ 2. GUARDAR (de lo independiente a lo dependiente) ============

        // 2.1 Director (no depende de nadie)
        Director director = new Director();
        director.setNombre(req.director().nombre());
        director.setCorreo(req.director().correo());
        director = directorRepo.save(director);

        // 2.2 Semillero (depende del director)
        Semillero semillero = new Semillero();
        semillero.setNombre(req.nombre());
        semillero.setSigla(req.sigla());
        semillero.setGrupoInvestigacion(req.grupoInvestigacion());
        semillero.setMision(req.mision());
        semillero.setVision(req.vision());
        semillero.setObjetivoGeneral(req.objetivoGeneral());
        semillero.setDirector(director);
        semillero.getLineas().addAll(req.lineas());

        // 2.3 Integrantes (dependen del semillero; cascade los guarda con él)
        for (IntegranteRequest i : req.integrantes()) {
            Integrante integrante = new Integrante();
            integrante.setCodigo(i.codigo());
            integrante.setNombre(i.nombre());
            integrante.setPrograma(i.programa());
            integrante.setEsLider(i.esLider());
            semillero.agregarIntegrante(integrante);   // conecta ambos lados
        }
        semillero = semilleroRepo.save(semillero);

        // ============ 3. RESPONDER ============
        return aResponse(semillero);
    }

    @Transactional(readOnly = true)
    public SemilleroResponse consultar(Long id) {
        Semillero s = semilleroRepo.findById(id)
                .orElseThrow(() -> new RecursoNoEncontradoException("Semillero no encontrado: " + id));   // 404
        return aResponse(s);
    }

    // Método privado de ayuda: Entity -> DTO de respuesta
    private SemilleroResponse aResponse(Semillero s) {
        return new SemilleroResponse(s.getId(), s.getSigla(), s.getIntegrantes().size(), "CREADO");
    }
}
```

**El patrón siempre es el mismo:**
1. **Validar** todo lo que se pueda con `if` + `throw`.
2. **Guardar** de lo independiente a lo dependiente.
3. **Responder** con un DTO.

### 5.6 Controller

```java
package com.ejemplo.proyecto.controller;

import com.ejemplo.proyecto.dto.request.SemilleroRequest;
import com.ejemplo.proyecto.dto.response.SemilleroResponse;
import com.ejemplo.proyecto.service.SemilleroService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/semilleros")
public class SemilleroController {

    private final SemilleroService service;

    public SemilleroController(SemilleroService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<SemilleroResponse> crear(@Valid @RequestBody SemilleroRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.crear(request));
    }

    @GetMapping("/{id}")
    public SemilleroResponse consultar(@PathVariable Long id) {
        return service.consultar(id);
    }
}
```

---

## 6. Parte C: Excepciones y códigos HTTP

### 6.1 Códigos HTTP que debes saber

| Código | Nombre | Cuándo |
|---|---|---|
| `200` | OK | GET / PUT exitoso |
| `201` | Created | POST que crea algo |
| `204` | No Content | DELETE exitoso (sin cuerpo) |
| `400` | Bad Request | Datos inválidos o regla de negocio incumplida |
| `404` | Not Found | No existe el recurso con ese id |
| `409` | Conflict | Duplicado (sigla, correo ya existe) |
| `500` | Internal Server Error | Error no controlado (bug) |

> Lanzar una excepción **no** devuelve automáticamente el código que quieres. Es el `ApiExceptionHandler` el que traduce cada excepción a su código.

### 6.2 Excepciones propias (`exception/`)

```java
// RecursoNoEncontradoException.java  -> 404
public class RecursoNoEncontradoException extends RuntimeException {
    public RecursoNoEncontradoException(String mensaje) { super(mensaje); }
}

// ConflictoException.java  -> 409
public class ConflictoException extends RuntimeException {
    public ConflictoException(String mensaje) { super(mensaje); }
}

// ReglaNegocioException.java  -> 400
public class ReglaNegocioException extends RuntimeException {
    public ReglaNegocioException(String mensaje) { super(mensaje); }
}
```

> Deben extender **`RuntimeException`**: así el `@Transactional` hace rollback automáticamente.

### 6.3 `ApiExceptionHandler.java` (traductor de errores)

```java
package com.ejemplo.proyecto.exception;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(RecursoNoEncontradoException.class)
    public ResponseEntity<Map<String, Object>> noEncontrado(RecursoNoEncontradoException ex) {
        return construir(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(ConflictoException.class)
    public ResponseEntity<Map<String, Object>> conflicto(ConflictoException ex) {
        return construir(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(ReglaNegocioException.class)
    public ResponseEntity<Map<String, Object>> reglaNegocio(ReglaNegocioException ex) {
        return construir(HttpStatus.BAD_REQUEST, ex.getMessage());
    }

    // Respaldo: la BD rechazó algo (UNIQUE, FK...)
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<Map<String, Object>> integridad(DataIntegrityViolationException ex) {
        return construir(HttpStatus.CONFLICT, "Conflicto de integridad de datos");
    }

    // Errores de @Valid (@NotBlank, @Email, @NotEmpty...)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> validacion(MethodArgumentNotValidException ex) {
        Map<String, String> errores = new HashMap<>();
        for (FieldError e : ex.getBindingResult().getFieldErrors()) {
            errores.put(e.getField(), e.getDefaultMessage());
        }
        Map<String, Object> cuerpo = new HashMap<>();
        cuerpo.put("timestamp", LocalDateTime.now());
        cuerpo.put("status", 400);
        cuerpo.put("error", "Validación fallida");
        cuerpo.put("detalles", errores);
        return ResponseEntity.badRequest().body(cuerpo);
    }

    // Método de ayuda para no repetir código
    private ResponseEntity<Map<String, Object>> construir(HttpStatus status, String mensaje) {
        Map<String, Object> cuerpo = new HashMap<>();
        cuerpo.put("timestamp", LocalDateTime.now());
        cuerpo.put("status", status.value());
        cuerpo.put("error", status.getReasonPhrase());
        cuerpo.put("message", mensaje);
        return ResponseEntity.status(status).body(cuerpo);
    }
}
```

**Para una excepción nueva:** (1) crea la clase que extienda `RuntimeException`, (2) agrega un `@ExceptionHandler` con el código HTTP que quieras.

---

## 7. Consultas derivadas

Spring **lee el nombre del método** y escribe el SQL por ti. Formato: `acción` + `By` + `Atributo` + `condición`. Los nombres de atributo son los de la **clase Java** (no los de la columna), con la primera letra en mayúscula.

| Método | Qué hace |
|---|---|
| `findByNombre(String n)` | `WHERE nombre = n` |
| `findByNombreIgnoreCase(String n)` | Igual, sin importar mayúsculas |
| `findByNombreContaining(String n)` | `LIKE %n%` |
| `findByNombreContainingIgnoreCase(String n)` | `LIKE %n%` sin mayúsculas |
| `findByNombreStartingWith(String n)` | Empieza por... |
| `findByPrecioGreaterThan(Double p)` | `precio > p` |
| `findByPrecioLessThanEqual(Double p)` | `precio <= p` |
| `findByPrecioBetween(Double a, Double b)` | Entre a y b |
| `findByNombreAndPrograma(String n, String p)` | Dos condiciones con AND |
| `findByNombreOrPrograma(String n, String p)` | Dos condiciones con OR |
| `findByProgramaOrderByNombreAsc(String p)` | Ordenado por nombre |
| `findByEsLiderTrue()` | `es_lider = true` |
| `findBySemilleroId(Long id)` | **Navega la relación:** integrantes cuyo `semillero.id = id` |
| `findBySemilleroSigla(String s)` | Integrantes cuyo `semillero.sigla = s` |
| `existsByCorreoIgnoreCase(String c)` | Devuelve `true/false` |
| `countByPrograma(String p)` | Devuelve cuántos hay |
| `deleteByCodigo(String c)` | Borra (necesita `@Transactional`) |
| `findFirstByOrderByPrecioDesc()` | El primero (el más caro) |
| `Optional<Producto> findByCodigo(String c)` | Devuelve uno o vacío |

### Consulta personalizada con JPQL (cuando el nombre no alcanza)

```java
@Query("SELECT p FROM Producto p WHERE p.precio > :min AND LOWER(p.nombre) LIKE LOWER(CONCAT('%', :texto, '%'))")
List<Producto> buscarAvanzado(@Param("min") Double min, @Param("texto") String texto);
```
Imports: `org.springframework.data.jpa.repository.Query` y `org.springframework.data.repository.query.Param`.

> En JPQL se usan los nombres de **clases y atributos Java** (`Producto p`, `p.precio`), no los de tablas y columnas.

### Paginación

```java
Page<Producto> findByNombreContainingIgnoreCase(String nombre, Pageable pageable);
```
```java
@GetMapping("/paginado")
public Page<Producto> paginado(@RequestParam String nombre,
                               @RequestParam(defaultValue = "0") int page,
                               @RequestParam(defaultValue = "5") int size) {
    return repo.findByNombreContainingIgnoreCase(nombre, PageRequest.of(page, size));
}
```
Imports: `org.springframework.data.domain.Page`, `Pageable`, `PageRequest`.

---

## 8. Diccionario de anotaciones

### Clases (qué es cada clase)

| Anotación | Dónde | Significado |
|---|---|---|
| `@SpringBootApplication` | Clase principal | Arranca la aplicación |
| `@RestController` | Controller | Atiende peticiones web y responde JSON |
| `@Service` | Service | "Aquí vive la lógica" |
| `@Repository` | Repository | "Aquí se accede a la BD" |
| `@Entity` | Entity | Esta clase es una tabla |
| `@RestControllerAdvice` | Handler de excepciones | Traduce excepciones a respuestas HTTP |

### Controller

| Anotación | Significado |
|---|---|
| `@RequestMapping("/api/x")` | Ruta base de todo el controller |
| `@GetMapping` | Responde a GET (leer) |
| `@PostMapping` | Responde a POST (crear) |
| `@PutMapping` | Responde a PUT (actualizar) |
| `@DeleteMapping` | Responde a DELETE (borrar) |
| `@PathVariable` | Toma un valor de la ruta `/{id}` |
| `@RequestParam` | Toma un valor de `?clave=valor` |
| `@RequestBody` | Convierte el JSON del cuerpo en un objeto |
| `@Valid` | Activa las validaciones del objeto |
| `@ResponseStatus` | Fija el código HTTP de la respuesta |

### Entity

| Anotación | Significado |
|---|---|
| `@Table(name = "x")` | Nombre exacto de la tabla |
| `@Id` | Clave primaria |
| `@GeneratedValue(strategy = GenerationType.IDENTITY)` | El id lo genera la BD (autoincremental) |
| `@Column(name, nullable, unique, length)` | Detalles de la columna |
| `@ManyToOne` | Muchos de este → uno de aquel (lleva la FK) |
| `@OneToMany(mappedBy = "x")` | Uno de este → muchos de aquel |
| `@OneToOne` | Uno a uno |
| `@ManyToMany` + `@JoinTable` | Muchos a muchos con tabla intermedia |
| `@JoinColumn(name = "x_id")` | Nombre de la columna de la llave foránea |
| `cascade = CascadeType.ALL` | Guardar/borrar al padre afecta a los hijos |
| `orphanRemoval = true` | Un hijo quitado de la lista se borra de la BD |
| `@ElementCollection` | Lista de datos simples (textos, números) en otra tabla |

### Validaciones (Bean Validation)

| Anotación | Significado |
|---|---|
| `@NotBlank` | Texto no nulo, no vacío, no solo espacios |
| `@NotNull` | No puede ser null |
| `@NotEmpty` | Lista/texto no vacío |
| `@Email` | Formato de correo |
| `@Size(min, max)` | Longitud del texto o tamaño de lista |
| `@Min(n)` / `@Max(n)` | Valor mínimo / máximo |
| `@Positive` | Mayor que 0 |
| `@Pattern(regexp = "...")` | Cumple una expresión regular |
| `@Valid` | Valida también los objetos/listas internos |

Todas llevan `message = "..."` para personalizar el error.

### Transacciones

| Anotación | Significado |
|---|---|
| `@Transactional` | Todo el método es una unidad: si hay error, se deshace |
| `@Transactional(readOnly = true)` | Solo consulta (más eficiente) |
| `@Transactional(rollbackFor = Exception.class)` | Hace rollback también con excepciones *checked* |

---

## 9. Transacciones: reglas y trampas

**Qué es:** un grupo de operaciones que se ejecutan como **una sola**. Si todas salen bien → *commit* (se guarda todo). Si una falla → *rollback* (se deshace todo).

**Ejemplo:** si guardas el director y falla al guardar el tercer integrante, el director **no** debe quedar en la BD.

### Reglas

1. Se pone sobre métodos **`public`** del **Service** (no en el Controller).
2. Por defecto, hace rollback con **`RuntimeException`** (y sus hijas). Con excepciones *checked* (`Exception`, `IOException`) **no**, salvo `rollbackFor = Exception.class`.
3. El import correcto es `org.springframework.transaction.annotation.Transactional` (hay otro de `jakarta`; el de Spring es el que usas aquí).
4. Valida **antes** de guardar: así evitas escribir cosas que luego tienes que deshacer.

### Trampas

| Trampa | Por qué falla |
|---|---|
| Atrapar la excepción con `try/catch` y no relanzarla | Spring no se entera del error → **no hay rollback** |
| Método `private` o `@Transactional` en el controller | No funciona (o no cubre lo que crees) |
| Llamar a un método `@Transactional` de la misma clase (`this.otroMetodo()`) | Se salta el proxy de Spring; la anotación se ignora |
| Lanzar una excepción *checked* | Sin `rollbackFor`, **no** deshace |
| Olvidar `extends RuntimeException` en tus excepciones | Mismo problema de arriba |

### Cómo probar el rollback (caso T7 de la guía)

1. Dentro de `crear(...)`, **justo después** de `directorRepo.save(director)`, escribe temporalmente:
   ```java
   throw new RuntimeException("Fallo simulado");
   ```
2. Cuenta los registros antes: `SELECT COUNT(*) FROM directores;`
3. Envía el POST válido: responde error 500.
4. Cuenta de nuevo: **debe ser igual** → el director no quedó guardado.
5. Quita el `throw` simulado.

### Duplicados simultáneos

La validación `existsBySigla` en Java **no basta** si dos peticiones llegan al mismo tiempo. Por eso la regla se refuerza con `UNIQUE` en la BD: si se cuela un duplicado, la BD lo rechaza y se captura `DataIntegrityViolationException` (409).

---

## 10. Errores comunes y cómo arreglarlos

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Schema-validation: missing table / column` | La entity no coincide con la tabla (`ddl-auto=validate`) | Revisa `@Table(name=...)` y `@Column(name=...)`; crea la tabla/columna |
| `No default constructor for entity` | Falta constructor vacío | Agrega `public Entidad() {}` |
| `No qualifying bean of type ...` | Falta `@Service`/`@Repository`, o la clase está fuera del paquete base | Agrega la anotación; mueve la clase dentro del paquete de `@SpringBootApplication` |
| `415 Unsupported Media Type` | Postman sin cabecera JSON | Body → raw → **JSON** |
| `400` con "JSON parse error" | JSON mal formado (coma de más, comillas) | Revisa el JSON |
| El `@Valid` no valida objetos internos | Falta `@Valid` en el atributo anidado | Pon `@Valid` en `director` y en `List<IntegranteRequest>` |
| `LazyInitializationException` | Se accede a una lista de la entity fuera de la transacción | Haz esa operación dentro del método del Service |
| Respuesta JSON infinita / `StackOverflow` | Entities con relaciones en ambos sentidos devueltas directo | Devuelve un **DTO de respuesta**, no la entity |
| `404` en una ruta que debería existir | Ruta mal escrita o falta `@RequestMapping`/`@GetMapping` | Compara la URL con las anotaciones |
| `password authentication failed` | Usuario sin sufijo del proyecto (Supabase) | Usa `postgres.[PROJECT_REF]` |
| `Network is unreachable` | Host directo (IPv6) | Usa el host del **Session Pooler** |
| `Port 8080 was already in use` | Otro proceso lo usa | `server.port=8081` |
| `null value in column ... violates not-null` | Falta asignar un campo requerido antes de guardar | Revisa los `setX(...)` del service |
| La FK queda en null | No conectaste ambos lados de la relación | Usa `semillero.agregarIntegrante(i)` (hace `setSemillero(this)`) |

---

## 11. Pruebas en Postman

Configuración para los POST/PUT: **Body → raw → JSON**.

### 11.1 Crear semillero (esperado: `201`)

`POST http://localhost:8080/api/semilleros`

```json
{
  "nombre": "Semillero de Aplicaciones Web",
  "sigla": "SAW-01",
  "grupoInvestigacion": "Grupo de Ingeniería de Software",
  "mision": "Promover la investigación aplicada",
  "vision": "Desarrollar soluciones universitarias",
  "objetivoGeneral": "Fortalecer competencias investigativas",
  "director": {
    "nombre": "Docente Ejemplo",
    "correo": "docente@universidad.edu"
  },
  "lineas": ["Aplicaciones Web", "Sistemas Distribuidos"],
  "integrantes": [
    { "codigo": "2026001", "nombre": "Ana Ejemplo",  "programa": "Ingeniería de Sistemas", "esLider": true },
    { "codigo": "2026002", "nombre": "Luis Ejemplo", "programa": "Ingeniería de Sistemas", "esLider": false }
  ]
}
```

### 11.2 Tabla de casos

| Caso | Qué envías | Esperado |
|---|---|---|
| T1 | JSON válido | `201`; se guardan todas las filas |
| T2 | Misma sigla otra vez | `409`; no hay filas nuevas |
| T3 | `"integrantes": []` | `400` |
| T4 | Dos integrantes con `esLider: true` | `400`; sin escrituras |
| T5 | Dos integrantes con el mismo `codigo` | `400`; sin escrituras |
| T6 | `"correo": "no-es-correo"` | `400` (validación del DTO) |
| T7 | `throw` simulado tras guardar el director | Error 500 y **sin director huérfano** |
| T8 | `GET /api/semilleros/999` | `404` |

### 11.3 CRUD simple (Producto)

| Acción | Método y ruta | Cuerpo |
|---|---|---|
| Crear | `POST /api/productos` | `{"nombre":"Mouse","codigo":"P-01","precio":25000}` |
| Listar | `GET /api/productos` | — |
| Uno | `GET /api/productos/1` | — |
| Actualizar | `PUT /api/productos/1` | JSON completo |
| Borrar | `DELETE /api/productos/1` | — |
| Buscar | `GET /api/productos/buscar?nombre=mou` | — |

### 11.4 Comandos de terminal

```bash
./mvnw spring-boot:run          # Linux / macOS
.\mvnw.cmd spring-boot:run      # Windows
mvn clean spring-boot:run       # con Maven instalado
```

Variables de entorno antes de ejecutar:

```bash
# Linux / macOS
export DB_URL="jdbc:postgresql://HOST:5432/postgres?sslmode=require"
export DB_USER="postgres.xxxx"
export DB_PASSWORD="mi_clave"
```
```powershell
# Windows PowerShell
$env:DB_URL="jdbc:postgresql://HOST:5432/postgres?sslmode=require"
$env:DB_USER="postgres.xxxx"
$env:DB_PASSWORD="mi_clave"
```

---

## 12. Checklist final del parcial

Antes de entregar, revisa en orden:

- [ ] Las dependencias están en el `pom.xml` y el proyecto compila.
- [ ] `application.properties` conecta a la BD (y no tiene contraseñas en el repo).
- [ ] Cada tabla tiene su **Entity** con `@Entity`, `@Table`, `@Id`, constructor vacío, getters y setters.
- [ ] Las relaciones tienen `@ManyToOne` en el lado de la FK y `@OneToMany(mappedBy=...)` en el otro.
- [ ] Cada Entity tiene su **Repository** que extiende `JpaRepository<Entity, Long>`.
- [ ] Los DTOs tienen sus validaciones y `@Valid` en los objetos/listas internos.
- [ ] El **Service** tiene `@Service`, constructor con los repositorios y `@Transactional` en los métodos que escriben.
- [ ] Las **reglas de negocio** están como `if` + `throw` **antes** de guardar.
- [ ] Las excepciones extienden `RuntimeException` y el `ApiExceptionHandler` las traduce a 400/404/409.
- [ ] El **Controller** tiene `@RestController`, `@RequestMapping`, constructor con el service y los códigos HTTP correctos (201 al crear, 204 al borrar).
- [ ] Probé el caso feliz y **un caso por cada regla**.
- [ ] Probé el **rollback** (si el enunciado habla de transacciones).
- [ ] Ningún `try/catch` se está "tragando" una excepción dentro de un método transaccional.

---

### Si te trabas en el parcial

Busca en internet con estas frases:
- `spring data jpa derived query methods`
- `spring boot @OneToMany @ManyToOne example`
- `spring boot @Transactional rollback`
- `spring boot @RestControllerAdvice exception handler`
- `spring boot bean validation @Valid nested object`

Documentación oficial: https://docs.spring.io/spring-data/jpa/reference/
