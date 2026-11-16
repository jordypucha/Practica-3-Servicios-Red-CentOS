# Práctica 3: La Línea de Comando para Levantar Servicios en Red

**Universidad de Cuenca (UCUENCATEC) — Instituto Superior Universitario Tecnológico del Azuay**  
**Carrera:** Tecnología Superior en Administración de Infraestructura y Plataformas Tecnológicas  
**Asignatura:** `4292 - TICS-00087` — Sistemas Operativos I (Linux)  
**Docente Responsable:** Ing. Jonnathan F. Nivicela A.  
**Laboratorio:** C-007, Campus Balzay | **Periodo Académico:** Septiembre 2026 – Febrero 2027  
**Fecha de Ejecución:** 15 de noviembre de 2026  

**Autores:**
* **Justin Omali Márquez Pesántez** — Estudiante, UCUENCATEC / TecAzuay, Cuenca - Ecuador
* **Jordy Steven Pucha Galarza** — Estudiante, UCUENCATEC / TecAzuay, Cuenca - Ecuador

---

## Resumen

Este documento técnico presenta el despliegue, endurecimiento y validación operativa de cuatro servicios críticos de red —acceso remoto seguro (**SSH**), transferencia de archivos (**FTP** mediante `vsftpd`), publicación web (**HTTP** mediante Apache `httpd`) y sincronización horaria corporativa (**NTP** mediante `chronyd`)— sobre un servidor **CentOS Stream** virtualizado en modo puente (*Bridged Adapter*) dentro de Oracle VM VirtualBox. La metodología integra la configuración explícita del cortafuegos basado en zonas (`firewalld`), el ajuste de políticas de control de acceso obligatorio (**SELinux**) y el aislamiento de cuentas de servicio mediante intérpretes restringidos (`/sbin/nologin`). Asimismo, se documenta un **Runbook de Operaciones** estructurado en la Wiki del repositorio junto con un plan de contingencia que reproduce y resuelve los tres incidentes más frecuentes en entornos de producción: el rechazo de autenticación PAM por ausencia de `/sbin/nologin` en `/etc/shells`, el bloqueo HTTP `403 Forbidden` derivado de etiquetas SELinux heredadas (`user_home_t`) y la denegación de consultas NTP externas en redes locales. Las pruebas cliente-servidor ejecutadas desde Windows confirman la disponibilidad, integridad y aislamiento de cada servicio.

**Palabras clave:** CentOS Stream, Servicios en Red, `firewalld`, SELinux, `vsftpd`, Apache `httpd`, `chronyd`, Runbook Técnico.

---

## Abstract

This technical report documents the deployment, hardening, and operational validation of four core network services—secure remote administration (**SSH**), file transfer (**FTP** via `vsftpd`), web publishing (**HTTP** via Apache `httpd`), and corporate time synchronization (**NTP** via `chronyd`)—on a **CentOS Stream** server virtualized in bridged networking mode within Oracle VM VirtualBox. The methodology encompasses explicit zone-based firewall configuration (`firewalld`), Mandatory Access Control policy tuning (**SELinux**), and service account isolation using non-interactive shells (`/sbin/nologin`). Furthermore, an operational **Technical Runbook** is structured across the repository Wiki alongside a contingency plan that reproduces and resolves three common production incidents: PAM authentication rejection (`530 Login incorrect`) caused by omitting `/sbin/nologin` from `/etc/shells`, HTTP `403 Forbidden` errors resulting from inherited SELinux file contexts (`user_home_t`), and external NTP query blocking across local subnets. Client-server validation tests conducted from Windows verify the availability, integrity, and security isolation of every deployed service.

**Keywords:** CentOS Stream, Network Services, `firewalld`, SELinux, `vsftpd`, Apache `httpd`, `chronyd`, Technical Runbook.

---

## 1. Introducción

La administración moderna de infraestructuras Linux exige dominar la interfaz de línea de comandos (CLI) para aprovisionar, auditar y asegurar servicios esenciales de red sin depender de entornos gráficos[cite: 1]. En distribuciones empresariales derivadas de Red Hat Enterprise Linux (RHEL), como **CentOS Stream**, la puesta en producción de demonios de red implica coordinar tres capas de defensa: el filtrado de paquetes en red mediante `firewalld`, el confinamiento de procesos y contextos de archivos mediante *Security-Enhanced Linux* (SELinux) y la gestión granular de privilegios de usuario mediante módulos PAM (*Pluggable Authentication Modules*)[cite: 1].

De acuerdo con Hidalgo Guijarro et al. [1], una configuración robusta de servidores Linux requiere delimitar estrictamente los puertos expuestos, restringir las cuentas operativas destinadas exclusivamente a transferencia de archivos y garantizar una referencia horaria unificada para la correlación confiable de registros de auditoría (*logs*). En el presente informe y su correspondiente **Runbook Operativo en GitHub Wiki**, se documenta paso a paso la implementación, verificación de calidad (QA) y resolución de fallos reales sobre CentOS Stream conectándose directamente con un nodo cliente Windows anfitrión.

---

## 2. Objetivos

### 2.1 Objetivo General
Configurar servicios de red corporativos (acceso remoto SSH, almacenamiento FTP, servidor web HTTP y sincronización horaria NTP) sobre CentOS Stream mediante un entorno virtualizado, aplicando políticas estrictas de seguridad en `firewalld` y SELinux para garantizar su correcto funcionamiento y validación operativa[cite: 4].

### 2.2 Objetivos Específicos
* Desplegar y configurar un servidor de almacenamiento y transferencia de archivos (`vsftpd`) estableciendo el enjaulamiento (`chroot`), un usuario sin consola interactiva (`/sbin/nologin`) y las políticas de seguridad requeridas en el sistema operativo[cite: 4, 5].
* Implementar un servicio de sincronización horaria corporativa (`chronyd`/NTP) para mantener la precisión temporal entre el servidor CentOS Stream y el equipo cliente Windows[cite: 4, 6].
* Desplegar un servidor web HTTP (`httpd`), modificando la página predeterminada (`/var/www/html/index.html`) para incorporar los datos de los integrantes y un recurso gráfico validado bajo contextos SELinux[cite: 4, 5, 7].
* Establecer las directrices de seguridad en el cortafuegos (`firewalld`) y políticas de control de acceso obligatorio (SELinux) requeridas para la operación segura de cada servicio[cite: 4].
* Documentar el procedimiento de validación QA y el plan de contingencia (*troubleshooting*) ante incidentes comunes de seguridad y red en las cuatro pestañas del Runbook de GitHub Wiki[cite: 4, 6].

---

## 3. Arquitectura, Topología y Especificaciones del Entorno

### 3.1 Recursos de Hardware y Software Empleados

| Componente | Especificación Técnica | Función en la Infraestructura |
| :--- | :--- | :--- |
| **Equipo Anfitrión (Cliente)** | Windows (PowerShell 5.1, Navegador Web) | Nodo cliente para pruebas HTTP, sincronización NTP (`w32time`) y administración remota[cite: 5, 6, 22, 25]. |
| **Hipervisor** | Oracle VM VirtualBox (Red en modo *Bridged Adapter*) | Virtualización del servidor invitado con visibilidad directa en la subred local `192.168.18.0/24`[cite: 5, 15]. |
| **Servidor Invitado** | CentOS Stream (`localhost.localdomain`, interfaz `enp0s3`) | Servidor corporativo con dirección IPv4 `192.168.18.157/24` y usuario administrativo `jgalarza`[cite: 15, 21]. |
| **Cliente Remoto** | MobaXterm Personal Edition v26.5 | Ejecución de sesiones SSH administrativas y transferencias de archivos por FTP pasivo[cite: 5, 15, 18]. |
| **Servicios Desplegados** | OpenSSH (`sshd`), `vsftpd`, Apache `httpd` 2.4.63, `chronyd` 4.9 | Pila de servicios de red auditados bajo `firewalld` y SELinux en modo `Enforcing`[cite: 4, 21, 24]. |

### 3.2 Diagrama de Arquitectura y Topología de Red

La comunicación entre el host cliente Windows (`192.168.18.31`) y la máquina virtual CentOS Stream (`192.168.18.157`) opera sobre la red local `192.168.18.0/24` mediante el adaptador puente de VirtualBox:

```mermaid
flowchart LR
    subgraph HOST["Host Anfitrión Windows (192.168.18.31)"]
        MOBA["MobaXterm v26.5\n(Cliente SSH / FTP)"]
        WEB["Navegador Web\n(Cliente HTTP)"]
        W32["Servicio w32time\n(Cliente NTP)"]
    end

    subgraph NET["Red Local LAN - Modo Puente (192.168.18.0/24)"]
        BR["VirtualBox Bridged Adapter"]
    end

    subgraph VM["Servidor CentOS Stream (192.168.18.157 - enp0s3)"]
        FW["Firewalld (Zona Public) + SELinux (Enforcing)"]
        SSH["OpenSSH (sshd)\nPuerto 22/TCP"]
        FTP["vsftpd (chroot)\nPuertos 21/TCP + 40000-40100/TCP"]
        HTTP["Apache httpd 2.4.63\nPuerto 80/TCP (/var/www/html)"]
        NTP["chronyd 4.9 (Stratum 3/10)\nPuerto 123/UDP"]
    end

    MOBA -- "SSH (22/TCP)" --> BR
    MOBA -- "FTP Pasivo (21, 40000-40100/TCP)" --> BR
    WEB -- "HTTP GET (80/TCP)" --> BR
    W32 -- "NTP Sync (123/UDP)" --> BR

    BR --> FW
    FW --> SSH
    FW --> FTP
    FW --> HTTP
    FW --> NTP
    NTP -. "Upstream NTP (123/UDP)" .-> EXT["Servidor NTP Externo\n(186.69.15.72)"]
