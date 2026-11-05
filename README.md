# Revelar indicadores al pasar el cursor por la barra de Omarchy

Este repositorio documenta cómo recuperar el comportamiento que muestra los indicadores y el widget Pomodoro al pasar el cursor por cualquier parte de la barra. Está pensado para que un agente lo adapte en otra computadora con Omarchy.

## Problema y causa

En el equipo original, Omarchy 4.5.0-3 limita la señal de revelado al área de la barra anterior al bloque central. Al pasar el cursor sobre el reloj o a su derecha, los indicadores ocultos y el widget Pomodoro no aparecen. El widget Pomodoro usa la propiedad compartida `bar.centerSectionRevealHeld`, por lo que modificar solo ese widget no recuperaría los indicadores.

El cambio consiste en mantener activa esa señal mientras el cursor esté en cualquier parte de la barra. No modifica el código de Pomodoro ni sus datos. La barra usada aquí es la de Omarchy Shell, basada en Quickshell. No es Waybar.

## Instrucciones para el agente

1. Inspecciona primero el sistema, la versión de Omarchy y la configuración existente. Identifica la barra activa y cualquier personalización antes de editar. No sobrescribas un plugin o una configuración existente.

   ```bash
   omarchy version
   omarchy-shell shell listPlugins | jq '.[] | select(.kinds | index("bar")) | {id, active, clonedFrom}'
   cat ~/.config/omarchy/shell.json
   ```

2. Inspecciona la implementación instalada y compárala con [bar-hover.patch](bar-hover.patch). El parche es una referencia para Omarchy 4.5.0-3, no presupone que otras versiones tengan la misma estructura. Si el código cambió, entiende el comportamiento nuevo y adapta el cambio. Si ya funciona, no hagas cambios.

   ```bash
   rg -n 'peeksAt|centerSectionRevealHeld|setBarHovered' /usr/share/omarchy/shell/plugins/bar
   ```

3. Si está activa la barra de fábrica y no existe ya un clon del usuario, crea un clon con el comando de Omarchy. Este comando copia la barra a `~/.config/omarchy/plugins/<usuario>.bar/` y la selecciona. Si ya existe una barra personalizada, examínala y adapta el cambio allí, conservando sus modificaciones. No ejecutes el comando de clonación sobre un destino existente.

   ```bash
   omarchy plugin clone omarchy.bar
   ```

   No edites `/usr/share/omarchy/`, ya que pertenece al paquete y sus cambios se pierden al actualizar.

4. En el `Bar.qml` del clon, localiza el `HoverHandler` que llama a `root.setBarHovered(peeking)`. En la versión probada, cambia esta expresión:

   ```qml
   readonly property bool peeking: hovered && BarModel.peeksAt(point.position, barWindow.peekEdge, root.vertical)
   ```

   por:

   ```qml
   readonly property bool peeking: hovered
   ```

   Conserva `onPeekingChanged` y el tratamiento de destrucción del componente. Actualiza los comentarios cercanos que describan el límite anterior. Haz una copia de seguridad de cualquier archivo personalizado antes de editarlo.

5. Comprueba que el clon esté seleccionado. Para cargar el cambio en el equipo original fue necesario reiniciar Omarchy Shell. El reinicio cierra paneles y puede reiniciar el estado en memoria de los plugins. Si Pomodoro tiene una sesión activa o pausada, consulta al usuario antes de reiniciar para no perderla. El plugin de este equipo expone `omarchy-shell pomodoro status`, aunque otros plugins podrían no tener ese comando.

   ```bash
   omarchy-shell shell listPlugins | jq '.[] | select(.kinds | index("bar")) | {id, active}'
   omarchy restart shell
   ```

6. Verifica con el cursor real que los indicadores y Pomodoro aparezcan al pasar por la izquierda y por la derecha del reloj, que permanezcan visibles al cruzar la barra y que se oculten al salir. Comprueba que los paneles sigan abriendo y que no haya errores nuevos relacionados con la barra en el registro del usuario. Confirma que el resultado coincide con la configuración de ese equipo.

   ```bash
   journalctl --user -b --no-pager | rg -i 'bar option|bar\.qml|pomodoro|error' | tail -50
   ```

Si Pomodoro no está instalado o no está configurado en la barra de destino, este cambio no lo instala. Comprueba su instalación y configuración por separado antes de atribuir su ausencia al hover.

## Archivos y mantenimiento

- `bar-hover.patch` contiene la diferencia exacta aplicada a `Bar.qml` respecto de la barra de Omarchy 4.5.0-3. Incluye cambios de comentarios además de la expresión funcional. No lo apliques a ciegas sobre otra versión.
- La barra clonada es una copia local. Una actualización de Omarchy no incorporará automáticamente sus futuras correcciones o funciones a ese clon. Después de actualizar, compara el clon con la versión instalada y adapta la personalización cuando corresponda.

## Verificación en el equipo original

Se confirmó con eventos de movimiento de un dispositivo virtual que, al situar el cursor a la derecha del reloj, aparecieron Pomodoro y los indicadores. También se comprobó que se ocultaran al sacar el cursor de la barra. La personalización quedó activa en una copia local de la barra y se cargó tras reiniciar Omarchy Shell.
