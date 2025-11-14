Quiero que me ayudes a **generar un set de 3 mini proyectos** que abarquen los siguientes temas de **Frontend con React, TypeScript y TailwindCSS**.

El escenario principal es que los participantes **recibirán una API Backend (documentada, ej. con Swagger u OpenAPI)** y deberán **construir el Frontend** para consumirla, aplicando las tecnologías y patrones solicitados.

### Requisitos de los mini proyectos

* **Duración estimada:** deben poder resolverse en máximo **3-4 horas**, de manera **individual**.
* **Nivel:** dificultad **medio/alto** (requiere familiaridad con React, pero no nivel experto).
* **Recursos permitidos:** se asume acceso a **documentación oficial (React, Vite, Shadcn, etc.), inteligencia artificial, y buscadores**.
* **Formato del enunciado:** el enunciado debe ser claro, autocontenido y debe **especificar explícitamente**:
    * **El Stack Tecnológico OBLIGATORIO**: Vite, React, TypeScript, React Router, TailwindCSS, Shadcn UI.
    * **La API a consumir**: Proveer los *endpoints* clave (ruta, verbo HTTP, qué *body* espera, qué *response* devuelve).
    * **Vistas (Pages) requeridas**: Qué pantallas debe implementar el alumno (ej. `/login`, `/dashboard`, `/items/:id`).
    * **Flujos de usuario principales**: "El usuario debe poder loguearse. Si no está logueado, debe ser redirigido a /login. Una vez logueado, debe ver un dashboard con datos paginados".
    * **Validaciones de formulario** requeridas (ej. "el email debe ser válido", "la contraseña debe tener 8 caracteres").
    * **Estructura de carpetas** sugerida (ej. `pages`, `components`, `services`, `hooks`, `context`, `lib/utils`).

---

### Temas clave a cubrir en los enunciados

1.  **Configuración Inicial (Vite + TS)**
    * Creación del proyecto con Vite, plantilla `react-ts`.
    * Configuración de **TailwindCSS** (archivo `tailwind.config.js`).
    * Instalación y configuración de **Shadcn UI** (o la librería de componentes designada).
2.  **Enrutamiento (React Router DOM)**
    * Configuración de `createBrowserRouter`.
    * Implementación de **Rutas Públicas** (ej. Login, Register).
    * Implementación de **Rutas Privadas** (requieren autenticación).
    * Uso de **Rutas Dinámicas** (ej. `/dashboard/products/:productId`).
    * Creación de un `Layout` principal que contenga el `Outlet`.
3.  **UI/UX y Componentes (Shadcn UI)**
    * Uso efectivo de componentes de Shadcn como `Button`, `Input`, `Card`, `Table`, `Dialog`, `Toast`.
    * Creación de un diseño limpio y responsivo.
4.  **Autenticación y Manejo de Estado Global**
    * Creación de un **AuthContext** (o un *store* de Zustand/Redux) para gestionar el estado del usuario y el token JWT.
    * Implementación de formularios de Login y Register.
    * Almacenamiento seguro del token (localStorage) y envío automático en *headers* de peticiones (ej. con interceptores de Axios).
    * Lógica de *Logout*.
5.  **Formularios y Validación**
    * Uso de **React Hook Form** para gestionar el estado de los formularios.
    * Validación de esquemas (datos del formulario) usando **Zod**.
    * Manejo de estados de envío (loading, error, success) en los botones.
6.  **Consumo de API (Data Fetching)**
    * Uso de `fetch` o `axios` para las peticiones. (Opcional avanzado: `react-query / tanstack-query`).
    * **Consumo de endpoints de Autenticación** (Login, Register).
    * **Consumo de endpoints de Creación** (ej. `POST /api/v1/products`) usando un formulario.
    * **Consumo de endpoints con Paginación** (ej. `GET /api/v1/items?page=1&limit=10`) y renderizar los datos en una tabla (`<Table>` de Shadcn).
    * Implementar controles de paginación (botones "Siguiente", "Anterior").
7.  **Manejo de Errores y Notificaciones**
    * Mostrar notificaciones "Toast" al usuario tras acciones (ej. "Registro exitoso", "Error: credenciales inválidas").
    * Manejo de errores de API y mostrarlos en el UI (ej. mensajes de error bajo los campos del formulario).

---

### Reto Adicional (Desacoplado)

Como un desafío opcional, que se puede agregar a cualquiera de los enunciados principales:

* **Integración de Pasarela de Pagos:**
    * Integrar **Stripe.js** o **Mercado Pago (Checkout Pro)**.
    * Añadir un botón "Comprar" o "Suscribirse" en alguna parte de la aplicación.
    * Al hacer clic, el frontend debe llamar a un endpoint (simulado o provisto) del backend para crear una "intención de pago" o "preferencia de pago".
    * Con la respuesta, redirigir al usuario al checkout de la pasarela de pago.

---

👉 **En resumen:** quiero que los enunciados de proyectos estén diseñados para que los alumnos **configuren un proyecto de React moderno desde cero** y **demuestren su habilidad para consumir una API REST compleja**, manejando autenticación, formularios, carga de datos paginados y creando una UI/UX profesional con Tailwind y Shadcn.
