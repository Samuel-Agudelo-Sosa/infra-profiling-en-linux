# Análisis

Nombre y código: Samuel Agudelo Sosa, 202459419

## El programa visto desde afuera

Con `evidencia/ps.txt`: Aparecen 4 hilos (TIDs 22981, 22983, 22984, 22985). `ps` reporta un `%CPU` para cada uno de 39.6, 79.6, 103 y 104 respectivamente. Sin embargo, `top` reporta consumos de 99.9%, 36.0%, 0.0% y 0.0% en ese instante. Los números no coinciden porque calculan el consumo de manera diferente: `ps` muestra el promedio de uso de CPU desde que el hilo arrancó, mientras que `top` muestra el uso de CPU instantáneo en el último intervalo de medición (0.5 segundos). Como los hilos terminaban en tiempos escalonados, para cuando `top` midió, algunos hilos ya habían terminado de trabajar. En ese instante, los hilos corrían en los procesadores lógicos (`PSR`) 1, 10, 3 y 8.

## Lo que mostró el perfil

El perfilador Callgrind demostró que el 98.50% de las instrucciones ejecutadas (`Ir`: 609,686,052) se concentran en la función `main`, específicamente en la línea que ejecuta el bucle de multiplicación (`s += a[i*N+k] * b[k*N+j]`), consumiendo el 97.58% del total. Además, `perf stat` respaldó el impacto en memoria reportando 239,243 fallos de caché (`cache-misses`) sobre un total de 64,023,317 referencias a la caché (`cache-references`).

## Los fallos de caché, versión por versión

| Versión | Accesos a datos (`D refs`) | Fallos en D1 | Tasa D1 |
|---|---:|---:|---:|
| lento | 141,902,471 | 67,648,450 | 47.7% |
| rápido | 209,142,409 | 17,055,686 | 8.2% |

Por qué la tasa cambia tanto cuando la cantidad de multiplicaciones es la misma:
En la versión lenta, el bucle más interno recorre las columnas de la matriz B (`k`). Dado que C++ almacena las matrices por filas, avanzar por columnas significa dar saltos largos en la memoria (stride), lo que provoca que los datos cargados en la línea de caché se desperdicien, disparando los fallos D1. Al recorrer la memoria de forma contigua en la versión rápida, se aprovecha al máximo la localidad espacial de la caché.

## Qué se cambió y por qué

Se alteró el orden de los bucles de multiplicación de `i -> j -> k` a `i -> k -> j`. De esta manera, el bucle más profundo avanza sobre `j`. Como las matrices `c` y `b` se acceden mediante `c[i*N+j]` y `b[k*N+j]`, al incrementar `j` se leen y escriben datos que están uno al lado del otro físicamente en la memoria RAM, maximizando los aciertos de caché.

## Resultado

| Versión | Tiempo | Checksum |
|---|---:|---|
| lento | 273 ms | 100662527.125 |
| rápido | 56 ms | 100662527.125 |

Aceleración obtenida: 4.87x veces más rápido (273 / 56).