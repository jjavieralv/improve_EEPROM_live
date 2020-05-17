# Extend EEPROM life 
EEPROM (Electrically Erasable Programmable Read-Only Memory) is a type of non-volatile memory that allows storing values between system reboots or power supply lost. 

## EEPROM Features
An EEPROM has the next characteristics:
- Divided into parts called cells, usually of 1 byte
- Slow write process (1byte 3.3 ms)
- Fast read process (1024 Bytes 0.3ms)
- Write process decreases memory cell lifetime (around 100.000 write access for each cell)
- Read access no decreases memory cell lifetime

A fast view of characteristics is enough to see that there is a big problem useful life cell. If a cell has been written a lot of times, data stored here could be corrupted. Due to that, some "algorithms" have been developed to try to solve the life problem. They are called **wear-leveling algorithms**. 

## Actual problems and solutions 
Usually for EEPROM memories the method is to use half memory space with a counter, increase the counter each time that there is a write. Once the counter has reached a specified limit, the first half of memory is marked as "unusable" and the data is copied to the other half and the counter continues incrementing in the same way until it reaches a new marked limit. Some problems are implementing this method: there is only using half memory at a time; if only a bunch of cells are rewritten the counter increments likewise; not sure that all the cells have been rewritten approximately the same times. 

## Extend EEPROM life algorithm
I propose an alternative method to improve EEPROM wear-leveling. It consists of group cells creating packages. Each package has a state indicator and an identification number. Also, in the case that the state indicator does not meet certain conditions specified below, there is a counter variable that indicates how much memory lifetime has been consumed. This method allows the use of cells to be more even. This algorithm is designed 
