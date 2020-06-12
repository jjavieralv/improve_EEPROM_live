# Extender la vida de las memorias EEPROM
EEPROM (Electrically Erasable Programmable Read-Only Memory) es un tipo de memoria no volatil (no se pierden aunque falte alimentación o el sistema se reinicie) que permite almacenar datos.


## Caracteristicas de EEPROM
Cabe destacar las siguientes caracteristicas de este tipo de memorias:
	- Estan compuestas por partes llamadas células, usualmente de 1 byte
	- Escritura muy lento (1byte 3.3ms)
	- El proceso de escritura daña las células reduciendo su vida útil (entre 100k y 1M aproximadamente)
	- Lectura rapida (1024 bytes 0.3ms)
	- El proceso de lectura no daña a las células
De un vistazo a las características podemos ver que es un problema que la escritura reduzca la vida útil. Si una célula ha sido reescrita un gran numero de veces, los datos almacenados en esta corren el riesgo de corromperse. Debido a esto se han desarrolado diversos algoritmos para evitar trabajar con datos corruptos. Estos algoritmos son llamados **wear-leveling algorithms**. 


## Soluciones y fallos actuales al problema
1.  Dividir la memoria en partes, usar una hasta la saciedad y pasar a la siguiente

	Usualmente para las memorias EEPROM este problema se soluciona dividiendo la memoria en bloques grandes, supongamos que se divide por la mitad. Se empieza a utilizar la primera mitad creando en una parte de esta un contador. Ese contador se incrementa cada vez que se realiza una escritura. Cuando el contador alcanza un valor en el que no se puede garantizar la integridad de los valores almacenados (como hemos dicho antes 100k aproximadamente), el bloque entero pasa a ser inservible, se copian todos los datos de la primera mitad a la segunda y se inicia en esta un nuevo contador. Cuando este contador, al igual que con el anterior, alcanza un valor en el que no se pueden garantizar la integridad de los datos almacenados, el bloque pasa a ser considerado inservible. Ya que hemos usado las 2 mitades, la memoria pasa a ser considerada como inservible o imposible de asegurar su integridad. Esto falla en:

        a. No se puede usar toda la memoria a la vez
        b. No se hacen escrituras de manera uniformes dentro del mismo bloque
        c. No se hacen escrituras de manera uniforme en toda la memoria
        d. Puede haber celulas que ni siquiera se hayan escrito en un bloque marcado como "gastado"

2.  Uso de matrices para valores incrementales.

	Se usa cuando tan solo se quiere tener un contador que incrementa. Cada vez que se quiere actualizar el contador, el valor se guarda en la siguiente posición del array. Cuando el sistema se inicia, recorre el array y toma el valor mayor. 
	Explicado en el siguiente [enlace](https://embeddedgurus.com/stack-overflow/2017/07/eeprom-wear-leveling/)
	
## Mi propuesta de algoritmo para aumentar la vida de las EEPROM

La alternativa que propongo consite en que los datos no estarán en una posición fija, si no que siempre que se quiera hacer una escritura se hará sobre el espacio que tenga menos escrituras realizadas. Además se implementarán técnicas ya conocidas como la comparación del valor a escribir con el escrito para evitar reescrituras innecesarias

### Estructura de la memoria
Para ello hay que dividir la memoria en estructuras. Las estructuras tienen que tener el mismo tamaño, van a estar compuestas por 3 partes:

    a. La primera va a ser un ID único para esa estructura
    b. La segunda va a ser un indicador de estado
    c. La tercera va a ser el espacio en el que van a ir los datos

Además, las primeras direcciones de memoria pasan a estar reservadas para información de la memoria:

- **ra**: 5 Primeros bytes de la memoria. El primer byte tendrá la secuencia (0100 1010‬) en caso de que la memoria esté estructurada. Los otros 4 bytes indican el número de células usadas para los apartados rb,rc,rd,re
- **rb**: Valor máximo del indicador de estado (con este valor tambien podemos el número de células reservadas para el indicador de estado en cada estructura). Siempre tiene que ser un número impar
- **rc**: Veces que el indicador de estado se ha reiniciado
- **rd**: Tamaño reservado para el ID en cada estructura
- **re**: Tamaño reservado para los datos en cada estructura


#### Organización de estas estructuras
Una vez que ya tenemos claro cómo va a estar estructurada la memoria, paso a explicar como van a estar gestionadas estas estructuras. Por comodidad y facilidad de explición, se van a usar células completas, aunque en muchos casos se pueda optimizar usando un tamaño menor. En el ejemplo se ha supuesto que la memoria ya estaba estructurada. Más tarde se indicará como inicializar la memoria en caso de que no lo esté.

#### Primero vamos a ver el tamaño de cada parte

Para poder hacer esto primero vamos a aclarar algunas variable fijas. 
Los tamaños van a ser de ejemplo.

Tamaño memoria:

- Tamaño de la memoria EEPROM **(TM)**: 1024 Bytes
- Tamaño reservado para información **(TRI)**: ra + rb + rc + rd = 9 Bytes
- Tamaño usable para estructuras **(TU)**: TM - TRI = 1015 Bytes

Tamaño espacio reservado:

- **ra** : 5 Bytes
- **rb** : 1 Byte. Siempre tiene que ser un número impar, por lo que va a acabar en 1.
- **rc** : 1 Byte 
- **rd** : 1 Byte
- **re** : 1 Bytes 

Ejemplo de composición del espacio reservado con los parámetros anteriores:

| **ra** | **rb** | **rc** | **rd** | **re** |
| -- | -- | -- | -- | -- |
| 01001010‬ 00000001 00000001 00000001 00000001| 00000011 | 00000000 | ‭01100100‬ | 00001000 |

Tamaño de las estructuras:

- Tamaño estructura individual **(TE)**: valor(rd) + valor(ra[1 ])valor(re)
- Tamaño de la Estructura asignado al ID **(TEID)**: Para hacer esto hacemos los siguientes cálculos incrementando la i hasta que la condición se cumpla: 
    - TU / ( i + TEE + TED ) <= 2 ^ (i*8)
- Tamaño de la Estructura asignado a Estado **(TEE)**: Igual a ra
- Tamaño de la Estructura asignado a Datos **(TED)**: 8 Bytes

| Nombre de la columna| Posicion en la estructura | Tamaño |
| -- | -- | -- |
| a. ID de la estructura | Primera |  |
| b. ID de estado | Segunda | |
| c. Tamaño para datos | Tercera | 8Bytes