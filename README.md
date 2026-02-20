# BPArchitecture
Diseño de un sistema de Banca Online para la entidad BP

1. JUSTIFICACIONES DE DECISIONES ARQUITECTONICAS
    1.1. Front-End: SPA y Aplicación Móvil
        1.1.1. SPA: Se optó por REACT sobre ANGULAR, Vue.js, Svelte entre otros por:
            a. VIRTUAL DOM y Rendimiento: Permite actualizaciones rápidas de la interfaz sin recargar la página, crítico para la fluidez en consultas de movimientos bancarios.
            b. Ecosistema y Seguridad: Amplio soporte para bibliotecas de seguridad (como OIDC-client) que facilitan la integración con el servidor de autenticación OAuth2.
            c. Librería madura y con amplio soporte de comunidades.
        1.1.2. FLUTTER: Se optó por FLUTTER sobre React Native, Kotlin multiplataforma, Ionic entre otros por:
            a. Rendimiento Nativo: A diferencia de soluciones basadas en WebView, Flutter compila a código nativo, lo que garantiza una respuesta táctil fluida y acceso eficiente a biometría (FaceID/Huella).
            b. Consistencia de UI: El motor de renderizado propio asegura que la experiencia de marca de BP sea idéntica en Android e IOS con un solo codebase
            c. Framework maduro y con amplio soporte de comunidades.
    1.2. Autenticación, Onboarding, Lambda Authorizer:
        1.2.1. Flujo OAuth2.0
            a. Authorization Code Flow con PKCE (Proof Key for Code Exchange)
            b. Dado que las SPA y Apps Móviles no pueden guardar un client_secret de forma segura, PKCE mitiga la interceptación del código de autorización
            c. Es la recomendación actual de la IETF para aplicaciones móviles y SPA (RFC 7636)
        1.2.2. Biometría
            a. Para el ingreso recurrente, se usará la API nativa de Biometrics (LocalAuthentication) vinculando el Secure Enclave del dispositivo con un Refresh Token de larga duración. 
            b. Una vez validado el usuari localmente, la app móvil solicita los tokens a Cognito (via PKCE flow)
        1.2.3. Lambda Authorizer
            a. Se llama desde API GATEWAY para validar la firma del JWT emitido por COGNITO.
            b. Genera política de IAM dinámica para permitir el acceso al microservicio específico        
    1.3. Seguridad en el Perímetro
        1.3.1. WAF: Debe filtrar ataques comunes (SQLi, XSS, etc) y tener reglas de Rate Limiting para evitar ataques de fuerza bruta en los endpoints de transferencia
        1.3.2. CloudFront: Utiliza Origin Access Control (OAC) para asegurar que nadie pueda saltarse el CDN y atacar directamente al API Gateway o al bucket de la SPA
        1.3.3. Agentes de seguridad para monitoreo: Algunos SW especializados para monitoreo de seguridad en tiempo real en el Docker, como Datadog/Dynatrace/CrowdStrike, a través de un agente ultraligero instalado en el contenedor, permiten saber procesos, comportamientos, request/response a los microservicios, buscar firmas de virus, bloquear ataques, etc. FluidAttack permite un monitoreo de seguridad como análisis de dependencias, a nivel de IC/DC. Si un desarrollador sube código con una vulnerabilidad crítica, FluidAttacks bloquea el despliegue (rompe el pipeline) hasta que el fallo sea remediado, asegurando que el SW vulnerable no llegue a producción.    
    1.4. Integración y Persistencia
        a. Se usa Patrón API Gateway, que permite tener una puerta centralizada de entrada y control, perimitiendo que los microservicios se especialicen en la lógica de negocio y su funcionalidad más no en otros procesos como la validación de acceso.
        b. Patrón Event-Drive con Publisher/Subscriber: Permite desacoplar procesos que no son críticos en la fluidez de la experiencia de usuario en el sistema y que pueden ser encaradas de manera asíncrona.
        c. DynamoDB: Permite almacenar los registros que nos den trazabillidad de lo que ha hecho un usuario en una sesión, guardando esta iformación de acuerdo a la normativa del país y de la entidad bancaria entre 6 a 13 meses (se debe especificar una política de backups durante este periodo).
        d. Core, OAuth2 y Sistema Independiente: Generalmente los Bancos mantienen control de la data de sus clientes en sus servidores locales, por tanto se opta por una Arquitectura Híbrida, mientras modernizan sus Core.
    1.5. Procesamiento Asíncrono (Auditoría, Notificaciones e Información Bancaria)
        a. EventBridge: Recomendado para una Arquitectura Bancaria que permita fluidez en la experiencia de usuario, sin que que los procesos se detengan mientras los procesos de Auditoría, Notificaciones e Información Bancaria terminan.
        b. Existe desacoplamiento de los microservicios con los procesos de  Auditoría, Notificaciones e Información Bancaria, ya que sólo envían los eventos con la información necesaria, que posteriormente será procesada de manera asíncrona.
        c. La Auditoría debe ser almacenada de forma simple y generalmente son los request y response de los microservicios, por lo cual se opta por una BDD NoSQL (DynamoDB) que permita escalabilidad horizontal en caso de requerirlo y registros de unos cuantos Kb
    1.6. Arquitectura Microservicios y aplicaciones FrontEnd
        Se opta por una Arquitectura Limpia, ya que nos permite separar responsabilidades, aislando la lógica del negocio, de la conectividad con la infraestructura a través de un intermediario en la capa de Aplicación (casos de uso). Aprovechamos este desacoplamiento entre Infraestructura, Aplicación y Dominio para tener una alta testiabilidad sin dependencias externar. Si las tecnologías de la periferia cambian nos permite cambiar la conectividad sin afectar la lógica de negocio.
        Una Arquitectura Limpia hace que un desarrollador nuevo pueda mirar la estructura de carpetas e identificar inmediatamente de que trata el negocio, sin perderse en configuraciones de red o dependencias de SW, haciendo más mantenible a largo plazo.
 

2. ANALISIS TECNICO DETALLADO DEL DIAGRAMA DE CONTENEDORES (C4 NIVEL 2) PARA EL SISTEMA DE BANCA POR INTERNET DE BP
    La arquitectura sigue un enfoque moderno de microservicios sobre AWS EKS, con una fuerte separación entre procesos síncronos y asíncronos (Event-Driven).
    1.⁠ ⁠Capa de Acceso y Seguridad Perimetral
        Esta capa asegura que solo el tráfico legítimo llegue a la infraestructura.
        * AWS WAF (Web Application Firewall): Se justifica para proteger la aplicación contra exploits comunes (SQL Injection, XSS) y ataques de denegación de servicio (DDoS) antes de que lleguen a la lógica de negocio.
        * Amazon CloudFront: Actúa como Content Delivery Network (CDN). Su función es doble:
            -   Distribución de contenido estático: Sirve el build de la SPA (React) alojada en Amazon S3 con baja latencia.
            -   Ruteo dinámico: Basado en el path (/api/*), redirige las peticiones al balanceador del cluster EKS y a Cógnito durante el proceso de OAuth2 (/auth/*)
    2.⁠ ⁠Gestión de Identidad y Autenticación (OAuth2)
        El diagrama muestra un flujo de autenticación robusto y desacoplado.
        * AWS Cognito & Sistema de Identidad: Cognito actúa como el Identity Pool que se federa con un Sistema de Identidad externo (OAuth2). Esto permite centralizar la gestión de usuarios mientras se delega la validación de credenciales a un sistema legacy o corporativo.
        * Lambda Authorizer: Esta es una pieza clave de seguridad. Antes de que el MS-ApiGateway procese una petición, la Lambda valida el JWT (JSON Web Token).
            Justificación: Al validar el token en el borde (Edge/Gateway), se evita que tráfico no autorizado consuma recursos computacionales de los microservicios internos.
    3.⁠ ⁠Orquestación y Microservicios Síncronos (EKS)
        El núcleo de la lógica reside en un cluster de Amazon EKS (Kubernetes).
        * MS-ApiGateway: Actúa como el único punto de entrada para los servicios internos, realizando el ruteo de peticiones y la agregación de servicios si fuera necesario.
        * Microservicios de Negocio (Datos Básicos, Movimientos, Transferencias):
        * Se comunican de forma síncrona con el Core Bancario y Sistemas Independientes.
            Justificación: Las consultas de saldos o transferencias requieren una respuesta inmediata (Request-Response) para confirmar al usuario que la operación fue recibida o para mostrar datos en tiempo real.
    4.⁠ ⁠Arquitectura Orientada a Eventos (Event-Driven)
        Aquí es donde el sistema maneja procesos pesados o secundarios sin bloquear la experiencia del usuario.
        * Amazon EventBridge: Funciona como el Bus de Eventos central. Cuando un microservicio (ej. Transferencias) termina una acción, publica un evento.
            Justificación: Desacopla totalmente al productor del consumidor. El servicio de transferencias no necesita saber quién más necesita esa información.
        * SQS (Simple Queue Service): Actúa como buffer entre EventBridge y los consumidores.
            Justificación: Garantiza la resiliencia y persistencia. Si un microservicio consumidor (como MS-Auditoria) falla o está bajo mucha carga, los mensajes permanecen en la cola hasta ser procesados (retry logic), evitando la pérdida de datos críticos.
        Componentes de Procesamiento Asíncrono:
        * MS-InfoBancaria: Envía datos a la Entidad Regulatoria. Al ser asíncrono, si el sistema del regulador está lento o fuera de línea, no afecta la transacción del cliente.
        * MS-Auditoria & DynamoDB:
            Justificación de DynamoDB: Se elige una base de datos NoSQL por su capacidad de escalar horizontalmente y manejar grandes volúmenes de escritura con latencia de milisegundos. Los logs de auditoría son ideales para documentos JSON que pueden variar ligeramente en estructura.
        * MS-Notificaciones: Se conecta con proveedores externos (Twilio para SMS o SendGrid para Email). Al ser un proceso fuera de la "ruta crítica", el usuario recibe su confirmación de transferencia en la App mientras la notificación se procesa en segundo plano.
    5.⁠ ⁠Integraciones Externas
        * Core Bancario / Sistema Independiente: Representan los sistemas de registro (Systems of Record) on-premise o legados que mantienen la integridad financiera.
        * Sistema Monitor de Seguridad: Recibe data del Gateway para observabilidad y detección de anomalías en tiempo real.
        Resumen de Beneficios Técnicos
        * Escalabilidad: El uso de EKS y componentes Serverless (Lambda, EventBridge, DynamoDB) permite crecer según la demanda.
        * Resiliencia: El desacoplamiento mediante SQS asegura que las caídas de sistemas externos (Notificaciones, Reguladores) no impacten la disponibilidad del sistema principal.
        * Seguridad: Validación de identidad multicapa (WAF -> CloudFront -> Lambda Authorizer).

3. LINK DIAGRAMA ICEPANEL.IO (Para entrar en los diferentes niveles del modelo C4 dar clic en la lupa con un número de la esquina superior izquierda de las representaciones)
        https://s.icepanel.io/6rZ4SZLCzsjejf/b7JB
