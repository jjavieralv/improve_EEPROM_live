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

## 2.1.1 Información de la organización de la memoria
Las primeras direcciones de memoria pasan a estar reservadas para información de la memoria:

- **ra**: 5 Primeros bytes de la memoria. El primer byte tendrá la secuencia (0100 1010‬) en caso de que la memoria esté estructurada. Los otros 4 bytes indican el número de células usadas para los apartados rb,rc,rd,re
- **rb**: Valor máximo del indicador de estado. Siempre tiene que ser un número impar
- **rc**: Veces que el indicador de estado se ha reiniciado
- **rd**: Cantidad máxima posible de estructuras(maximo valor de ID)
- **re**: Tamaño reservado para almacenar los datos en cada estructura

## 2.1.2 Segmentos de la memoria
Hay que dividir la memoria en segmentos del tamaño de las estructuras, pudiendo cada segmento contener o no una estructura. Cada segmento va a tener un indicador de estado al comienzo, este parte tendrá un tamaño indicado en valor(ra[1]). Cada segmento puede contener una estructura, la cual esta compuesta por:

    a. Un ID único para esa estructura 
    b. Un espacio en el que van a ir los datos
Por lo tanto cada segmento que contenga una estructura va a tener el formato

| Indicador de estado | ID estrucutra | Espacio para datos |
| -- | -- | -- |

### 2.1.2.1 Indicador de estado
Este valor indica el estado en el que se encuentra ese segmento de la memoria. El estado nos indica no solo si ese segmento de la memoria contiene información válida o inválida, si no tambien cuántas escrituras se han hecho sobre esa parte con el fin de que el número de escrituras sea lo más uniforme posible a lo largo de toda la memoria. El valor máximo del indicador de estado se almacena en **rb**, pudiendo tomar todos los valores entre 0 y este valor, pero siempre siendo un valor impar para lograr un número par de estados. Los estados impares indican que esa parte esta ocupada por información válida, mientras que los valores impares indican que esa parte esta ocupada por información inválida. Además los estados son cíclicos, lo que significa que una vez que se alcanza el último estado, se vuelve al primero, pero se incrementa el contador de vueltas **rc** en 1. Por ejemplo, si **rb** vale 3, los estados podrán ser 0,1,2,3. El estado 0 y 2 indicarían que ese segmento puede ser sobreescrito, mientras que el 1 y el 3 indicarían que ese segmento contiene datos válidos, y por lo tanto no deber ser sobreescrito.

### 2.1.2.2 ID. Identificador de estructura
Cada estructura que creemos va a tener asignado un valor que será usado como identificador único de esa estructura. Estos valores van a ser asingados de manera incremental en 1, desde el 0 hasta el máximo indicado (almacenado en **rd**). Las estructuras no tienen por que estar ordenadas en la memoria, pero cada estructura siempre será identificable gracias a su identificador único.

### 2.1.2.3 Espacio para datos
Cada estructura contiene un número de celdas, idicado en el valor **re**, que será usado para almacenar los datos. En caso de que se quieran almacenar datos de una longitud mayor a la capacidad de almacenamiento de una éstructura no habrá problema, ya que se pueden ir creando estructuras de manera consecutiva para almacenar ese dato. 

## 2.1.3 Espacio vacio
Es el espacio que aún no ha sido utilizado por ninguna estructura. Aún asi, lo interpretaremos como si estuviese dividido en segmentos, y puesto que esos segmentos no se han utilizado nunca, su indicador de estado valdrá 0. 

## 2.2 Organización de estas estructuras
Una vez que ya tenemos claro cómo va a estar estructurada la memoria, paso a explicar como van a estar gestionadas estas estructuras. Por comodidad y facilidad de explición, se van a usar células completas, aunque en muchos casos se pueda optimizar usando un tamaño menor. En el ejemplo se ha supuesto que la memoria ya estaba estructurada. Más tarde se indicará como inicializar la memoria en caso de que no lo esté.

## 2.2.1 Organizacion de la memoria

Para poder hacer esto primero vamos a aclarar algunas variables fijas. 
Los tamaños van a ser de ejemplo.

Tamaño memoria:

- Tamaño de la memoria EEPROM **(TM)**: 1024 Bytes
- Tamaño reservado para información **(TRI)**: ra + rb + rc + rd + re = 9 Bytes
- Tamaño usable para segmentos **(TU)**: TM - TRI = 1015 Bytes

### 2.2.1.1 Tamaño espacio reservado:

- **ra** : 5 Bytes. Siempre será este valor.
- **rb** : 1 Byte. Valor impar siempre acaba en 1. En el ejemplo toma valor 3, por lo que el estado puede ser(0,1,2,3)
- **rc** : 1 Byte. Nunca se ha reiniciado(valor 0)
- **rd** : 1 Byte. En el ejemplo 60 estructuras
- **re** : 1 Bytes. En el ejemplo 4 células para datos

Ejemplo de composición del espacio reservado con los parámetros anteriores:

| **ra** | **rb** | **rc** | **rd** | **re** |
| -- | -- | -- | -- | -- |
| 01001010‬ 00000001 00000001 00000001 00000001| 00000011 | 00000000 | ‭00111100‬ | 00000100 |

### 2.2.1.2 Tamaño de las estructuras:

- Tamaño estructura individual(ID+estado+datos) **(TE)**: valor(ra[3]) + valor(ra[1]) + valor (re)

Tan solo podremos crear estructuras diferentes hasta llenar la memoria. Sabiendo el tamaño de la memoria, el tamaño reservado para la información al principio y el tamaño de cada estructura. Podremos saber cuántas estructuras diferentes podermos crear. Por simplificar, vamos a hacer la cuenta usando células completas. Para hacer esto hacemos los siguientes cálculos incrementando la i hasta que la condición se cumpla con el mayor valor de i: 
    - **TU** / (i + valor(ra[1]) + valor (re)) >=1

Ejemplo de composición de las estructuras individuales(ejemplo primera estructura escrita):

| **ID** | **Indicador de Estado** | **Datos** |
| -- | -- | -- |
| 00000000 | 00000001 | 00000000 00000000 00000000 00000000 |

## 2.3 Explicación de cómo funciona el algoritmo








