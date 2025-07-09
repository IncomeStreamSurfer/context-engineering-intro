
## 1. Introducción y Visión General

Como desarrollador fullstack especializado en SaaS para clubes de pádel, he diseñado este documento técnico completo para el **Padel Manager Club**, un plugin de WordPress que funciona como un **Producto Mínimo Viable (MVP)** para la gestión eficiente de clubes de pádel[1].

El plugin está diseñado para **replicar la funcionalidad esencial de plataformas como Playtomic Manager**, adaptada a la flexibilidad y facilidad de uso de WordPress, con un enfoque en la escalabilidad para miles de usuarios potenciales[1].

### Objetivos Principales
- Proporcionar una herramienta intuitiva para supervisar la actividad diaria del club
- Gestionar partidos y visualizar información clave del negocio
- Ofrecer una interfaz limpia y minimalista enfocada en la claridad y facilidad de uso[1]

## 2. Arquitectura del Sistema

### 2.1 Flujo Principal del Sistema

El sistema se centra en la **gestión de entidades de un club de pádel** (partidos, jugadores, pistas) a través de **Custom Post Types (CPT)** de WordPress[2]:

1. **Administrador**: Configura el plugin, gestiona pistas, jugadores y supervisa los partidos desde el panel de administración
2. **Usuario Frontend**: Interactúa con los shortcodes para ver, filtrar y crear partidos
3. **Shortcodes**: Renderizan la interfaz de usuario en el frontend, obteniendo datos a través de WP_Query
4. **API AJAX**: Las acciones del usuario se manejan mediante llamadas AJAX a funciones específicas del plugin
5. **Base de Datos**: WordPress almacena los datos en las tablas `wp_posts` y `wp_postmeta`[2]

### 2.2 Componentes Técnicos

#### Backend
- Se gestiona a través de clases que se enganchan a los hooks de WordPress
- La clase principal `PadelManagerClub` inicializa todos los componentes
- Clases específicas como `PadelManagerCustomPostTypes` o `PadelManagerAdminDashboard` manejan áreas concretas[2]

#### Frontend
- La interacción del usuario se realiza principalmente a través de shortcodes (`[padel_matches]`, `[padel_calendar]`, etc.)
- Cada shortcode tiene una clase asociada que se encarga de renderizar el HTML y gestionar la lógica de negocio[2]

#### API
- Utiliza el sistema AJAX de WordPress (`admin-ajax.php`) para la comunicación entre frontend y backend
- Seguridad implementada mediante nonces de WordPress[2]

## 3. Estructura de Menús del Plugin

### 3.1 Menú Principal en WordPress Admin

El plugin implementa un **menú principal** en el panel de administración de WordPress con las siguientes páginas[3]:

#### 1. Dashboard
- **KPIs en tiempo real**: Visualización de métricas clave del negocio
  - Ingresos del día/semana
  - Número de partidos jugados hoy
  - Ocupación de pistas (porcentaje)
  - Número de jugadores registrados
  - Próximos partidos programados
- **Resumen visual**: Gráficos y estadísticas de ocupación
- **Accesos rápidos**: Enlaces directos a las funciones más utilizadas[3]

#### 2. Partidos
- **Vista de partidos del día**: Lista detallada con hora, pista, jugadores y estado
- **Navegación temporal**: Botones para avanzar/retroceder días
- **Calendario visual**: Vista semanal/mensual con bloques de 1.5 horas
- **Indicadores de estado**:
  - Verde: Partido lleno (4 jugadores)
  - Amarillo: Falta un jugador (3 jugadores)
  - Gris/Blanco: Pista disponible (0-2 jugadores)[3]

#### 3. Jugadores
- **Listado completo**: Tabla paginada con capacidad de búsqueda por nombre/apellido
- **Perfiles de jugador**: Información detallada incluyendo:
  - Nombre completo
  - Información de contacto (email, teléfono)
  - Nivel de pádel (campo editable por el administrador)
  - Historial de partidos jugados
- **Conexión con frontend**: Integración directa con el shortcode `[padel_matches]`[3]

#### 4. Configuración
- **Información del club**: Nombre, logo, información de contacto
- **Configuración de pistas**: Número de pistas, nombres, estado activo/inactivo
- **Horarios operativos**: Horario de apertura/cierre, configuración de bloques horarios
- **Precios**: Tarifas estándar por partido/hora[3]

### 3.2 Implementación Técnica de Menús

```php
public function add_admin_menu() {
    add_menu_page(
        'Padel Manager',
        'Padel Club',
        'manage_options',
        'padel-manager-dashboard',
        array($this, 'dashboard_page'),
        'dashicons-location-alt',
        30
    );
    
    add_submenu_page(
        'padel-manager-dashboard',
        'Dashboard',
        'Dashboard',
        'manage_options',
        'padel-manager-dashboard'
    );
    
    add_submenu_page(
        'padel-manager-dashboard',
        'Partidos',
        'Partidos',
        'manage_options',
        'padel-manager-matches',
        array($this, 'matches_page')
    );
    // ... resto de submenús
}
```

## 4. Arquitectura de Base de Datos

### 4.1 Modelo de Datos Recomendado

Para un **SaaS de gestión de clubes de pádel** donde los jugadores pueden participar en múltiples clubes, la opción más eficiente y escalable es **crear una tabla independiente** en lugar de usar exclusivamente los usuarios de WordPress[4].

#### Ventajas de Tabla Independiente
1. **Aislamiento de datos**: Cada club mantiene su información específica sin interferir con otros
2. **Escalabilidad**: Mejor rendimiento que `wp_usermeta` para grandes volúmenes de datos
3. **Flexibilidad**: Permite campos específicos del dominio de pádel
4. **Integridad referencial**: Mejor control de relaciones entre entidades[4]

### 4.2 Estructura de Tablas Propuesta

#### Tabla Principal de Jugadores Globales
```sql
CREATE TABLE padel_players (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    wp_user_id BIGINT UNSIGNED NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    birth_date DATE,
    global_level ENUM('beginner', 'intermediate', 'advanced', 'professional'),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    FOREIGN KEY (wp_user_id) REFERENCES wp_users(ID) ON DELETE SET NULL,
    INDEX idx_email (email),
    INDEX idx_wp_user (wp_user_id)
);
```

#### Tabla de Relación Jugador-Club
```sql
CREATE TABLE padel_player_clubs (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    player_id BIGINT UNSIGNED NOT NULL,
    club_id BIGINT UNSIGNED NOT NULL,
    site_id BIGINT UNSIGNED NOT NULL, -- WordPress site ID
    local_level ENUM('beginner', 'intermediate', 'advanced', 'professional'),
    membership_type ENUM('member', 'guest', 'trial'),
    joined_date DATE NOT NULL,
    left_date DATE NULL,
    is_active BOOLEAN DEFAULT TRUE,
    total_matches INT DEFAULT 0,
    wins INT DEFAULT 0,
    losses INT DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    FOREIGN KEY (player_id) REFERENCES padel_players(id) ON DELETE CASCADE,
    UNIQUE KEY unique_player_club (player_id, club_id, site_id),
    INDEX idx_club_site (club_id, site_id),
    INDEX idx_player_active (player_id, is_active)
);
```

### 4.3 Custom Post Types Actuales

El sistema actual utiliza **Custom Post Types** para las entidades principales[2]:

#### CPT `padel_match`
- **post_title**: Nombre del partido
- **post_content**: Descripción del partido
- **Meta-fields**:
  - `match_date`: Fecha del partido (YYYY-MM-DD)
  - `match_time`: Hora del partido (HH:MM)
  - `court_id`: ID del post de la pista
  - `players`: Array de IDs de los usuarios/jugadores inscritos
  - `max_players`: Número máximo de jugadores
  - `level`: Nivel del partido
  - `price`: Precio del partido[2]

#### CPT `padel_player`
- **post_title**: Nombre del jugador
- **Meta-fields**:
  - `user_id`: ID del usuario de WordPress asociado
  - `level`: Nivel de juego del jugador
  - `phone`: Teléfono de contacto[2]

## 5. Funcionalidades Core del Plugin

### 5.1 Dashboard con KPIs Principales

Proporciona una **vista rápida y concisa** de las métricas de negocio más relevantes para un club de pádel[1]:

- Ingresos del día/semana
- Número de partidos jugados hoy
- Ocupación de pistas (porcentaje)
- Número de jugadores registrados
- Próximos partidos programados

### 5.2 Gestión de Partidos

#### Visualización de Partidos
- Lista de partidos con hora, pista, jugadores confirmados y estado de ocupación
- Navegación temporal con botones o selector de fecha para avanzar/retroceder días
- Detalle de partido al hacer clic mostrando información completa[1]

#### Schedule con Calendario
- **Vista de calendario**: Calendario diario/semanal que muestra bloques de 1.5 horas por partido
- **Indicadores visuales**:
  - Verde: Partido lleno (4 jugadores)
  - Amarillo: Falta un jugador (3 jugadores)
  - Gris/Blanco: Pista disponible (0-2 jugadores o no programado)[1]

### 5.3 Shortcode para Frontend

El shortcode `[padel_matches]` permite a los usuarios **crear nuevos partidos** directamente desde el frontend[1]:

- Mostrar un listado de partidos disponibles para unirse o iniciar uno nuevo
- Identificación clara de los bloques horarios disponibles en las pistas
- Formulario sencillo para definir los detalles básicos del nuevo partido

#### Flujo de Funcionamiento del Shortcode
1. Al cargar una página con el shortcode, el método `render_shortcode()` genera el HTML inicial
2. El método `get_matches_html()` realiza una WP_Query para obtener los partidos y mostrarlos
3. El archivo JS se encarga de las interacciones: abrir el modal, enviar el formulario de creación y actualizar la lista mediante AJAX[2]

## 6. Guía de Estilos y UI/UX

### 6.1 Sistema de Colores

El plugin utiliza un **sistema de colores consistente** con prefijo `padel-club-` para evitar conflictos[5]:

```css
:root {
    --padel-club-primary: #4F46E5;
    --padel-club-success: #10B981;
    --padel-club-warning: #F59E0B;
    --padel-club-danger: #EF4444;
    --padel-club-info: #3B82F6;
    --padel-club-light: #F8FAFC;
    --padel-club-dark: #1E293B;
}
```

#### Estados de Reservas
```css
.padel-club-available {
    color: #10B981;
    background-color: #DCFCE7;
}

.padel-club-reserved {
    color: #F59E0B;
    background-color: #FEF3C7;
}

.padel-club-completed {
    color: #8B5CF6;
    background-color: #EDE9FE;
}

.padel-club-cancelled {
    color: #EF4444;
    background-color: #FEE2E2;
}
```

### 6.2 Componentes de UI

#### Botones Compatibles con WordPress
```css
.padel-club-btn {
    display: inline-block;
    padding: 8px 16px;
    border-radius: 4px;
    text-decoration: none;
    font-weight: 500;
    transition: all 0.3s ease;
    cursor: pointer;
    border: none;
    font-size: 14px;
}

.padel-club-btn-primary {
    background-color: var(--padel-club-primary);
    color: white;
}
```

#### Calendario de Reservas
```css
.padel-club-calendar {
    background: white;
    border: 1px solid #E2E8F0;
    border-radius: 8px;
    overflow: hidden;
}

.padel-club-calendar-day.available {
    background-color: #DCFCE7;
    color: #166534;
}

.padel-club-calendar-day.reserved {
    background-color: #FEF3C7;
    color: #92400E;
}
```

### 6.3 Responsive Design

El plugin implementa un **diseño responsive** con breakpoints optimizados[5]:

```css
/* Mobile First */
.padel-club-container {
    width: 100%;
    padding: 0 15px;
}

@media (min-width: 576px) {
    .padel-club-container {
        max-width: 540px;
        margin: 0 auto;
    }
}

@media (min-width: 768px) {
    .padel-club-container {
        max-width: 720px;
    }
    
    .padel-club-grid {
        grid-template-columns: repeat(2, 1fr);
    }
}
```

## 7. Integración con WordPress

### 7.1 Enfoque Híbrido Recomendado

La **solución óptima combina ambos enfoques**[4]:

1. **Usuarios WordPress**: Para autenticación y funcionalidades básicas del sistema
2. **Tabla independiente**: Para datos específicos de pádel y relaciones multi-club

### 7.2 Implementación de Seguridad

```php
class PadelSecurity {
    public function can_access_player_data($user_id, $player_id) {
        // Verificar si el usuario tiene acceso al jugador en su club
        global $wpdb;
        
        $club_id = $this->get_user_club_id($user_id);
        $site_id = get_current_blog_id();
        
        $has_access = $wpdb->get_var($wpdb->prepare(
            "SELECT COUNT(*) FROM padel_player_clubs 
             WHERE player_id = %d AND club_id = %d AND site_id = %d",
            $player_id, $club_id, $site_id
        ));
        
        return $has_access > 0;
    }
}
```

## 8. Optimización de Rendimiento

### 8.1 Índices Optimizados

```sql
-- Índices optimizados para consultas frecuentes
CREATE INDEX idx_player_email_active ON padel_players(email, created_at);
CREATE INDEX idx_club_players_active ON padel_player_clubs(club_id, site_id, is_active);
CREATE INDEX idx_player_matches ON padel_player_clubs(player_id, total_matches DESC);
```

### 8.2 Vista Optimizada para Dashboard

```sql
-- Vista optimizada para dashboard
CREATE VIEW padel_club_stats AS
SELECT 
    c.id as club_id,
    c.name as club_name,
    c.site_id,
    COUNT(pc.player_id) as total_players,
    SUM(pc.total_matches) as total_matches,
    AVG(pc.total_matches) as avg_matches_per_player
FROM padel_clubs c
LEFT JOIN padel_player_clubs pc ON c.id = pc.club_id AND pc.is_active = 1
GROUP BY c.id, c.name, c.site_id;
```

## 9. Estructura de Archivos del Plugin

```
padel-manager-club/
├── admin/
│   ├── class-admin-dashboard.php
│   ├── class-admin-matches.php
│   ├── class-admin-players.php
│   └── class-admin-settings.php
├── assets/
│   ├── css/
│   │   ├── admin-styles.css
│   │   ├── frontend-styles.css
│   │   ├── components.css
│   │   └── responsive.css
│   └── js/
│       ├── admin-scripts.js
│       └── frontend-scripts.js
├── includes/
│   ├── class-database-setup.php
│   ├── class-custom-post-types.php
│   ├── class-padel-matches-shortcode.php
│   └── class-padel-manager-club.php
├── templates/
│   ├── dashboard.php
│   ├── matches.php
│   ├── players.php
│   └── settings.php
├── languages/
│   └── padel-manager-club.pot
├── padel-manager-club.php
└── readme.txt
```

## 10. Roadmap de Desarrollo

### Phase 1: Foundation (MVP)
- Configuración del proyecto y estructura base del plugin
- Dashboard básico con KPIs principales
- Gestión de pistas con CRUD básico
- CPT para Partidos y Jugadores
- Entorno de desarrollo con Docker + WordPress[1]

### Phase 2: Core Features
- Implementación completa del sistema de partidos
- Shortcode para frontend con funcionalidad completa
- Sistema de gestión de jugadores
- Integración con sistema de usuarios de WordPress

### Phase 3: Advanced Features
- Optimización de rendimiento con tablas personalizadas
- Sistema de notificaciones
- Integración con pasarelas de pago
- API REST completa para integraciones externas

## 11. Consideraciones de Escalabilidad

### 11.1 Arquitectura Multi-Tenant

Para manejar **múltiples clubes** en una instalación, el sistema implementa:

- **Base de datos compartida**: Usar las mismas tablas personalizadas en todas las instalaciones
- **API centralizada**: Servicios REST para sincronización de datos
- **Eventos de sincronización**: Hooks para actualizar datos en tiempo real[4]

### 11.2 Beneficios de la Arquitectura Propuesta

1. **Escalabilidad**: El sistema puede manejar cientos de miles de jugadores y múltiples clubes
2. **Flexibilidad**: Fácil adaptación a diferentes modelos de negocio
3. **Rendimiento**: Consultas optimizadas y menos carga en `wp_usermeta`
4. **Integridad**: Mejor control de datos y relaciones consistentes
5. **Portabilidad**: Independiente de la estructura interna de WordPress[4]

Esta documentación proporciona la **base sólida** para un SaaS de gestión de clubes de pádel que puede escalar eficientemente manteniendo la integridad de los datos y la flexibilidad necesaria para diferentes modelos de negocio.