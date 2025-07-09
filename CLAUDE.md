# Especificación Técnica Completa: Plugin Padel Manager Club

## 1. Visión General del Proyecto

### 1.1 Propósito
Sistema de gestión integral para clubes de pádel desarrollado como plugin de WordPress, diseñado para replicar la funcionalidad esencial de plataformas como Playtomic Manager con enfoque en escalabilidad y facilidad de uso.

### 1.2 Objetivos Principales
- **Gestión eficiente de partidos**: Sistema de reservas con 4 jugadores fijos
- **Control de niveles**: Sistema decimal del 1.0 al 7.0 con compatibilidad ±1
- **Dashboard administrativo**: KPIs en tiempo real y gestión completa
- **Frontend público**: Interfaz para usuarios finales con shortcodes
- **Escalabilidad multi-club**: Arquitectura preparada para múltiples instalaciones

### 1.3 Arquitectura General
- **Base**: Plugin WordPress con Custom Post Types
- **Base de datos**: Tablas personalizadas + wp_posts/wp_postmeta
- **Frontend**: Shortcodes con AJAX para interactividad
- **Backend**: Panel administrativo integrado en WordPress
- **Escalabilidad**: Modelo multi-tenant con aislamiento de datos

## 2. Modelo de Datos y Relaciones

### 2.1 Entidades Principales

#### Jugadores (padel_players)
**Propósito**: Almacenar información global de jugadores
**Campos principales**:
- `id`: Identificador único
- `wp_user_id`: Relación con usuario WordPress (opcional)
- `email`: Email único del jugador
- `first_name`, `last_name`: Nombre completo
- `phone`: Teléfono de contacto
- `global_level`: Nivel decimal (1.0-7.0)
- `created_at`, `updated_at`: Timestamps

#### Relación Jugador-Club (padel_player_clubs)
**Propósito**: Gestionar membresías específicas por club
**Campos principales**:
- `player_id`: FK a padel_players
- `club_id`: Identificador del club
- `site_id`: ID del sitio WordPress
- `local_level`: Nivel específico en este club (1.0-7.0)
- `membership_type`: member, guest, trial
- `is_active`: Estado de la membresía

#### Clubes (padel_clubs)
**Propósito**: Configuración específica de cada club
**Campos principales**:
- `site_id`: Identificador único del sitio
- `name`: Nombre del club
- `courts_count`: Número de pistas
- `opening_time`, `closing_time`: Horarios
- `price_per_hour`: Precio estándar

#### Partidos (Custom Post Type: padel_match)
**Propósito**: Gestionar partidos y reservas
**Meta-fields**:
- `match_date`: Fecha YYYY-MM-DD
- `match_time`: Hora HH:MM
- `court_id`: Número de pista (1-N)
- `players`: Array de 4 IDs de jugadores
- `match_level`: Nivel del partido (decimal)
- `min_level`, `max_level`: Rango permitido
- `created_by_user_id`: Creador del partido
- `is_full`: Boolean (siempre true con 4 jugadores)
- `price`: Precio del partido
- `status`: scheduled, playing, completed, cancelled

### 2.2 Diagrama de Relaciones

```
padel_players (1) ←→ (N) padel_player_clubs (N) ←→ (1) padel_clubs
      ↓
wp_users (WordPress)
      ↓
padel_match (CPT) → players (array de IDs)
```

### 2.3 Índices de Rendimiento
- `padel_players`: email, wp_user_id
- `padel_player_clubs`: club_id+site_id, local_level, is_active
- `padel_match`: match_date, match_time, court_id

## 3. Lógica de Negocio

### 3.1 Sistema de Niveles

#### Funcionamiento
1. **Niveles decimales**: Rango 1.0 a 7.0 (ej: 3.5, 4.2, 6.8)
2. **Creación de partido**: Nivel = nivel del jugador creador
3. **Compatibilidad**: Otros jugadores pueden unirse si su nivel está en ±1
4. **Ejemplo**: Partido nivel 4.0 acepta jugadores de 3.0 a 5.0

#### Reglas de Validación
- Nivel mínimo partido: max(1.0, nivel_creador - 1.0)
- Nivel máximo partido: min(7.0, nivel_creador + 1.0)
- Verificación automática antes de unirse

### 3.2 Gestión de Partidos

#### Flujo de Creación
1. **Validación inicial**: Fecha, hora, pista disponible
2. **Obtención nivel creador**: Consulta a padel_player_clubs
3. **Cálculo nivel partido**: Aplicar lógica ±1
4. **Creación CPT**: Post con meta-fields completos
5. **Estado inicial**: 1 jugador, 3 espacios disponibles

#### Flujo de Unión
1. **Verificación disponibilidad**: < 4 jugadores
2. **Validación nivel**: Compatibilidad ±1
3. **Verificación duplicados**: Jugador no inscrito
4. **Actualización**: Añadir a array players
5. **Estado final**: Si 4 jugadores → is_full = true

#### Estados del Partido
- **Creado**: 1 jugador (creador)
- **Parcial**: 2-3 jugadores
- **Completo**: 4 jugadores exactos
- **En juego**: Estado durante el partido
- **Finalizado**: Partido completado

### 3.3 Disponibilidad de Pistas

#### Lógica de Slots
- **Duración estándar**: 1.5 horas por partido
- **Intervalos**: Cada 30 minutos para flexibilidad
- **Verificación**: No solapamiento en misma pista/hora
- **Horarios**: Configurables por club (ej: 08:00-23:00)

## 4. Funcionalidades del Sistema

### 4.1 Dashboard Administrativo

#### KPIs Principales
- **Ingresos del día**: Suma de precios de partidos
- **Partidos programados**: Contador por fecha
- **Ocupación de pistas**: Porcentaje de uso
- **Jugadores activos**: Miembros con is_active=true
- **Próximos partidos**: Lista ordenada por fecha/hora

#### Gestión de Partidos
- **Vista calendario**: Navegación por fechas
- **Filtros**: Por pista, nivel, estado
- **Acciones**: Crear, editar, cancelar partidos
- **Detalles**: Información completa de cada partido

#### Gestión de Jugadores
- **Listado completo**: Tabla paginada con búsqueda
- **Perfiles individuales**: Información detallada
- **Edición de niveles**: Actualización por administrador
- **Historial**: Partidos jugados por jugador
- **Estados**: Activar/desactivar membresías

#### Configuración del Club
- **Información básica**: Nombre, dirección, contacto
- **Configuración de pistas**: Número y nombres
- **Horarios**: Apertura y cierre
- **Precios**: Tarifas estándar
- **Reglas**: Configuraciones específicas

### 4.2 Frontend Público

#### Shortcode Principal [padel_matches]
**Parámetros**:
- `date`: Fecha específica (default: hoy)
- `court`: Filtro por pista
- `show_create`: Mostrar botón crear (true/false)

#### Funcionalidades Frontend
- **Navegación temporal**: Días anterior/siguiente
- **Filtros dinámicos**: Por pista y nivel
- **Lista de partidos**: Tarjetas con información completa
- **Horarios disponibles**: Slots libres por pista
- **Creación de partidos**: Modal con formulario
- **Unión a partidos**: Verificación automática de compatibilidad

#### Información por Partido
- **Horario y pista**: Datos básicos
- **Nivel del partido**: Con rango permitido
- **Jugadores inscritos**: Lista con espacios disponibles
- **Precio**: Costo del partido
- **Acciones**: Unirse o estado completo

### 4.3 Sistema de Modales

#### Modal Crear Partido
- **Campos**: Fecha, hora, pista, precio
- **Validaciones**: Disponibilidad, formato
- **Confirmación**: Creación automática con nivel

#### Modal Unirse a Partido
- **Información**: Detalles del partido
- **Validación**: Compatibilidad de nivel
- **Confirmación**: Inscripción inmediata

## 5. Arquitectura Técnica

### 5.1 Estructura de Archivos

```
padel-manager-club/
├── padel-manager-club.php          # Plugin principal
├── includes/
│   ├── class-database-setup.php    # Configuración BD
│   ├── class-custom-post-types.php # CPT partidos
│   ├── class-match-manager.php     # Lógica partidos
│   ├── class-level-manager.php     # Gestión niveles
│   ├── class-player-manager.php    # Gestión jugadores
│   └── class-shortcode.php         # Frontend
├── admin/
│   ├── class-admin-dashboard.php   # Dashboard
│   ├── class-admin-matches.php     # Admin partidos
│   ├── class-admin-players.php     # Admin jugadores
│   └── class-admin-settings.php    # Configuración
├── assets/
│   ├── css/
│   │   ├── admin-styles.css
│   │   └── frontend-styles.css
│   ├── js/
│   │   ├── admin-scripts.js
│   │   └── frontend-scripts.js
│   └── images/
└── templates/
    ├── match-card.php
    ├── create-match-modal.php
    └── player-profile.php
```

### 5.2 Clases Principales

#### PadelMatchLevelManager
**Responsabilidad**: Gestión del sistema de niveles
**Métodos principales**:
- `calculateMatchLevel()`: Calcula nivel del partido
- `canPlayerJoinMatch()`: Verifica compatibilidad
- `getCompatiblePlayers()`: Lista jugadores válidos
- `formatLevel()`: Formato de visualización

#### PadelMatchManager
**Responsabilidad**: Lógica completa de partidos
**Métodos principales**:
- `createMatch()`: Creación con validaciones
- `addPlayerToMatch()`: Unión con verificaciones
- `removePlayerFromMatch()`: Salida y limpieza
- `getAvailableMatches()`: Consultas filtradas
- `isCourtAvailable()`: Verificación disponibilidad

#### PadelPlayerManager
**Responsabilidad**: Gestión de jugadores
**Métodos principales**:
- `createPlayer()`: Registro nuevo jugador
- `updatePlayerLevel()`: Modificación de nivel
- `getPlayersByClub()`: Listado por club
- `getPlayerHistory()`: Historial de partidos

#### PadelAdminDashboard
**Responsabilidad**: Panel administrativo
**Métodos principales**:
- `getKPIs()`: Cálculo de métricas
- `renderDashboard()`: Interfaz principal
- `getUpcomingMatches()`: Próximos partidos
- `generateReports()`: Informes estadísticos

### 5.3 Sistema de Hooks

#### Hooks de Acción
- `padel_manager_match_created`: Después de crear partido
- `padel_manager_player_joined`: Jugador se une
- `padel_manager_player_left`: Jugador abandona
- `padel_manager_match_completed`: Partido finalizado

#### Hooks de Filtro
- `padel_manager_match_display`: Personalizar visualización
- `padel_manager_player_level`: Modificar cálculo nivel
- `padel_manager_court_availability`: Personalizar disponibilidad

## 6. Interfaz de Usuario

### 6.1 Principios de Diseño

#### Estilo Visual
- **Paleta de colores**: Azul primario (#4F46E5), grises neutros
- **Tipografía**: System fonts para mejor rendimiento
- **Espaciado**: Grid de 8px para consistencia
- **Bordes**: Radio 8px, sombras sutiles
- **Responsive**: Mobile-first approach

#### Componentes Principales
- **Botones**: Primario, secundario, estados hover
- **Tarjetas**: Información estructurada con acciones
- **Modales**: Formularios y confirmaciones
- **Filtros**: Dropdowns y controles de navegación
- **Tablas**: Listados administrativos

### 6.2 Flujos de Usuario

#### Usuario Final
1. **Acceso**: Página con shortcode
2. **Navegación**: Selección de fecha
3. **Filtrado**: Por pista o nivel
4. **Visualización**: Lista de partidos disponibles
5. **Acción**: Crear partido o unirse
6. **Confirmación**: Feedback inmediato

#### Administrador
1. **Dashboard**: Vista general con KPIs
2. **Gestión**: Navegación por secciones
3. **Partidos**: Lista, creación, edición
4. **Jugadores**: Gestión de perfiles y niveles
5. **Configuración**: Ajustes del club
6. **Reportes**: Estadísticas y análisis

## 7. Configuración y Personalización

### 7.1 Opciones del Plugin

#### Configuración Básica
- `padel_club_name`: Nombre del club
- `padel_club_courts_count`: Número de pistas
- `padel_club_opening_time`: Hora apertura
- `padel_club_closing_time`: Hora cierre
- `padel_club_default_price`: Precio estándar
- `padel_club_contact_info`: Información de contacto

#### Configuración Avanzada
- `padel_club_match_duration`: Duración partidos (default: 90 min)
- `padel_club_booking_advance`: Días anticipación reserva
- `padel_club_cancellation_policy`: Política cancelaciones
- `padel_club_level_restrictions`: Restricciones por nivel

### 7.2 Personalización Visual

#### Variables CSS
```css
:root {
    --padel-club-primary: #4F46E5;
    --padel-club-secondary: #6B7280;
    --padel-club-success: #10B981;
    --padel-club-warning: #F59E0B;
    --padel-club-danger: #EF4444;
}
```

#### Clases CSS Principales
- `.padel-club-*`: Prefijo obligatorio
- `.padel-club-btn`: Botones base
- `.padel-club-card`: Tarjetas de contenido
- `.padel-club-modal`: Ventanas modales
- `.padel-club-grid`: Layouts en grid

## 8. Seguridad y Validaciones

### 8.1 Validaciones de Datos

#### Entrada de Datos
- **Fechas**: Formato YYYY-MM-DD, no pasadas
- **Horas**: Formato HH:MM, dentro de horarios
- **Niveles**: Rango 1.0-7.0, decimales válidos
- **Emails**: Formato válido, únicos
- **Teléfonos**: Formato internacional opcional

#### Permisos y Capacidades
- **Administradores**: Acceso completo
- **Usuarios registrados**: Crear/unirse partidos
- **Invitados**: Solo visualización
- **Nonces**: Validación CSRF en formularios

### 8.2 Sanitización y Escape

#### Datos de Entrada
- `sanitize_text_field()`: Campos de texto
- `sanitize_email()`: Direcciones email
- `absint()`: Números enteros
- `floatval()`: Números decimales

#### Datos de Salida
- `esc_html()`: Texto plano
- `esc_attr()`: Atributos HTML
- `wp_kses()`: HTML permitido

## 9. Rendimiento y Optimización

### 9.1 Consultas de Base de Datos

#### Índices Estratégicos
- Fechas de partidos para consultas temporales
- Niveles de jugadores para filtrado
- Estados activos para rendimiento
- Combinaciones club+site para multi-tenant

#### Consultas Optimizadas
- `WP_Query` con meta_query específicas
- Joins directos para relaciones complejas
- Límites y paginación en listados
- Cache de consultas frecuentes

### 9.2 Frontend Performance

#### Carga de Assets
- Enqueue condicional por página
- Minificación de CSS/JS
- Sprites para iconos
- Lazy loading de imágenes

#### AJAX Optimizado
- Debounce en búsquedas
- Paginación asíncrona
- Actualizaciones incrementales
- Manejo de errores robusto

## 10. Escalabilidad Multi-Club

### 10.1 Arquitectura Multi-Tenant

#### Aislamiento de Datos
- `site_id` en todas las tablas principales
- Consultas filtradas por instalación
- Configuraciones independientes
- Usuarios compartidos opcionales

#### Gestión Centralizada
- Jugadores globales con niveles locales
- Transferencias entre clubes
- Estadísticas agregadas
- Sincronización de datos

### 10.2 Consideraciones de Crecimiento

#### Escalabilidad Horizontal
- Particionado por fecha
- Archivado de datos históricos
- Índices compuestos optimizados
- Cache distribuido

#### Monitoreo y Métricas
- Logs de rendimiento
- Métricas de uso
- Alertas automáticas
- Análisis de carga

## 11. Plan de Implementación

### 11.1 Fases de Desarrollo

#### Fase 1: Foundation (2-3 semanas)
1. Configuración del plugin base
2. Creación de tablas y CPT
3. Clases principales sin UI
4. Validaciones básicas

#### Fase 2: Core Features (3-4 semanas)
1. Dashboard administrativo
2. Gestión de partidos completa
3. Shortcode básico frontend
4. Sistema de niveles funcional

#### Fase 3: UI/UX (2-3 semanas)
1. Estilos CSS completos
2. JavaScript interactivo
3. Modales y formularios
4. Responsive design

#### Fase 4: Testing & Polish (1-2 semanas)
1. Testing exhaustivo
2. Optimizaciones de rendimiento
3. Documentación usuario
4. Preparación para producción

### 11.2 Entregables por Fase

#### Documentación Técnica
- Especificaciones de API
- Guía de instalación
- Manual de usuario
- Documentación de hooks

#### Testing
- Unit tests para clases principales
- Integration tests para flujos completos
- User acceptance testing
- Performance testing

Este documento proporciona la especificación completa para que un desarrollador pueda implementar el plugin sin necesidad de decisiones de diseño adicionales. Cada sección detalla exactamente qué construir, cómo debe funcionar y qué consideraciones técnicas aplicar.