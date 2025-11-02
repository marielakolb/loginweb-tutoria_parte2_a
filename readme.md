# Guía para Construir una Aplicación de Tutorías en PHP con Patrón MVC (Extensión del Sistema de Login), Parte 2-a (Básica)

Esta guía es una extensión de la "Guía para Construir una Aplicación de Login en PHP con Patrón MVC" (README.md). Aquí, ampliamos el sistema de autenticación de usuarios para incluir un sistema de tutorías donde los usuarios pueden registrarse como tutores que ofrecen servicios en diversas asignaturas (matemáticas, lengua y literatura, idiomas, etc.), y otros usuarios autenticados pueden solicitar tutorías de los tutores disponibles.

El enfoque sigue el patrón MVC, manteniendo la simplicidad y basándose en el sistema de autenticación existente.

## ¿Qué es esta Extensión?

- **Usuarios**: Pueden registrarse, iniciar sesión y ver la lista de tutores disponibles.
- **Tutores**: Usuarios registrados que ofrecen tutorías en asignaturas específicas.
- **Solicitudes**: Usuarios autenticados pueden solicitar tutorías a tutores específicos en asignaturas disponibles.
- **Flujo**: Página pública muestra tutores; después de login, panel para solicitar tutorías.

## Paso 1: Preparar el Entorno (Igual que en README.md)

1. Crea una carpeta para tu proyecto, por ejemplo `tutoriaApp`.
2. Dentro de la carpeta, crea las subcarpetas: `models`, `views`, `controllers`.
3. Asegúrate de tener un servidor web (como WAMP o XAMPP) y una base de datos MySQL.

## Paso 2: Crear la Base de Datos

Ejecuta estas consultas SQL en tu base de datos para crear las tablas necesarias. Mantén la tabla `usuarios` del original y agrega las nuevas.

```sql
CREATE DATABASE bdnegocio;

USE bdnegocio;

-- Tabla de usuarios (del original)
CREATE TABLE usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(50) NOT NULL UNIQUE,
    pass VARCHAR(255) NOT NULL
);

-- Nueva tabla de asignaturas
CREATE TABLE asignaturas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL UNIQUE
);

-- Nueva tabla de tutores (relacionada con usuarios)
CREATE TABLE tutores (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    descripcion TEXT,
    FOREIGN KEY (user_id) REFERENCES usuarios(id) ON DELETE CASCADE
);

-- Nueva tabla de relación muchos-a-muchos entre tutores y asignaturas
CREATE TABLE tutor_asignaturas (
    tutor_id INT NOT NULL,
    asignatura_id INT NOT NULL,
    PRIMARY KEY (tutor_id, asignatura_id),
    FOREIGN KEY (tutor_id) REFERENCES tutores(id) ON DELETE CASCADE,
    FOREIGN KEY (asignatura_id) REFERENCES asignaturas(id) ON DELETE CASCADE
);

-- Nueva tabla de solicitudes de tutorías
CREATE TABLE solicitudes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    tutor_id INT NOT NULL,
    asignatura_id INT NOT NULL,
    estado ENUM('pendiente', 'aceptada', 'rechazada') DEFAULT 'pendiente',
    fecha_solicitud TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES usuarios(id) ON DELETE CASCADE,
    FOREIGN KEY (tutor_id) REFERENCES tutores(id) ON DELETE CASCADE,
    FOREIGN KEY (asignatura_id) REFERENCES asignaturas(id) ON DELETE CASCADE
);

-- Insertar algunas asignaturas de ejemplo
INSERT INTO asignaturas (nombre) VALUES ('Matemáticas'), ('Lengua y Literatura'), ('Idiomas'), ('Ciencias'), ('Historia');
```

### Prueba
- Crear un archivo temporal `test_db.php` con:
  ```php
  <?php
  require_once('config.php');
  try {
      $stmt = $con->query("SELECT * FROM asignaturas");
      $result = $stmt->fetchAll(PDO::FETCH_ASSOC);
      print_r($result);
  } catch (PDOException $e) {
      echo "Error: " . $e->getMessage();
  }
  ?>
  ```
- Acceder a `http://localhost/tu_proyecto/test_db.php` y verificar que se muestren las asignaturas insertadas.
- Si funciona, eliminar `test_db.php`.

## Paso 3: Crear el Archivo de Configuración (Igual que en README.md)

Usa el mismo `config.php` del original para la conexión a la base de datos.

## Paso 4: Crear los Modelos

### models/UserModel.php (Igual que en README.md)

Mantén el modelo de usuarios del original.

### models/TutorModel.php (Nuevo)

Maneja operaciones relacionadas con tutores y asignaturas.

```php
<?php
require_once('config.php'); // Incluye el archivo de configuración para conectar a la base de datos

class TutorModel { // Define una clase para manejar datos de tutores
    private $conn; // Propiedad privada para almacenar la conexión a la base de datos

    public function __construct() { // Método que se ejecuta al crear una instancia de la clase
        global $con; // Accede a la variable global de conexión
        $this->conn = $con; // Asigna la conexión a la propiedad
    }

    // Obtener todos los tutores con sus asignaturas
    public function getAllTutors() { // Método para obtener la lista de todos los tutores
        $stmt = $this->conn->prepare(" // Prepara una consulta SQL compleja para unir tablas
            SELECT t.id, u.nombre, t.descripcion, GROUP_CONCAT(a.nombre SEPARATOR ', ') AS asignaturas // Selecciona datos del tutor y agrupa asignaturas
            FROM tutores t // Desde la tabla tutores
            JOIN usuarios u ON t.user_id = u.id // Une con usuarios por ID
            LEFT JOIN tutor_asignaturas ta ON t.id = ta.tutor_id // Une con asignaturas del tutor
            LEFT JOIN asignaturas a ON ta.asignatura_id = a.id // Une con nombres de asignaturas
            GROUP BY t.id // Agrupa por tutor
        ");
        $stmt->execute(); // Ejecuta la consulta
        return $stmt->fetchAll(PDO::FETCH_ASSOC); // Devuelve todos los resultados como array asociativo
    }

    // Registrar un tutor
    public function registerTutor($userId, $descripcion, $asignaturas) { // Método para registrar un nuevo tutor
        // Insertar tutor
        $stmt = $this->conn->prepare("INSERT INTO tutores (user_id, descripcion) VALUES (:user_id, :descripcion)"); // Prepara inserción en tabla tutores
        $stmt->bindParam(':user_id', $userId); // Vincula el ID del usuario
        $stmt->bindParam(':descripcion', $descripcion); // Vincula la descripción
        $stmt->execute(); // Ejecuta la inserción
        $tutorId = $this->conn->lastInsertId(); // Obtiene el ID del tutor recién insertado

        // Insertar asignaturas
        foreach ($asignaturas as $asignaturaId) { // Para cada asignatura seleccionada
            $stmt = $this->conn->prepare("INSERT INTO tutor_asignaturas (tutor_id, asignatura_id) VALUES (:tutor_id, :asignatura_id)"); // Prepara inserción en tabla relación
            $stmt->bindParam(':tutor_id', $tutorId); // Vincula ID del tutor
            $stmt->bindParam(':asignatura_id', $asignaturaId); // Vincula ID de la asignatura
            $stmt->execute(); // Ejecuta la inserción
        }
        return true; // Retorna verdadero si todo salió bien
    }

    // Obtener asignaturas disponibles
    public function getAsignaturas() { // Método para obtener lista de asignaturas
        $stmt = $this->conn->prepare("SELECT id, nombre FROM asignaturas"); // Prepara consulta para seleccionar asignaturas
        $stmt->execute(); // Ejecuta la consulta
        return $stmt->fetchAll(PDO::FETCH_ASSOC); // Devuelve las asignaturas como array
    }
}
?>
```

### models/RequestModel.php (Nuevo)

Maneja solicitudes de tutorías.

```php
<?php
require_once('config.php'); // Incluye configuración para conectar a la base de datos

class RequestModel { // Define clase para manejar solicitudes
    private $conn; // Propiedad privada para la conexión

    public function __construct() { // Constructor que se ejecuta al crear la clase
        global $con; // Accede a la conexión global
        $this->conn = $con; // Asigna la conexión
    }

    // Crear una solicitud
    public function createRequest($userId, $tutorId, $asignaturaId) { // Método para crear nueva solicitud
        $stmt = $this->conn->prepare("INSERT INTO solicitudes (user_id, tutor_id, asignatura_id) VALUES (:user_id, :tutor_id, :asignatura_id)"); // Prepara inserción en tabla solicitudes
        $stmt->bindParam(':user_id', $userId); // Vincula ID del usuario
        $stmt->bindParam(':tutor_id', $tutorId); // Vincula ID del tutor
        $stmt->bindParam(':asignatura_id', $asignaturaId); // Vincula ID de la asignatura
        $stmt->execute(); // Ejecuta la inserción
        return true; // Retorna éxito
    }

    // Obtener solicitudes de un usuario
    public function getUserRequests($userId) { // Método para obtener solicitudes del usuario
        $stmt = $this->conn->prepare(" // Prepara consulta para unir tablas y obtener datos
            SELECT s.id, u.nombre AS tutor, a.nombre AS asignatura, s.estado, s.fecha_solicitud // Selecciona campos específicos
            FROM solicitudes s // Desde tabla solicitudes
            JOIN tutores t ON s.tutor_id = t.id // Une con tutores
            JOIN usuarios u ON t.user_id = u.id // Une con usuarios
            JOIN asignaturas a ON s.asignatura_id = a.id // Une con asignaturas
            WHERE s.user_id = :user_id // Filtra por usuario
        ");
        $stmt->bindParam(':user_id', $userId); // Vincula ID del usuario
        $stmt->execute(); // Ejecuta la consulta
        return $stmt->fetchAll(PDO::FETCH_ASSOC); // Devuelve resultados como array
    }
}
?>
```

### Prueba
- Crear `test_models.php`:
  ```php
  <?php
  require_once('config.php');
  require_once('models/TutorModel.php');
  $tutorModel = new TutorModel();
  $asignaturas = $tutorModel->getAsignaturas();
  print_r($asignaturas);
  ?>
  ```
- Acceder a `http://localhost/tu_proyecto/test_models.php` y verificar asignaturas.
- Probar `getAllTutors()` (debería retornar vacío inicialmente).
- Si funciona, eliminar `test_models.php`.

## Paso 5: Crear los Controladores

### controllers/AuthController.php (Igual que en README.md)

Mantén el controlador de autenticación del original.

### controllers/HomeController.php (Modificado)

Actualiza para mostrar la lista de tutores en la página principal.

```php
<?php
require_once('models/TutorModel.php'); // Incluye el modelo de tutores para usar sus métodos

class HomeController { // Define la clase para manejar la página principal
    public function index() { // Método que se ejecuta para mostrar la página de inicio
        $tutorModel = new TutorModel(); // Crea una instancia del modelo de tutores
        $tutors = $tutorModel->getAllTutors(); // Obtiene la lista de todos los tutores
        include('views/home.php'); // Incluye la vista de la página principal
    }
}
?>
```

### controllers/TutorController.php (Nuevo)

Maneja registro de tutores y solicitudes.

```php
<?php
require_once('models/TutorModel.php'); // Incluye modelo de tutores
require_once('models/RequestModel.php'); // Incluye modelo de solicitudes

class TutorController { // Define clase para controlar acciones de tutores
    private $tutorModel; // Propiedad para modelo de tutores
    private $requestModel; // Propiedad para modelo de solicitudes

    public function __construct() { // Constructor que inicializa modelos
        $this->tutorModel = new TutorModel(); // Crea instancia de TutorModel
        $this->requestModel = new RequestModel(); // Crea instancia de RequestModel
    }

    // Mostrar formulario para registrarse como tutor
    public function registerForm() { // Método para mostrar formulario de registro de tutor
        session_start(); // Inicia sesión
        if (!isset($_SESSION['user_id'])) { // Si no hay usuario logueado
            header("Location: index.php?action=login"); // Redirige a login
            exit(); // Termina ejecución
        }
        $asignaturas = $this->tutorModel->getAsignaturas(); // Obtiene asignaturas
        include('views/register_tutor.php'); // Incluye vista del formulario
    }

    // Procesar registro de tutor
    public function register() { // Método para procesar registro
        session_start(); // Inicia sesión
        if (!isset($_SESSION['user_id'])) { // Verifica login
            header("Location: index.php?action=login"); // Redirige si no logueado
            exit(); // Termina
        }

        if ($_SERVER["REQUEST_METHOD"] == "POST") { // Si es petición POST
            $descripcion = $_POST['descripcion']; // Obtiene descripción
            $asignaturas = $_POST['asignaturas']; // Obtiene asignaturas seleccionadas
            $this->tutorModel->registerTutor($_SESSION['user_id'], $descripcion, $asignaturas); // Registra tutor
            header("Location: index.php?action=home"); // Redirige a home
            exit(); // Termina
        }
    }

    // Mostrar formulario para solicitar tutoría
    public function requestForm($tutorId) { // Método para mostrar formulario de solicitud
        session_start(); // Inicia sesión
        if (!isset($_SESSION['user_id'])) { // Verifica login
            header("Location: index.php?action=login"); // Redirige si no logueado
            exit(); // Termina
        }
        $asignaturas = $this->tutorModel->getAsignaturas(); // Obtiene asignaturas
        include('views/request_tutoria.php'); // Incluye vista del formulario
    }

    // Procesar solicitud
    public function request() { // Método para procesar solicitud
        session_start(); // Inicia sesión
        if (!isset($_SESSION['user_id'])) { // Verifica login
            header("Location: index.php?action=login"); // Redirige si no logueado
            exit(); // Termina
        }

        if ($_SERVER["REQUEST_METHOD"] == "POST") { // Si es POST
            $tutorId = $_POST['tutor_id']; // Obtiene ID del tutor
            $asignaturaId = $_POST['asignatura_id']; // Obtiene ID de asignatura
            $this->requestModel->createRequest($_SESSION['user_id'], $tutorId, $asignaturaId); // Crea solicitud
            header("Location: index.php?action=my_requests"); // Redirige a mis solicitudes
            exit(); // Termina
        }
    }

    // Mostrar solicitudes del usuario
    public function myRequests() { // Método para mostrar solicitudes del usuario
        session_start(); // Inicia sesión
        if (!isset($_SESSION['user_id'])) { // Verifica login
            header("Location: index.php?action=login"); // Redirige si no logueado
            exit(); // Termina
        }
        $requests = $this->requestModel->getUserRequests($_SESSION['user_id']); // Obtiene solicitudes
        include('views/my_requests.php'); // Incluye vista de solicitudes
    }
}
?>
```

### Prueba
- Actualizar `index.php` temporalmente para probar `home`:
  ```php
  // En index.php, cambiar default a:
  case 'home':
  default:
      $controller = new HomeController();
      $controller->index();
      break;
  ```
- Acceder a `http://localhost/tu_proyecto/index.php` y verificar que la página principal cargue (lista vacía de tutores inicialmente).
- Probar login/registro del original para asegurar compatibilidad.

## Paso 6: Crear las Vistas

### views/home.php (Modificado)

Muestra la lista de tutores.

```php
<!DOCTYPE html> <!-- Declara documento HTML5 -->
<html> <!-- Inicio del documento HTML -->
<head> <!-- Sección de encabezado -->
    <title>Plataforma de Tutorías</title> <!-- Título de la página -->
</head> <!-- Fin de encabezado -->
<body> <!-- Cuerpo de la página -->
    <h2>Lista de Tutores Disponibles</h2> <!-- Encabezado principal -->
    <?php if (isset($_SESSION['user_id'])): ?> <!-- Si usuario logueado -->
        <p>Bienvenido, <?php echo $_SESSION['nombre']; ?>! <a href="index.php?action=register_tutor">Registrarse como Tutor</a> | <a href="index.php?action=my_requests">Mis Solicitudes</a> | <a href="index.php?action=logout">Cerrar Sesión</a></p> <!-- Mensaje de bienvenida con enlaces -->
    <?php else: ?> <!-- Si no logueado -->
        <p><a href="index.php?action=login">Iniciar Sesión</a> | <a href="index.php?action=register">Registrarse</a></p> <!-- Enlaces para login y registro -->
    <?php endif; ?> <!-- Fin del condicional -->

    <ul> <!-- Lista no ordenada -->
        <?php foreach ($tutors as $tutor): ?> <!-- Bucle para cada tutor -->
            <li> <!-- Elemento de lista -->
                <strong><?php echo $tutor['nombre']; ?></strong><br> <!-- Nombre del tutor en negrita -->
                Descripción: <?php echo $tutor['descripcion']; ?><br> <!-- Descripción del tutor -->
                Asignaturas: <?php echo $tutor['asignaturas']; ?><br> <!-- Asignaturas del tutor -->
                <?php if (isset($_SESSION['user_id'])): ?> <!-- Si logueado, mostrar enlace -->
                    <a href="index.php?action=request_tutoria&tutor_id=<?php echo $tutor['id']; ?>">Solicitar Tutoría</a> <!-- Enlace para solicitar tutoría -->
                <?php endif; ?> <!-- Fin condicional -->
            </li> <!-- Fin elemento de lista -->
        <?php endforeach; ?> <!-- Fin bucle -->
    </ul> <!-- Fin lista -->
</body> <!-- Fin cuerpo -->
</html> <!-- Fin documento HTML -->
```

### views/register_tutor.php (Nuevo)

Formulario para registrarse como tutor.

```php
<!DOCTYPE html> <!-- Declara documento HTML5 -->
<html> <!-- Inicio del documento HTML -->
<head> <!-- Sección de encabezado -->
    <title>Registrarse como Tutor</title> <!-- Título de la página -->
</head> <!-- Fin de encabezado -->
<body> <!-- Cuerpo de la página -->
    <h2>Registrarse como Tutor</h2> <!-- Encabezado principal -->
    <form method="post" action="index.php?action=register_tutor_process"> <!-- Formulario POST para procesar registro -->
        <label for="descripcion">Descripción:</label><br> <!-- Etiqueta para descripción -->
        <textarea name="descripcion" required></textarea><br> <!-- Campo de texto para descripción -->

        <label>Asignaturas:</label><br> <!-- Etiqueta para asignaturas -->
        <?php foreach ($asignaturas as $asignatura): ?> <!-- Bucle para cada asignatura -->
            <input type="checkbox" name="asignaturas[]" value="<?php echo $asignatura['id']; ?>"> <?php echo $asignatura['nombre']; ?><br> <!-- Checkbox para seleccionar asignatura -->
        <?php endforeach; ?> <!-- Fin bucle -->

        <input type="submit" value="Registrarse como Tutor"> <!-- Botón de envío -->
    </form> <!-- Fin formulario -->
    <a href="index.php?action=home">Volver</a> <!-- Enlace para volver -->
</body> <!-- Fin cuerpo -->
</html> <!-- Fin documento HTML -->
```

### views/request_tutoria.php (Nuevo)

Formulario para solicitar tutoría.

```php
<!DOCTYPE html> <!-- Declara documento HTML5 -->
<html> <!-- Inicio del documento HTML -->
<head> <!-- Sección de encabezado -->
    <title>Solicitar Tutoría</title> <!-- Título de la página -->
</head> <!-- Fin de encabezado -->
<body> <!-- Cuerpo de la página -->
    <h2>Solicitar Tutoría</h2> <!-- Encabezado principal -->
    <form method="post" action="index.php?action=request_tutoria_process"> <!-- Formulario POST para procesar solicitud -->
        <input type="hidden" name="tutor_id" value="<?php echo $_GET['tutor_id']; ?>"> <!-- Campo oculto con ID del tutor -->

        <label for="asignatura_id">Asignatura:</label> <!-- Etiqueta para asignatura -->
        <select name="asignatura_id" required> <!-- Select para elegir asignatura -->
            <?php foreach ($asignaturas as $asignatura): ?> <!-- Bucle para cada asignatura -->
                <option value="<?php echo $asignatura['id']; ?>"><?php echo $asignatura['nombre']; ?></option> <!-- Opción de asignatura -->
            <?php endforeach; ?> <!-- Fin bucle -->
        </select><br> <!-- Salto de línea -->

        <input type="submit" value="Solicitar"> <!-- Botón de envío -->
    </form> <!-- Fin formulario -->
    <a href="index.php?action=home">Volver</a> <!-- Enlace para volver -->
</body> <!-- Fin cuerpo -->
</html> <!-- Fin documento HTML -->
```

### views/my_requests.php (Nuevo)

Muestra las solicitudes del usuario.

```php
<!DOCTYPE html> <!-- Declara documento HTML5 -->
<html> <!-- Inicio del documento HTML -->
<head> <!-- Sección de encabezado -->
    <title>Mis Solicitudes</title> <!-- Título de la página -->
</head> <!-- Fin de encabezado -->
<body> <!-- Cuerpo de la página -->
    <h2>Mis Solicitudes de Tutoría</h2> <!-- Encabezado principal -->
    <ul> <!-- Lista no ordenada -->
        <?php foreach ($requests as $request): ?> <!-- Bucle para cada solicitud -->
            <li> <!-- Elemento de lista -->
                Tutor: <?php echo $request['tutor']; ?><br> <!-- Nombre del tutor -->
                Asignatura: <?php echo $request['asignatura']; ?><br> <!-- Nombre de la asignatura -->
                Estado: <?php echo $request['estado']; ?><br> <!-- Estado de la solicitud -->
                Fecha: <?php echo $request['fecha_solicitud']; ?> <!-- Fecha de la solicitud -->
            </li> <!-- Fin elemento de lista -->
        <?php endforeach; ?> <!-- Fin bucle -->
    </ul> <!-- Fin lista -->
    <a href="index.php?action=home">Volver</a> <!-- Enlace para volver -->
</body> <!-- Fin cuerpo -->
</html> <!-- Fin documento HTML -->
```

### Prueba
- Crear archivos de vistas temporales con contenido básico para probar renderizado.
- Acceder a `http://localhost/tu_proyecto/index.php?action=register_tutor` (después de login) y verificar formulario.
- Acceder a `http://localhost/tu_proyecto/index.php?action=request_tutoria&tutor_id=1` y verificar formulario de solicitud.
- Si renderiza correctamente, proceder.

## Paso 7: Actualizar el Router (index.php)

Actualiza el archivo index.php para incluir las nuevas acciones.

```php
<?php
require_once('controllers/AuthController.php');
require_once('controllers/HomeController.php');
require_once('controllers/TutorController.php');

$action = isset($_GET['action']) ? $_GET['action'] : 'home';

switch ($action) {
    case 'login':
        $controller = new AuthController();
        $controller->login();
        break;
    case 'register':
        $controller = new AuthController();
        $controller->register();
        break;
    case 'logout':
        $controller = new AuthController();
        $controller->logout();
        break;
    case 'register_tutor':
        $controller = new TutorController();
        $controller->registerForm();
        break;
    case 'register_tutor_process':
        $controller = new TutorController();
        $controller->register();
        break;
    case 'request_tutoria':
        $controller = new TutorController();
        $controller->requestForm($_GET['tutor_id']);
        break;
    case 'request_tutoria_process':
        $controller = new TutorController();
        $controller->request();
        break;
    case 'my_requests':
        $controller = new TutorController();
        $controller->myRequests();
        break;
    case 'home':
    default:
        $controller = new HomeController();
        $controller->index();
        break;
}
?>
```

### Prueba
- Actualizar `index.php` con las nuevas rutas.
- Acceder a `http://localhost/tu_proyecto/index.php?action=home` y verificar lista de tutores.
- Iniciar sesión y probar `action=register_tutor`, `action=request_tutoria&tutor_id=1`, `action=my_requests`.
- Verificar redirecciones y que no haya errores de routing.
- Si funciona, proceder a pruebas finales.

## Paso 8: Probar la Aplicación

1. Coloca todos los archivos en la carpeta raíz de tu servidor web.
2. Accede a `http://localhost/tu_proyecto/tu_proyecto/index.php`.
3. Registra usuarios.
4. Inicia sesión y registra algunos como tutores.
5. Ve la lista de tutores y solicita tutorías.
6. Verifica las solicitudes en "Mis Solicitudes".

Esta extensión mantiene la simplicidad del original mientras agrega funcionalidad para el sistema de tutorías.

## Estructura del Proyecto MVC

Esta aplicación sigue el patrón de diseño Modelo-Vista-Controlador (MVC), que separa la lógica de negocio, la presentación y el control de la aplicación en capas distintas para mejorar la mantenibilidad y escalabilidad.

### Carpetas y Archivos Principales

- **models/**: Contiene las clases que representan los datos y la lógica de negocio. Cada modelo interactúa con la base de datos para realizar operaciones CRUD (Crear, Leer, Actualizar, Eliminar).
  - `UserModel.php`: Maneja operaciones relacionadas con usuarios (autenticación, registro).
  - `TutorModel.php`: Gestiona datos de tutores y asignaturas.
  - `RequestModel.php`: Maneja solicitudes de tutorías.

- **views/**: Contiene los archivos de presentación (vistas) que muestran la interfaz de usuario. Estas vistas son archivos HTML con código PHP embebido para mostrar datos dinámicos.
  - `home.php`: Página principal que lista tutores disponibles.
  - `register_tutor.php`: Formulario para registrarse como tutor.
  - `request_tutoria.php`: Formulario para solicitar tutoría.
  - `my_requests.php`: Página que muestra las solicitudes del usuario.
  - (Otros como login.php, register.php del sistema original)

- **controllers/**: Contiene las clases controladoras que manejan la lógica de aplicación, procesan las solicitudes del usuario y coordinan entre modelos y vistas.
  - `AuthController.php`: Maneja autenticación (login, registro, logout).
  - `HomeController.php`: Controla la página principal.
  - `TutorController.php`: Gestiona registro de tutores y solicitudes de tutorías.

- **index.php**: Archivo principal que actúa como router o controlador frontal. Recibe todas las solicitudes y las dirige al controlador apropiado basado en el parámetro `action`.

- **config.php**: Archivo de configuración que establece la conexión a la base de datos.

- **README.md**: Guía original para el sistema de login básico.

- **README2.md**: Esta guía de extensión para el sistema de tutorías.

### Flujo MVC en la Aplicación

1. El usuario accede a una URL como `index.php?action=home`.
2. `index.php` determina el controlador apropiado (HomeController) y llama a su método correspondiente (index).
3. El controlador interactúa con el modelo (TutorModel) para obtener datos de la base de datos.
4. El controlador incluye la vista correspondiente (home.php), pasando los datos necesarios.
5. La vista renderiza el HTML final con los datos dinámicos y lo envía al navegador del usuario.

