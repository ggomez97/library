## Volume Access Mode


Para difrentes tipos de storage hay diferentes tipos de modos ya que  por ej. NFS soporta multiples clientes "write" y iSCSI solo uno

Los tipos de accesos son:

**ReadWriteOnce (RWO)**:

- Permite que el volumen sea montado en modo lectura/escritura **por un único nodo**.
- Es el modo más común para volúmenes persistentes donde solo un pod necesita escribir en el volumen.
- Aún si múltiples pods están en el mismo nodo, solo uno podrá montarlo con acceso de lectura y escritura.

**ReadOnlyMany (ROX)**:
- Permite que el volumen sea montado en modo **solo lectura por múltiples nodos**.
- Varios pods pueden acceder al volumen en modo de solo lectura, incluso si están distribuidos en varios nodos.

**ReadWriteMany (RWX)**:
- Permite que el volumen sea montado en modo **lectura/escritura por múltiples nodos** simultáneamente.
- Es útil en escenarios donde varios pods, incluso en diferentes nodos, necesitan acceder y modificar los mismos datos.
- Generalmente está soportado por sistemas de almacenamiento como NFS o soluciones de almacenamiento en la nube.

**ReadWriteOncePod (RWOP)**:
- Este es un modo más reciente introducido en Kubernetes **v1.22**.
- Permite que el volumen sea montado en modo **lectura/escritura por un único pod**, incluso si hay varios pods en el mismo nodo. Esto asegura que solo un pod específico pueda acceder al volumen para leer y escribir.
