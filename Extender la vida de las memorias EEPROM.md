# Extender la vida de las memorias EEPROM


## 1. Presentación
Lo primero será poner en situación dando un breve resumen de que son estas memorias, cuáles son sus carácterísticas, soluciones actuales, etc. Con el fin de lograr una mayor comprensión. 

## 1.1 ¿Qué son las memorias EEPROM?
EEPROM (Electrically Erasable Programmable Read-Only Memory) es un tipo de memoria no volatil (no se pierden aunque falte alimentación o el sistema se reinicie) que permite almacenar datos. Este tipo de memorias son ámpliamente utilizadas en electrónica debido a su bajo coste y utilidad. 

## 1.2 Caracteristicas de EEPROM
Cabe destacar las siguientes caracteristicas de este tipo de memorias:
	- Estan compuestas por partes llamadas células, usualmente de 1 byte
	- Escritura muy lento (1byte 3.3ms)
	- El proceso de escritura daña las células reduciendo su vida útil (entre 100k y 1M aproximadamente)
	- Lectura rapida (1024 bytes 0.3ms)
	- El proceso de lectura no daña a las células
De un vistazo a las características podemos ver que es un problema que la escritura reduzca la vida útil. Si una célula ha sido reescrita un gran numero de veces, los datos almacenados en esta corren el riesgo de corromperse. Debido a esto se han desarrolado diversos algoritmos para evitar trabajar con datos corruptos. Estos algoritmos son llamados **wear-leveling algorithms**. 
Se va a considerar que las EEPROM vienen con todos sus bits a 0 para facilitar las explicaciones.


## 1.3 Soluciones y fallos actuales al problema
1.  Dividir la memoria en partes, usar una hasta la saciedad y pasar a la siguiente

	Usualmente para las memorias EEPROM este problema se soluciona dividiendo la memoria en bloques grandes, supongamos que se divide por la mitad(como se suele hace usualmente). Se empieza a utilizar la primera mitad creando en una parte de esta un contador. Ese contador se incrementa cada vez que se realiza una escritura. Cuando el contador alcanza un valor en el que no se puede garantizar la integridad de los valores almacenados (como hemos dicho antes 100k aproximadamente), el bloque entero pasa a ser inservible, se copian todos los datos de la primera mitad a la segunda y se inicia en esta un nuevo contador. Cuando este contador, al igual que con el anterior, alcanza un valor en el que no se pueden garantizar la integridad de los datos almacenados, el bloque pasa a ser considerado inservible. Ya que hemos usado las 2 mitades, la memoria pasa a ser considerada como inservible o imposible de asegurar su integridad. Esto falla en:

        a. No se puede usar toda la memoria a la vez
        b. No se hacen escrituras de manera uniformes dentro del mismo bloque
        c. No se hacen escrituras de manera uniforme en toda la memoria
        d. Puede haber celulas que ni siquiera se hayan escrito en un bloque marcado como "gastado"

2.  Uso de matrices para valores incrementales.

	Se usa cuando tan solo se quiere tener un contador que incrementa. Cada vez que se quiere actualizar el contador, el valor se guarda en la siguiente posición del array. Cuando el sistema se inicia, recorre el array y toma el valor mayor. 
	Explicado en el siguiente [enlace](https://embeddedgurus.com/stack-overflow/2017/07/eeprom-wear-leveling/)
	
# 2. Mi propuesta de algoritmo para aumentar la vida de las EEPROM

La alternativa que propongo consite en que los datos no estarán en una  posición fija, si no que siempre que se quiera hacer una escritura se hará sobre el espacio que tenga menos escrituras realizadas. El usuario no se debe preocupar de la organización interna, ya que será el programa el que realice todo el proceso. Además se implementarán técnicas ya conocidas como la comparación del valor a escribir con el escrito para evitar reescrituras innecesarias a nivel de bit. 

## 2.1 Organizacion de la memoria
Tras ser inicializada, la memoria va a estar compuesta por 3 partes:

    a. Información de la organización de la memoria
    b. Segmentos para almacenar información
    c. Espacio vacío

### 2.1.1 Información de la organización de la memoria
Las primeras direcciones de memoria pasan a estar reservadas para información de la memoria:

- **ra**: 5 Primeros bytes de la memoria. El primer byte tendrá la secuencia (0100 1010‬) en caso de que la memoria esté estructurada. Los otros 4 bytes indican el número de células usadas para los apartados rb,rc,rd,re
- **rb**: Valor máximo del indicador de estado. Siempre tiene que ser un número impar
- **rc**: Veces que el indicador de estado se ha reiniciado
- **rd**: Cantidad máxima posible de estructuras(maximo valor de ID)
- **re**: Tamaño reservado para almacenar los datos en cada estructura

### 2.1.2 Segmentos de la memoria
Hay que dividir la memoria en segmentos del tamaño de las estructuras, pudiendo cada segmento contener o no una estructura. Cada segmento va a tener un indicador de estado al comienzo, este parte tendrá un tamaño indicado en valor(ra[1]). Cada segmento puede contener una estructura, la cual esta compuesta por:

    a. Un ID único para esa estructura 
    b. Un espacio en el que van a ir los datos
Por lo tanto cada segmento que contenga una estructura va a tener el formato

| Indicador de estado | ID estrucutra | Espacio para datos |
| -- | -- | -- |

#### 2.1.2.1 Indicador de estado
Este valor indica el estado en el que se encuentra ese segmento de la memoria. El estado nos indica no solo si ese segmento de la memoria contiene información válida o inválida, si no tambien cuántas escrituras se han hecho sobre esa parte con el fin de que el número de escrituras sea lo más uniforme posible a lo largo de toda la memoria. El valor máximo del indicador de estado se almacena en **rb**, pudiendo tomar todos los valores entre 0 y este valor, pero siempre siendo un valor impar para lograr un número par de estados. Los estados impares indican que esa parte esta ocupada por información válida, mientras que los valores impares indican que esa parte esta ocupada por información inválida. Además los estados son cíclicos, lo que significa que una vez que se alcanza el último estado, se vuelve al primero, pero se incrementa el contador de vueltas **rc** en 1. Por ejemplo, si **rb** vale 3, los estados podrán ser 0,1,2,3. El estado 0 y 2 indicarían que ese segmento puede ser sobreescrito, mientras que el 1 y el 3 indicarían que ese segmento contiene datos válidos, y por lo tanto no deber ser sobreescrito. Tenemos que recordar que el indicador de estado se actualiza tanto cuando se escribe un dato como cuando se "libera" el segmento, por lo tanto cuando hayamos recorrido en el ejemplo los 4 estados, habremos actualizado el valor 2 veces, pero hemos atacado a la memoria escribiendo 4 veces.

#### 2.1.2.2 ID. Identificador de estructura
Cada estructura que creemos va a tener asignado un valor que será usado como identificador único de esa estructura. Estos valores van a ser asingados de manera incremental en 1, desde el 0 hasta el máximo indicado (almacenado en **rd**). Las estructuras no tienen por que estar ordenadas en la memoria, pero cada estructura siempre será identificable gracias a su identificador único.

#### 2.1.2.3 Espacio para datos
Cada estructura contiene un número de celdas, idicado en el valor **re**, que será usado para almacenar los datos. En caso de que se quieran almacenar datos de una longitud mayor a la capacidad de almacenamiento de una éstructura no habrá problema, ya que se pueden ir creando estructuras de manera consecutiva para almacenar ese dato. 

### 2.1.3 Espacio vacio
Es el espacio que aún no ha sido utilizado por ninguna estructura. Aún asi, lo interpretaremos como si estuviese dividido en segmentos, y puesto que esos segmentos no se han utilizado nunca, su indicador de estado valdrá 0. 

## 2.2 Organización de estas estructuras
Una vez que ya tenemos claro cómo va a estar estructurada la memoria, paso a explicar como van a estar gestionadas estas estructuras. Por comodidad y facilidad de explición, se van a usar células completas, aunque en muchos casos se pueda optimizar usando un tamaño menor. En el ejemplo se ha supuesto que la memoria ya estaba estructurada. Más tarde se indicará como inicializar la memoria en caso de que no lo esté.

### 2.2.1 Organizacion de la memoria

Para poder hacer esto primero vamos a aclarar algunas variables fijas. 
Los tamaños van a ser de ejemplo.

Tamaño memoria:

- Tamaño de la memoria EEPROM **(TM)**: 1024 Bytes
- Tamaño reservado para información **(TRI)**: ra + rb + rc + rd + re = 9 Bytes
- Tamaño usable para segmentos **(TU)**: TM - TRI = 1015 Bytes

#### 2.2.1.1 Tamaño espacio reservado:

- **ra** : 5 Bytes. Siempre será este valor.
- **rb** : 1 Byte. Valor impar siempre acaba en 1. En el ejemplo toma valor 3, por lo que el estado puede ser(0,1,2,3)
- **rc** : 1 Byte. Nunca se ha reiniciado(valor 0)
- **rd** : 1 Byte. En el ejemplo 60 estructuras
- **re** : 1 Bytes. En el ejemplo 4 células para datos

Ejemplo de composición del espacio reservado con los parámetros anteriores:

| **ra** | **rb** | **rc** | **rd** | **re** |
| -- | -- | -- | -- | -- |
| 01001010‬ 00000001 00000001 00000001 00000001| 00000011 | 00000000 | ‭00111100‬ | 00000100 |

#### 2.2.1.2 Tamaño de las estructuras:

- Tamaño estructura individual(ID+estado+datos) **(TE)**: valor(ra[3]) + valor(ra[1]) + valor (re)

Tan solo podremos crear estructuras diferentes hasta llenar la memoria. Sabiendo el tamaño de la memoria, el tamaño reservado para la información al principio y el tamaño de cada estructura. Podremos saber cuántas estructuras diferentes podermos crear. Por simplificar, vamos a hacer la cuenta usando células completas. Para hacer esto hacemos los siguientes cálculos incrementando la i hasta que la condición se cumpla con el mayor valor de i: 
    - **TU** / (i + valor(ra[1]) + valor (re)) >=1

Ejemplo de composición de las estructuras individuales(ejemplo primera estructura escrita):

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 00000000 00000000 00000000 00000000 |

## 2.3 Explicación de cómo funciona el algoritmo
Ahora que ya tenemos claras las partes de las que va a estar compuesta la memoria, y el significado de cada una, procedo a explicar cómo funciona el algoritmo. 

Como ya he indicado antes, las escrituras se realizarán siempre sobre el segmento en el que menos escrituras se han realizado, sea esta la escritura de un nuevo valor o la actualización de un valor ya existente. Para esto usaremos el indicador de estado, que nos permite conocer tanto si un segmento tiene valores válidos como cuántas veces ha sido escrito. Tan solo tendremos que cumplir 2 condiciones a la hora de realizar una escritura:

    a. La memoria se recorre de arriba(inicio de memoria) a abajor(fin de memoria) 
    b. Escribir en el segmento que lleve más tiempo sin usarse. 

Como ejemplo de la condición b, pongamos que tenemos 4 estados(0,1,2,3):
    
    -Nunca escribimos en un 2 si tenemos un 0 por debajo
    -Nunca escribimos en un 0 si tenemos un 2 por debajo(esto sucede cuando se reinicia el contador de estados)

Además, a la hora de actualizar un valor, primero se escribe el nuevo y después se marca el antiguo como inútil. Gracias a esto en caso de falla, podemos mantener el valor antiguo y si falla a la hora de marcar el segmento antiguo como inservible, podrémos detectarlo fácilmente ya que 2 estructuras con el mismo ID aparecen como valores válidos. Nos quedaríamos con la más actual.


Voy a proceder con un ejemplo de memoria inicializada. A continuación indico cómo quedarian los primeras direcciones de una memoria inicializada en la que aún no se ha escrito ninguna estructura. Supongamos que esta memoria solo tiene espacio para la informacion de la memoria y 4 segmentos, indicados más adelante.

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |

Voy a indicar cómo irían cambiando los segmentos.
Ahora imaginemos que queremos guardar un valor nuevo en memoria:

Puesto que no hay ninguna estructura creada, creamos una en el primer segmento disponible, cumpliendo así con las condiciones a y b. Para esto le asignamos su ID correspondiente, y actualizamos el valor del indicador de estado, además del dato que queremos guardar, que será por ejemplo 2^32-1 (todos bits 1). Quedando en el segmento:
    
    -Indicador de estado pasa a 1 ya que tiene información válida
    -El ID de la primera estructura es 0
    -Los datos se ponen en su valor actualizado

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |

Ahora imaginemos que queremos escribir otro dato, por ejemplo el valor 2^31 -1(todo 1 menos el primer bit). Para ello tenemos que buscar el siguiente segmento con menos escrituras, asignarle el siguiente valor al ID y actualizar el indicador de estado de ese segmento.

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000001 | 00000001 | 01111111 11111111 11111111 11111111 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |

Volvemos a repetir guardando un nuevo valor, por ejemplo el valor 2^30 -1. Repitiendo todo el proceso.

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000001 | 00000001 | 01111111 11111111 11111111 11111111 |
| 00000001 | 00000002 | 00111111 11111111 11111111 11111111 |
| 00000000 | 00000000 | 00000000 00000000 00000000 00000000 |

Ahora imaginemos que queremos actualizar el segundo valor a otro número, por ejemplo 2^31 -2. Para esto, escribimos el valor en el siguiente segmento con menos escrituras, actualizamos su indicador de estado, mantenemos el mismo ID de estructura ya que es una actualización, y , finalmente, actualizamos el indicador de estado del segmento antiguo para indicar que contiene datos inválidos

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000010 | 00000001 | 01111111 11111111 11111111 11111111 |
| 00000001 | 00000010 | 00111111 11111111 11111111 11111111 |
| 00000001 | 00000001 | 01111111 11111111 11111111 11111110 |

Como podemos ver, en lugar de sobreescribirse machacando el segmento en el que se encontraba actualmente, hemos escrito el valor en el segmento que tenía menos escrituras realizadas.

Ahora imaginemos que queremos actualizar el valor que esta contenido en la estructura con ID 2(00000010) a un valor 2^30 -2.  Para ello buscamos el segmento libre con menos escrituras hacia abajo. Como no queda ninguno libre, continuamos la busqueda desde el principio de la memoria. Aquí vemos que el 3º segmento tiene un indicador de estado par, y, por lo tanto, podemos sobreescribirlo. Por lo que actualizamos sus valores y marcamos el segmento anterior como libre para sobreescribirse, actualizando el valor de su indicador de estado.

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000011 | 00000010 | 00111111 11111111 11111111 11111110 |
| 00000010 | 00000010 | 00111111 11111111 11111111 11111111 |
| 00000001 | 00000001 | 01111111 11111111 11111111 11111110 |

Ahora imaginemos que queremos añadir un nuevo valor, para ello buscaremos el segmento libre que tenga menor numero de escrituras. En este caso solo tenemos libre el 3º segmento, así que será ahi donde creemos la nueva estructura. Al igual que en los casos anteriores, la nueva estructura tendrá su identificador único, en este caso 3. Para los datos guardaremos por ejemplo el valor 2^29 -1. Igual que hay que incrementar el contador de estado de ese segmento. 

| **Indicador de Estado** | **ID** | **Datos** |
| -- | -- | -- |
| 00000001 | 00000000 | 11111111 11111111 11111111 11111111 |
| 00000011 | 00000010 | 00111111 11111111 11111111 11111110 |
| 00000011 | 00000011 | 00011111 11111111 11111111 11111111 |
| 00000001 | 00000001 | 01111111 11111111 11111111 11111110 |

Como podemos ver, todas las estructuras tienen su ID único, por lo que se puede usar como una memoria tradicional, pero con la ventaja que que se han ido repartiendo las escrituras a lo largo de la memoria en lugar de machacar todo el rato las mismas posiciones. 

Este saca su mayor potencial si el número máximo de estructuras es menor al número máximo de segmentos disponibles. 

## 2.4 Ejemplos de mejora frente a sistemas tradicionales.
Imaginemos que tenemos una memoria EEPROM, con capacidad para 1000 Bytes y que a partir de las 100.000 los datos pueden corromperse. Vamos a ver una comparativa de los 2 métodos en diferentes circunstancias, junto con un resumen final. 

### 2.4.1 Escribiendo pocos valores
Muchas veces estas memorias se utilizan para guardar tan solo un puñado de valores entre reinicios. Por ello vamos a hacer la cuenta por ejemplo para escribir un valor tipo int(4Bytes). 

#### 2.4.1.1 ANTIGUO: Dividiendo la memoria en 3 partes.
En este ejemplo vamos a dividir la memoria en 3 partes, cada una de ellas con un contador. Una vez que el contador de una parte alcance el valor 100.000, pasaremos al siguiente bloque. Dejándonos por lo tanto:

3 Bytes por cada contador (para poder contar hasta 100.000), por lo que el sistema usa 9 bytes en total de la memoria.

La memoria restante será la que podamos usar, en total 991 Bytes. Al ser 3 bloques, tenemos que dividir este valor entre 3. Por lo que cada bloque tendrá 330Bytes. Así que es como considerar que tenemos una memoria de 330Bytes. 

Si escribimos un valor, por ejemplo un int(4 Bytes) y lo reescribimos constantemente, (lees y escribes siempre un valor de memoria fijada) podremos escribir este valor 100.000 veces. Pasamos al bloque 2, escribimos otras 100.000. Pasamos al bloque 3, otras 100.000 escrituras. Por lo tanto podemos escribir 1 valor int 300.000 veces hasta que la memoria quede inútil

#### 2.4.1.2 NUEVO: Actualización dinámica de memoria
Para que sea más fácil comparar, voy a hacer las cuentas dejando para tener el mismo espacio de datos que permitía el ejemplo anterior 330Bytes.

Ahora lo primero, inicializamos los valores de la memoria, puesto que nosotros podemos elegir el tamaño para los datos. Vamos a asignar que sean estructuras con una capacidad de datos de 4Bytes. 

Como queremos una comparación más justa, si nuestras estructuras permiten almacenar 4 Bytes, necesitaremos 83 para poder almacenar los 330Bytes del ejemplo anterior, pero puesto que solo estamos actualizando un valor, tan solo habrá 1 estructura. 

Para no complicar las cosas, usaremos un indicador de estados de 1 Byte, permitiendonos tener 256 estados (0...255). Puesto que buscamos llegar a las 100.000 escrituras, pasaremos por los 256 estados en un total de 390 vueltas, por lo que necesitaremos 2 Bytes para indicar el número de vueltas que hemos hecho. Por lo que las información de la memoria quedaría:

| **ra** | **rb** | **rc** | **rd** | **re** |
| -- | -- | -- | -- | -- |
| 01001010‬ 00000001 00000010 00000001 00000001| 11111111 | 00000000 00000000 | 01010011 | 00000100 |

Con esto gastamos un total de 10 Bytes. Dejando 990 Bytes para segmentos. Estos segmentos tienen que poder contener: 1Byte para el indicador de estado, 1Byte para el ID de estructura y 4Bytes para datos. Por lo tanto cada segmento son 6 Bytes. Puesto que tenemos 990 Bytes, podemos hacer 165 segmentos (990/6). 

Cada vez que damos una vuelta a los 256 estados, es por que en ese segmento hemos escrito el valor 128 veces (ya que las otras 128 ha sido para "liberar el espacio"). Como consideramos que solo se ha reescrito un valor, podemos hacer las cuentas.

Usamos 1 estructura, que puede ir pasando por 165 segmentos, en cada segmento se puede escribir 128 veces por vuelta y para cumplir las 100.000 escrituras tenemos que dar 390 vueltas. Por lo tanto:

1*165*128*390= 8.236.800

#### 2.4.1.3 Resumen escribiendo 1 valor
| Método | Nº valores guardados | Tamaño de esos valores | Escrituras hasta inutil |
| -- | -- | -- | -- |
| Antiguo | 1 | 4Bytes | 300.000 |
| Nuevo | 1 | 4Bytes | 8.236.800 |

Escribiendo un valor tenemos una mejora del 2745,6%. 
Para pocos valores almacenados el nuevo método presenta una increible mejora.










