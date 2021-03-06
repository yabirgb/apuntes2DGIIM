#### Hilos kernel

Son hilos que no tienen espacio de direcciones de usuario. Por tanto, su descriptor tiene `task_struct->mm=NULL`. Realizan labores de sistema (sistuyendo a los demonios de Unix)  Para crearlos, debemos hacerlo desde otro hilo kernel con la funcion kthread_create().

#### Terminar un proceso

* Involuntariamente: recibe una señal, y se aplica la acción por defecto, que es terminar.

* Voluntariamente: se utiliza `exit()` o `return()` que finalizan el proceso primero a nivel de biblioteca, o utilizando directamente `_exit()` (llamada al SO) que no da la oportunidad de finalizar el proceso a nivel de biblioteca.

#### Exit()
El objetivo de `do_exit()` es borrar todas las referencias del proceso. Sigue estos pasos:

* Activa `PF_EXITING` (bandera que notifica que el proceso va a terminar)

* Decrementa los contadores de uso de *mm_struct*, *fs_struct* y *files_struct*.  Si estos contadores alcanzan el valor 0, se liberan los recursos.

* Ajusta el *exit_code* del descriptor, que será devuelto al padre, con el valor pasado al inovar a *exit()*. 

* Envía al padre la señal de finalización; si tiene algún hijo le busca un padre en el grupo o el initi, y pone el estado a *TASK_ZOMBIE*.

* Invoca a `schedule()`para ejecutar otro proceso.

Solo liberar la pila kernel, el *thread_info* y *task_struct* para que el padre pueda recuperar el código de finalización, para ello se invoca a `wait()`

#### wait()

Llamada que bloquea a un proceso padre hasta que uno de sus hijos finaliza; cuando esto ocurre, devuelve al llamador el PID del hijo finalizado y el estado de finalización.

Esta función invoca a `release_task()` que:

* Elimina el descriptor de la lista de tareas.

* Si es la última tarea de su grupo, y el líder está zombi, notifica al padre del líder zombi.

* Libera la memoria de la pila kernel, *thread_info* y *task_struct*


---
### 3\. Planificación de la CPU

El planificador asigna los procesos a ser ejecutados por el procesador/es a lo largo del tiempo, de forma que se cumplan los objetivos de tiempo de respuesta, rendimiento y eficiencia del procesador. 

####Tipos de planificador:
* **Planificador a largo plazo:** Es el que toma la decisión de añadir un nuevo proceso al conjuntos de procesos a ser ejecutados. Es decir, toma un programa o trabajo y lo convierte en un proceso que lo añade a la cola de procesos de "listo" del planificador a corto plazo. En algunos sistemas, en vez de pasar directamente a la cola de "listo", pasa primero a la zona de intercambio, es decir, se añaden a la cola del planificador a medio plazo.

* **Planificador a medio plazo:** Es parte de la función de intercambio. Toma la decisión de añadir  un proceso al número de procesos que están parcialmente o totalmente en la memoria principal. (La verdad es que de este no me he entardo muy bien, pero con la ilustración, se pilla la idea aprox.)

* **Planificador a corto plazo:** Es el 'scheduler' y es el que se ejecuta con más frecuencia. Dedice qué proceso de entre la cola de procesos 'listos' ejecutar el siguiente. Este planificador se invoca siempre que ocurre un evento que conlleva el bloqueo del actual, y da la oportunidad de bloquear al proceso actualmente en ejecución para ejecutar otro. El criterio con el que se elige el siguiente proceso, dependerá del algoritmo de planificación que se utilice.

Ilustrativamente:

![](/imagenes/tipos_planificadores.JPG)

Podemos distinguir dos tipos de planificación según la política de expulsión:

* **Planificación no apropiativa (nonpreemptive):** *"Sin expulsión".* Una vez que el proceso está ejecutandose, continúa ejecutándose hasta que termine o se bloquee para esperar una E/S o solicitar un sevicio al sistema operativo. Es decir, al proceso actual no se le puede retirar la CPU. Estos se han utilizado como mecanismo de grano grueso de sincronización en modo kernel.

* **Planifcación apropiativa (preemptive):** *"Con expulsión".* Un proceso que está ejecutándose puede ser interrumpido y pasar al estado de listo. Esta decisión puede ser tomada cuando se crea un nuevo proceso, un proceso pasa de bloqueado a listo o por interrupciones de reloj. Los kernel de tiempo-real necesitan ser apropiativos.
 
#### Algoritmos de planificación:

* **FIFO - First in, first out**

Cuando un proceso pasa al estado listo se une a la cola. Cuando el proceso que está en ejecución deja de ejecutadar, se selecciona el proceso que más lleva en la cola para pasar a ejecutando.

Un inconveniente es que si un proceso corto está detrás de uno largo, el corto tendrá que estar mucho rato esperando para poder ejecutarse en comparación con lo que tarda en ejecutarse. Además, otro problema se presenta cuando un proceso que está limitado por el procesador está ejecutándose, ya que el resto debe esperar, y probablemente los procesos que estaban esperando una E/S pasarán de bloqueados a listos en ese tiempo, dejando los dispositivos de E/S ociosos (sin usar), aunque haya posible trabajos que puedan realizar.  Cuando el proceso limitado por la CPU deja de ejecutarse, los procesos que necesitaban E/S se ejecutarán y pueden volverse
a bloquear por la necesidad de E/S. Esto hace que el uso del procesor de los dispositivos de E/S sean ineficientes.


* **Prioridades**

A cada proceso se le asigna una prioridad. En lugar de una sola cola de procesos listos, se proporciona un conjunto de oclas en orden decreciente de prioridades: CL0, CL1, ... , CLn. Cuando se va a elegir un proceso para la ejecución, se empieza por la prioridad más alta CL0, y se elige un proceso de esa cola utilizando otro de los algoritmos de planificación. Si está vacía se pasa a CL1 y así sucesivamente. 

Un problema es que los procesos de prioridad baja puede que no lleguen a ejecutarse. (pueden sufrir inanición).


* **Round robin**

Una forma de reducir el "castigo" de los procesos cortos en FIFO es usando este algoritmo. Se genera una interrupción cada cierto intervalo de tiempo, donde el proceso actualmente en ejecución pasa a listo y segun FIFO se selecciona el siguiente.

Los intervalos de tiempos que se usan se llaman *quantum*, es necesario comentar si que el quantum es muy pequeño se puede producir una sobrecarga de procesamiento ya que a os procesos no les da tiempo a hacer nada en ese tiempo y siempre necesitarán mas de un quantum. Si en cambio el queantum es muy grande se degenera a un FIFO. Lo ideal es que sea ligeramente mayor al tiempo requerido por una interrupción.

Una desventaja es que los procesos que necesitan E/S se bloquean antes de tiempo y tienen que pasar a bloqueados, mientras que los que tienen limitado el procesador usan siempre todo el quantum, por lo que hay una desigualdad en la distribución de tiempo.

* **SPN - shortest process next**

Es una política no apropiativa que elige en proceso con el tiempo de procesamiento más corto.

Un problema es la necesidad de estimar el tiempo de ejecución de cada proceso. (En Stalling hay muchas formulitas de como lo hace, pero no me suena de que se haya ni siqueira nombrado este algoritmo en clase por lo que lo veo innesario)

Otro problema es la iniciación que pueden sufrir lo procesos más largos.

* **SRT - shortes remaining time**

Es la versión apropiativa de la SPN. Si durante la ejecución de un proceso, se crea uno nuevo que tiene menos tiempo de ejecución restante que el que se está ejecutando, se pasa el actual a listo y se ejecuta el nuevo.

* **Retroalimentación - feedback**

Da preferencia a los procesos más cortos penalizando a los que han estado ejecutando más tiempo. Para ello se realiza una planificación apropiativa por interrupciones (la version apropiativa del de priodridades).
Cuando un nuevo proceso entra al sistema, se le situa en CL0 (cola con mayor prioridad). Al expulsarlo se le devuelve a listo
pero en la cola CL1, cada vez que es expulsado va a una cola menor. Los procesos cortos terminarán pronto, ya que no tendrán que irse muy atrás en las colas, mientras que los largos se degradan gradualmente. En cada cola se elige según FIFO. En la cola con menor prioridad, el proceso no puede descender más, por lo que vuelve a la misma cola, convirtiendose esta cola en un round robin.

Los procesos largos pueden alargarse demasiado, en especial si no paran de entrar procesos cortos. Una solución es compensar los tiempos de expulsión, la cola CL0 tendrá un quantum de una unidad de tiempo, el CL1 de dos unidades, y así hasta CLn que tendra 2^<sup>n </sup> unidades de t tiempo antes de ser expulsado.

####Proceso nulo

Este proceso se crea para siempre haya un proceso que el planificador a corto plazo que pueda encontrar un proceso en preparados para ejecutar. Por tanto, este proceso siempre está listo para ejecutarse y tiene la prioridad más baja.

**Implementación** 
Gracias al proceso nulo, la implementación del planificador a óptima, y quedaría así:

``` cpp
Planificador(){
  while (true){
	Selecciona(Pj)

  Cambio contexto(Pi,Pj)
}

```

Podemos hacerlo así ya que siempre hay un proceso nulo que podemos seleccionar, si no lo hubiera tendríamos que añadir un `if(cola_preprados==vacia) halt; else Selecciona (Pj)`, por lo que teniendo en cuenta que llamamos a planificar muchas veces, quitar un if genera más eficacia.
