# Servidor seguro con Ansible

Playbook y roles de **Ansible** para dejar un servidor **Ubuntu 24.04** listo para producción con una configuración endurecida. Los roles se prueban con **Molecule** en un contenedor con systemd. La prueba aplica el playbook, verifica que una segunda pasada no cambie nada (idempotencia) y comprueba el resultado.

## Roles

| Rol | Qué hace |
|---|---|
| `base` | Paquetes base, zona horaria `America/Montevideo`, sincronización horaria con chrony, **actualizaciones de seguridad automáticas** con reinicio programable y parámetros del núcleo endurecidos (`rp_filter`, sin redirecciones ICMP, `kptr_restrict`…) |
| `usuarios` | Usuario administrador con clave SSH exclusiva y sudo (archivo validado con `visudo`); contraseña de root bloqueada. Si falta la clave pública, se detiene antes de cortar el acceso. |
| `ssh` | `PermitRootLogin no`, sólo clave pública, `AllowUsers`, 3 intentos, sin reenvíos ni X11, algoritmos modernos (incluido el intercambio poscuántico `sntrup761x25519`). La plantilla se valida con `sshd -t` antes de instalarse. |
| `cortafuegos` | UFW con denegación entrante por defecto, SSH con límite de conexiones y HTTP/HTTPS/HTTP3. La regla de SSH se agrega **antes** de activar el cortafuegos, para no perder la conexión. |
| `nginx` | Sin versión en las respuestas, límite de pedidos por IP, cabeceras de seguridad (CSP, `X-Frame-Options`, `nosniff`…), bloqueo de archivos ocultos (`.git`, `.env`). `nginx -t` valida la configuración antes de cada recarga. |
| `fail2ban` | Cárceles para SSH (modo agresivo) y para el límite de pedidos de nginx, leyendo el *journal* de systemd y bloqueando con UFW, con tiempo de bloqueo creciente. |

## Uso

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --check --diff     # simulación
ansible-playbook site.yml                    # aplicar
ansible-playbook site.yml --tags ssh         # sólo un rol
```

El inventario de ejemplo está en `inventario/produccion.yml`, con una IP del rango de documentación, y las variables del grupo en `inventario/group_vars/servidores.yml`. Cada rol documenta sus variables en `defaults/main.yml`, con el nombre del rol como prefijo.

## Pruebas

`molecule test` hace lo siguiente:

1. Crea un contenedor Ubuntu 24.04 con systemd.
2. Aplica el playbook.
3. Lo aplica de nuevo y falla si algo cambia.
4. Verifica el resultado con `verify.yml`:
   - la configuración **efectiva** de OpenSSH (`sshd -T`);
   - el estado de UFW y sus reglas;
   - la cárcel de fail2ban activa;
   - las cabeceras HTTP de nginx;
   - el `403` al pedir `/.env`;
   - los permisos de sudo, la clave autorizada y la zona horaria.

En el escenario de prueba no se aplican los parámetros del núcleo (`base_sysctl_aplicar: false`): en un contenedor afectarían al anfitrión. El archivo sí se genera.

## Integración continua

1. **Lint**: `yamllint`, `ansible-lint` con el perfil `production` (el más estricto) y `--syntax-check`.
2. **Molecule**: la secuencia completa de prueba sobre Docker en GitHub Actions.

## Autor

Matías Nicolás Echeveste Cardozo — licencia MIT.
