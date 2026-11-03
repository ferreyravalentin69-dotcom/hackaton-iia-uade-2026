# <T27> · <¿Paramos en boxes en esta vuelta?>

> **Hackatón IIA · UADE · 3.4.257.** Este README es la plantilla del grupo. En su fork, reemplacen todo lo que está entre `< >` y borren esta nota. Las bases están en `BASES.md` y su ficha en `problematicas/`.

**Integrantes:** <Valentin Ferreyra Gutierrez> · <Nicolas Quiroga>
**IA declarada:** <claude>
**Pista:** <Reglas>
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** <Para cada caso, ¿SÍ o NO? SÍ = parar en esta vuelta hace ganar la posición al auto de adelante.>

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | <Maximizar las posiciones ganadas (undercuts exitosos) minimizando el riesgo de falsos positivos (perder tiempo detrás de autos lentos) y falsos negativos (que el rival te pase).> |
| Entorno | <El circuito de Fórmula 1.> |
| Actuadores | < El auto de F1 (motor, neumáticos, caja de cambios) y la comunicación por radio al piloto.> |
| Sensores | <Telemetría del auto (degradación de neumáticos), sistemas de cronometraje (distancia con el rival) y datos de estrategia (vueltas restantes).> |

**Objetivo y métrica.** <Qué es lo que de verdad importa en su problema y con qué métrica lo van a medir. ¿Son lo mismo?>

## 2. El sistema

**Cómo decide.** < 1.SI Distancia al auto de adelante > 3.0s ENTONCES NO parar. (El undercut es matemáticamente muy difícil de lograr, incluso con parada rápida; casos 6, 9 y 12 confirman esto). 2.SI Degradación del neumático < 2.0 décimas ENTONCES NO parar. (El neumático tiene vida útil; parar es tirar tiempo a la basura; casos 1, 2, 4 y 11 confirman esto). 3.SI Vueltas restantes < 10 Y Degradación < 4.0 ENTONCES NO parar. (Evita paradas innecesarias al final de la carrera si el neumático no está destruido; caso 4). 4.SI Degradación ≥ 4.0 Y Distancia ≤ 2.0s ENTONCES SÍ parar. (Escenario ideal de undercut: el rival sufre mucho y está cerca; casos 5, 8 y 10). 5.SI Distancia ≤ 0.5s Y Vueltas restantes ≤ 25 ENTONCES SÍ parar. (Ataque directo en pista; el rival está a tiro y la ventana de undercut es óptima; caso 7).6.SI Degradación ≥ 2.5 Y Distancia ≤ 3.0s ENTONCES SÍ parar. (Regla de cierre para casos límite como el 3, donde la degradación justifica el intento aunque la distancia no sea mínima). 7.EN CASO CONTRARIO NO parar. (Política conservadora por defecto para no perder tiempo).>

**Variables que usa y por qué.** <Distancia al auto de adelante (s): Es la variable más crítica. Si la distancia es mayor a 3 segundos, el undercut es inútil porque el rival tiene margen de reacción.Degradación del neumático (décimas por vuelta): Define la ventaja de ritmo que tendrá el auto al salir con neumáticos nuevos. Si la degradación es baja (<2.0), no hay ganancia real.Vueltas restantes (cantidad): Se usa como filtro de contexto (Regla 3). Si quedan muy pocas vueltas y el neumático no está destruido, no hay tiempo material para recuperar el tiempo perdido en boxes.Duración de la parada (s): Se descarta como variable de decisión principal. El dato de contexto indica que el equipo premia las paradas rápidas, lo que genera un sesgo peligroso. Una parada de 2.4s (caso 7) es buena, pero una de 2.9s (caso 1) es irrelevante si la degradación es baja. El sistema asume una duración de parada estándar (~2.5s) para el cálculo mental, pero no permite que una parada rápida obligue a parar si las reglas 1, 2 o 3 fallan.Conjunto (Entrenamiento/Validación): Se descarta para la decisión en vivo; solo se usa para calibrar y probar las reglas.>

**Cómo se corre en vivo.** <Entrada de datos: El ingeniero recibe por telemetría los valores de la vuelta actual (Degradación, Distancia, Vueltas restantes y Duración estimada de parada).Evaluación secuencial: Se aplican las reglas del 1 al 7 en orden. La primera regla que se cumpla dicta la acción (SÍ o NO). Las reglas están diseñadas para no ser contradictorias.Ejecución en Colab/Planilla: En la planilla de cálculo (o script de Python en Colab), se usan funciones IF anidadas o np.where con las condiciones exactas.
Salida: El sistema devuelve un "SÍ" (Boxes) o "NO" (Stay Out) inmediato, priorizando la seguridad de no perder posiciones por falsos positivos.>

**Dos casos de ejemplo.**

| Caso inventado | Valores | Salida del sistema |
|---|---|---|
| A | <Datos: Degradación = 5.1, Distancia = 0.9s, Vueltas restantes = 40, Parada = 2.8s.Evaluación: Regla 1 (0.9 > 3.0) no. Regla 2 (5.1 < 2.0) no. Regla 3 (40 < 10) no. Regla 4: Degradación (5.1) ≥ 4.0 Y Distancia (0.9) ≤ 2.0.> | <SÍ parar. El resultado real fue 1 (éxito).> |
| B | <Datos: Degradación = 4.1, Distancia = 4.1s, Vueltas restantes = 22, Parada = 3.9s.Evaluación: Regla 1: Distancia (4.1s) > 3.0s.> | <NO parar. Aunque la degradación es alta (4.1), la distancia de 4.1s hace que el undercut sea inútil (el rival tiene más de 4 segundos de margen y la parada tarda 3.9s). El resultado real fue 0 (acertamos)> |

## 3. Medición

**Entrenamiento (casos 1–14)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = <> | FN = <> |
| **Real NO** | FP = <> | VN = <> |

**Prueba (casos 15–20)**

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = <> | FN = <> |
| **Real NO** | FP = <> | VN = <> |

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | <> | <> |
| Precisión | <> | <> |
| Sensibilidad | <> | <> |
| Especificidad | <> | <> |

**Entrenamiento contra prueba.** <¿Hay diferencia? ¿Por qué? ¿Sobreajuste o generalización?>

## 4. El error más caro

<¿Es peor el falso positivo o el falso negativo en su problema? ¿Por qué, según la ficha? ¿Qué cambiaron en el sistema para cometer menos ese error?>

## 5. ¿Comparado contra qué?

**Línea base trivial.** <Qué resultado saca un sistema que responde siempre lo mismo, y cómo se compara el suyo.>

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | <> |
| ¿Qué métrica? | <> |
| ¿Comparado contra qué? | <> |
| ¿Quién lo midió? | <> |

## 6. Riesgo: el camino completo

<Datos → modelo → decisión → consecuencia aplicado a su caso. Al menos un riesgo concreto.>

## 7. Bitácora

**Uso de la IA**

| # | Qué le pedimos | Qué aceptamos | Qué rechazamos y por qué |
|---|---|---|---|
| 1 | <> | <> | <> |

**Corridas contra el evaluador**

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | <> | <> | <> |
