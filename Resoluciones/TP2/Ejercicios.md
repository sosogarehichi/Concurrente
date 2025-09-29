# Práctica 2

## Ejercicio 7
Suponga que se tiene un curso con 50 alumnos. Cada alumno debe realizar una tarea y existen 10 enunciados posibles. Una vez que todos los alumnos eligieron su tarea, comienzan a realizarla. Cada vez que un alumno termina su tarea, le avisa al profesor y se queda esperando el puntaje del grupo (depende de todos aquellos que comparten el mismo enunciado). Cuando un grupo terminó, el profesor les otorga un puntaje que representa el orden en que se terminó esa tarea de las 10 posibles.
Nota: Para elegir la tarea suponga que existe una función elegir que le asigna una tarea a un alumno (esta función asignará 10 tareas diferentes entre 50 alumnos, es decir, que 5 alumnos tendrán la tarea 1, otros 5 la tarea 2 y así sucesivamente para las 10 tareas).

```c
sem mutexListos[A] = ([A] 0); // para sincronizar el comienzo de la tarea
sem mutexTarea[A] = ([A] 0);
sem hayNota[G] = ([G] 0);
sem mutexCola = 1;
sem hayTarea = 0;


int A = 50; // cantidad de alumnos
int G = 10; // cantidad de grupos
int notas[G];
int tareas[A];
queue cola;

Process Alumno[id: 0..49]{
    int numTarea;
    text tarea;

    // espera recibir tarea
    // cuando recibe la tarea
    // aumenta en uno el contador de tareas entregadas
    // si ya están todos, señaliza que se puede comenzar --> ver si profesor o alumno

    P(mutexTarea[id]); // espera que le asignen número de tarea
    numTarea = tareas[id]; // número de tarea asignado
 

    P(mutexListos[id]); // espera que estén todos

    // resuelve tarea
    tarea = resolverTarea(numTarea);

    // señaliza semaforo grupal? // un semáforo por grupo para suma de var del grupo
    // si se completa --> le avisan al profesor
    // CORRECCIÓN: cuando un alumno termina avisa al profesor
    // se quedan esperando la nota del grupo (vector)
    P(mutexCola);
    push(cola, (numTarea, tarea));
    V(mutexCola);

    V(hayTarea); // señalización

    P(hayNota[numTarea]);
    
    // puede acceder a nota en notas[numTarea] --> nota del grupo
}

// profesor al desencolar va sumando en una var de grupo
// cuando están los 5 le asigna el valor a la nota del grupo

Process Profesor() {
    text tarea;
    int numGrupo;
    int nota = 10;
    int cantFin[G] = ([G] 0);
    int entregas = 0;

    // reparte tareas
    for (i=0; i<A; i++) {
        tareas[i] = elegir();
        V(mutexTarea[i]);
    }
    // termina de repartir y avisa a los alumnos que pueden arrancar
        for (i=0; i<A; i++) {
            V(mutexListos[i]);
    }

    while (entregas < A) {
        // espera a que haya una tarea
        P(hayTarea);
        // popea la tarea y la corrige
        P(mutexCola);
        pop(cola, (numGrupo, tarea));
        V(mutexCola);
        entregas++;

        // chequea que todos hayan entregado
        cantFin[numGrupo]++;
        if (cantFin[numGrupo] == 5) {
            notas[numGrupo] = nota;
            // entrega la nota
            for (i=0; i<5; i++) {
                V(hayNota[numGrupo]);
            } 
            nota--;
        }
    }
}
```

## Ejercicio 8

```c
sem mutexPieza = 1;
sem mutexFin = 1;
sem mutexLlegada = 1; // inicializa en 1 para que el primero pueda entrar
sem esperaInicio[E] = ([E] 0);
sem iniciarTrabajo = 0; // en 0 porque es de señalización
sem finJornada = 0;
sem esperarPremio = 0;

int cantP[E] = ([E] 0);
int E; // cant empleados
int T; // cant piezas
int piezas = 0; // piezas fábricadas
int fin = 0; // empleados que finalizaron
queue cola;

// esperar a que lleguen todos los empleados
// var cant que sume --> semáforo protege la variable
// si soy el último empleado como le aviso a los demás?
// lo hace la fábrica 

Process Empleado[id: 0..E-1] {

    // avisan que llega
    // cuando están todos le avisan a a fábrica
    // se queda esperando que loe permitan iniciar a trabajar

    P(mutexLLegada);
    cantE++;
    if (cantE == E) V(iniciarTrabajo);
    V(mutexLLegada);

    P(esperaInicio[id]);

    // alguien podría modificar el valor de pieza mientras se consulta?
    P(mutexPieza);
    while (piezas < T) {
        // mientras haya piezas, toman una
        // luego de realizarla se suman en su variable contador
        tomarPieza();
        piezas++;
        realizarPieza();

        P(mutexCargar);
        cantP[id]++;
        V(mutexCargar);
    }
    V(mutexPieza);

    // esperan para poder avisar que terminaron
    P(mutexFin);
    fin++;
    if (fin == E) V(finJornada);
    V(mutexFin);

    P(esperarPremio);
}

Process Fabrica(){
    int i;
    Empleado e;

    // espera a que estén todos los trabajadores
    P(iniciarTrabajo);

    // le avisa a cada uno de los trabajadores para iniciar su trabajo
    for (i=0; i<E; i++) {
        V(esperaInicio[i]);
    }

    // fábrica pone piezas en una cola? o solo se toman?
    // espera a que terminen todos
    P(finJornada);

    // determina el empleado que más piezas hizo
    // revisar:
    // podría ser
    // empMaxPiezas = maxPiezas(cantP);,,
    empMaxPiezas = cantP.indexOf(cantP.max());

}
```

## Ejercicio 10
A una cerealera van T camiones a descargarse trigo y M camiones a descargar maíz. Sólo hay lugar para que 7 camiones a la vez descarguen, pero no pueden ser más de 5 del mismo tipo de cereal.
a. Implemente una solución que use un proceso extra que actúe como coordinador entre los camiones. El coordinador debe atender a los camiones según el orden de llegada. Además, debe retirarse cuando todos los camiones han descargado.

```c
sem hayCamion = 0;
sem descarga[T+M] = ([T+M] 0); // semáforo privado para descarga de cada camion

sem trigo = 5;
sem maiz = 5;
sem mutexCola = 1; // tiene lugar para 7
sem hayEspacio = 7; // para contar el lugar

int trigo = T;
int maiz = M;
queue cola;

process CamionTrigo[id: 0..T-1] {

    P(hayEspacio);
    P(trigo);
    
    P(mutexCola);
    push(cola,(id, "Trigo"));
    V(mutexCola);

    V(hayCamion);

    P(descarga[id]);
    descargar();

}

process CamionMaiz[id: T..M-1] {

    P(hayEspacio);
    P(maiz);
    P(mutexCola);
    push(cola,(id, "Maiz"));
    V(mutexCola);

    V(hayCamion);

    P(descarga[id]);
    descargar();
}

process Coordinador{
    aux = true;
    int id;
    string tipo;
    int cant[2] = ([2] 0); // para contar la cantidad de camiones de cada tipo

    // chequear mutex de las variables
    // podrían ser del coordinador solo
    while (aux) {
        // chequea que hayan descargado todos los camiones
        if (cant[0] == T) and (cant[1] == M) {
            aux = false;
        } else {
            // espera que haya camion
            P(hayCamion);

            P(mutexCola);
            pop(cola(id,tipo));
            V(mutexCola);

            // dar permiso al camión para que descargue
            V(descarga[id]);
            // aumenta en 1 la cantidad de camiones
            // del tipo correspondiente
            if (tipo() == "Trigo") {
                cant[0]++;
                V(trigo);
            } else {
                cant[1]++;
                V(maiz);
            }
        }
    }
}
```

b. Implemente una solución que no use procesos adicionales (sólo camiones). No importa el orden de llegada para descargar. Nota: maximice la concurrencia.

```c
sem hayEspacio = 7;
sem mutexCola = 1;
sem trigo = 5;
sem maiz = 5;



process CamionTrigo[id: 0..T-1] {

    P(hayEspacio);
    P(trigo);
    
    P(mutexCola);
    push(cola,(id, "Trigo"));
    V(mutexCola);

    descargar();
    V(trigo);

}

process CamionMaiz[id: 0..M-1] {

    P(hayEspacio);
    P(maiz);

    P(mutexCola);
    push(cola,(id, "Maiz"));
    V(mutexCola);

    descargar();
    V(maiz);

}
```

## Ejercicio 11
En un vacunatorio hay un empleado de salud para vacunar a 50 personas. El empleado de salud atiende a las personas de acuerdo con el orden de llegada y de a 5 personas a la vez. Es decir, que cuando está libre debe esperar a que haya al menos 5 personas esperando, luego vacuna a las 5 primeras personas, y al terminar las deja ir para esperar por otras 5. Cuando ha atendido a las 50 personas el empleado de salud se retira. Nota: todos los procesos deben terminar su ejecución; suponga que el empleado tienen una función VacunarPersona() que simula que el empleado está vacunando a UNA persona.

```c

sem atendido[50] = ([50] 0); // señalización

sem mutexCola = 1;
sem mutexLlegada = 1;
sem libre = 1;

int cantP = 0;
queue cola;


Process Persona[id: 0..49] {

    P(libre); // espera que el empleado este libre

    P(mutexLLegada); 
    cantP++;
    if (cantP == 1) P(libre); // reserva al empleado

    if (cantP == 5) {
        V(somosCinco);
        cantP = 0; // reinicia contador
    }
    
    P(mutexCola);
    push(cola, id);
    V(mutexCola);

    V(mutexLLegada);

    P(atendido[i]);

}

Process Empleado {
    int i, j, id;

    for (i=0; i<10; i++) {
        P(somosCinco);

        for (j=0; j<5; j++) {
            P(mutexCola);
            pop(cola, id);
            V(mutexCola);

            vacunarPersona(id);
            V(atendido[id]);
        }

        V(libre); // señaliza que está libre
    }
    retirarse();

}
```