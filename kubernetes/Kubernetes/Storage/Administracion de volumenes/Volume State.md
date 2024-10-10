## Volume state

Cuando un `pod` hace un `claim` a un `volumen` el cluster inspecciona el `claim` con los requerimientos del mismo y monta el `volumen` para el `pod`.
Una vez el `pod` tiene el `claim` y esta en el estado **bound** (enlazado) el `volumen` pertenece al `pod`

Un volumen tiene estos **states**:

  * **Available**:  El volumen esta **DISPONIBLE** ya que aun no fue  `bound` a  un `claim`
  * **Bound**:  El volumen esta atado a un `claim`
  * **Released**:  El `claim` fue borrado pero el volumen aun no esta disponibilizado
  * **Failed**: El volumen fallo

El volumen es considerado **released** cuando el `claim` esta borrado, pero aun no esta **available** para otro `claim`.