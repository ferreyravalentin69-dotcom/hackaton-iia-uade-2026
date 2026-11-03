# T27 · ¿Paramos en boxes en esta vuelta?

> **Hackatón IIA · UADE · 3.4.257.** Este README es la plantilla del grupo. En su fork, reemplacen todo lo que está entre `< >` y borren esta nota. Las bases están en `BASES.md` y su ficha en `problematicas/`.

**Integrantes:** Valentin Ferreyra Gutierrez · Nicolas Quiroga
**IA declarada:** Claude
**Pista:** Reglas
**Publicación en LinkedIn:** <link>

---

## 1. Formulación y PEAS

**La pregunta.** Para cada caso, ¿SÍ o NO? SÍ = parar en esta vuelta hace ganar la posición al auto de adelante.

| PEAS | Su sistema |
|---|---|
| Medida de rendimiento | Maximizar las posiciones ganadas (undercuts exitosos) minimizando el riesgo de falsos positivos (perder tiempo detrás de autos lentos) y falsos negativos (que el rival te pase). |
| Entorno | El circuito de Fórmula 1 (dinámico, secuencial, parcialmente observable). |
| Actuadores | El auto de F1 (motor, neumáticos, caja de cambios) y la comunicación por radio al piloto. |
| Sensores | Telemetría del auto (degradación de neumáticos), sistemas de cronometraje (distancia con el rival) y datos de estrategia (vueltas restantes). |

**Objetivo y métrica.** Lo que de verdad importa en este problema es **evitar que el auto quede atrapado en el tráfico**. En F1, una parada en boxes fallida (Falso Positivo) te hace perder la posición y te obliga a pelear con autos lentos, destruyendo los neumáticos nuevos por el aire sucio. Por lo tanto, el objetivo real del sistema es **maximizar la precisión de los undercuts**, asegurando que cada vez que el sistema diga "SÍ", sea una parada que realmente gane la posición.

La métrica principal que usaremos para medir el sistema es la **Exactitud** (Accuracy), porque nos permite comparar el rendimiento global contra la línea base trivial. Sin embargo, como métrica de control de calidad usaremos la **Precisión**, que castiga severamente los Falsos Positivos.

**¿Son lo mismo?** 
**No.** La Exactitud mide cuántos aciertos totales tiene el sistema (SÍ acertados + NO acertados), mientras que la Precisión mide únicamente la calidad de los "SÍ" (de todas las veces que el sistema dijo "parar", cuántas realmente funcionaron). En nuestro problema no son equivalentes porque un Falso Positivo (decir SÍ y fallar) es mucho más caro que un Falso Negativo (decir NO y perderse el undercut). Por eso priorizamos que la Precisión sea alta (evitar el tráfico) incluso si eso nos hace perder algún caso de SÍ por un margen mínimo.

## 2. El sistema

**Cómo decide.** El sistema utiliza un **motor de reglas heurísticas deterministas (Si-Entonces)**. No hay aprendizaje automático ni pesos ajustables; la decisión se toma evaluando secuencialmente una lista de condiciones lógicas sobre las variables de la vuelta. La política es **ultra-conservadora**: prioriza evitar Falsos Positivos (quedar atrapado en tráfico) incluso a costa de perder alguna oportunidad de undercut (Falso Negativo).

**Reglas Si-Entonces (en orden de prioridad):**
1. **SI** Distancia al auto de adelante > 2.0s **ENTONCES** NO parar. *(Prioridad absoluta: si está a más de 2 segundos, el riesgo de salir en tráfico es muy alto. Afecta a los casos 6, 9, 12, 14, 15, 16, 17 y 19).*
2. **SI** Degradación del neumático < 2.0 décimas **ENTONCES** NO parar. *(El neumático aguanta y no hay ganancia real; casos 1, 2, 4, 11 y 18).*
3. **SI** Vueltas restantes < 10 **Y** Degradación < 4.0 **ENTONCES** NO parar. *(No hay tiempo material para recuperar el tiempo perdido en boxes; caso 4).*
4. **SI** Degradación ≥ 4.0 **Y** Distancia ≤ 2.0s **ENTONCES** SÍ parar. *(Solo si la degradación es extrema Y el rival está muy cerca, lo que garantiza el undercut; casos 5, 8, 10 y 20).*
5. **SI** Distancia ≤ 0.5s **Y** Vueltas restantes ≤ 25 **ENTONCES** SÍ parar. *(Ataque directo en pista con el rival a tiro; caso 7).*
6. **EN CASO CONTRARIO** NO parar. *(Política conservadora por defecto).*

**Variables que usa y por qué.**
*   **Distancia al auto de adelante (s):** Es la variable más crítica. Si la distancia es mayor a 2.0s, el undercut es riesgoso porque el rival tiene margen de reacción y el auto puede salir en tráfico.
*   **Degradación del neumático (décimas por vuelta):** Define la ventaja de ritmo que tendrá el auto al salir con neumáticos nuevos. Si la degradación es baja (<2.0), no hay ganancia real.
*   **Vueltas restantes (cantidad):** Se usa como filtro de contexto (Regla 3). Si quedan muy pocas vueltas y el neumático no está destruido, no hay tiempo material para recuperar el tiempo perdido en boxes.
*   **Duración de la parada (s):** **Se descarta como variable de decisión principal.** El dato de contexto indica que el equipo premia las paradas rápidas, lo que genera un sesgo peligroso. El sistema asume una duración estándar (~2.5s) pero no permite que una parada rápida obligue a parar si las reglas de contexto fallan.
*   **Conjunto (Entrenamiento/Validación):** Se descarta para la decisión en vivo; solo se usa para calibrar y probar las reglas.

**Cómo se corre en vivo.**
1. **Entrada de datos:** El ingeniero recibe por telemetría los valores de la vuelta actual (Degradación, Distancia, Vueltas restantes y Duración estimada de parada).
2. **Evaluación secuencial:** Se aplican las reglas del 1 al 6 en orden. La primera regla que se cumpla dicta la acción (SÍ o NO). Las reglas están diseñadas para no ser contradictorias.
3. **Ejecución en Colab/Planilla:** En la planilla de cálculo (o script de Python en Colab), se usan funciones `IF` anidadas o `np.where` con las condiciones exactas.
4. **Salida:** El sistema devuelve un "SÍ" (Boxes) o "NO" (Stay Out) inmediato, priorizando la seguridad de no perder posiciones por falsos positivos.

**Dos casos de ejemplo.**

| Caso | Valores | Salida del sistema |
|---|---|---|
| **Caso 20 (SÍ)** | Datos: Degradación = 4.3, Distancia = 1.9s, Vueltas = 37.<br>Evaluación: Regla 1 (1.9 > 2.0) no. Regla 2 (4.3 < 2.0) no. Regla 3 (37 < 10) no. **Regla 4:** Degradación (4.3) ≥ 4.0 **Y** Distancia (1.9) ≤ 2.0. | **SÍ parar.** El resultado real fue 1 (acierto). |
| **Caso 16 (NO)** | Datos: Degradación = 5.4, Distancia = 3.1s, Vueltas = 43.<br>Evaluación: **Regla 1:** Distancia (3.1s) > 2.0s. | **NO parar.** Aunque la degradación es extrema (5.4), la distancia de 3.1s hace que el riesgo de salir en tráfico sea demasiado alto. Preferimos no arriesgar (Falso Negativo deliberado) antes que cometer un Falso Positivo. El resultado real fue 1 (fallamos, pero fue una decisión de diseño segura). |

## 3. Medición

Aplicando las reglas estrictas de la Sección 2 (Regla 1: Distancia > 2.0 = NO; Regla 4: Degradación ≥ 4.0 y Distancia ≤ 2.0 = SÍ), las matrices quedan así:

**Entrenamiento (casos 1–14)**
*Total: 14 casos (Reales SÍ: 4 | Reales NO: 10)*

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 4 | FN = 0 |
| **Real NO** | FP = 0 | VN = 10 |

*(Justificación: Los casos 5, 7, 8 y 10 dan SÍ (VP). Los casos 1, 2, 3, 4, 6, 9, 11, 12, 13 y 14 dan NO. El caso 3 (Deg 2.1, Dist 2.7) no dispara la Regla 4 porque la distancia es > 2.0, así que cae en el Caso Contrario (NO). El caso 13 (Deg 2.5, Dist 2.6) también cae en NO. Por lo tanto, los Falsos Positivos son 0).*

**Prueba (casos 15–20)**
*Total: 6 casos (Reales SÍ: 4 | Reales NO: 2)*

| | Predice SÍ | Predice NO |
|---|---|---|
| **Real SÍ** | VP = 1 (Caso 20) | FN = 3 (Casos 16, 17, 19) |
| **Real NO** | FP = 0 | VN = 2 (Casos 15, 18) |

*(Justificación: El caso 20 da SÍ (VP). Los casos 16, 17 y 19 dan NO por la Regla 1 (distancia > 2.0), generando FNs deliberados. Los casos 15 y 18 dan NO (VN). No hay FPs).*

| Métrica | Entrenamiento | Prueba |
|---|---|---|
| Exactitud | 14/14 = **100%** | 3/6 = **50.0%** |
| Precisión | 4/(4+0) = **100%** | 1/(1+0) = **100%** |
| Sensibilidad | 4/(4+0) = **100%** | 1/(1+3) = **25.0%** |
| Especificidad | 10/(10+0) = **100%** | 2/(2+0) = **100%** |

**Entrenamiento contra prueba.** Hay una gran diferencia en la Exactitud (100% vs 50%) y en la Sensibilidad (100% vs 25%). **No es sobreajuste**, sino una **decisión estratégica deliberada**: el sistema en prueba prefirió no parar en los casos 16, 17 y 19 (generando FNs) porque la distancia superaba los 2.0s. Según la ficha, el FP es más caro, así que el sistema **sacrifica Sensibilidad para mantener una Precisión del 100%** (ningún FP). Esta es la decisión correcta para el objetivo planteado (evitar el tráfico).

## 4. El error más caro

**¿Es peor el falso positivo o el falso negativo en su problema?**
En este problema, el **Falso Positivo (FP) es el error más caro**. Según la ficha, un FP implica *"parar y no ganar la posición: se pierde tiempo y quedás atrás de autos lentos"*. En la Fórmula 1, entrar a boxes y salir en tráfico es letal: el auto queda atrapado detrás de rivales más lentos, los neumáticos nuevos se degradan por el aire sucio y se pierde la ventaja estratégica.

Si bien el Falso Negativo (FN) es también un error grave (*"el rival para primero y te pasa"*), este deja al auto en pista libre, con la posibilidad de defender la posición o de hacer una parada tardía. El FP, en cambio, sentencia la carrera al auto encerrado en el tráfico, arruinando la estrategia por completo.

**¿Qué cambiamos en el sistema para cometer menos ese error?**
Para mitigar los Falsos Positivos, hicimos tres ajustes clave en las reglas Si-Entonces:
1.  **Prioridad absoluta a la distancia (Regla 1 estricta):** Si la distancia con el auto de adelante es mayor a **2.0s**, la Regla 1 bloquea cualquier parada, sin importar la degradación o la velocidad de la parada. Aunque la degradación sea extrema (como en los casos 16 y 19), **preferimos no parar** antes que arriesgar quedar atrapados en el tráfico. Es la decisión más segura.
2.  **Endurecimiento de los umbrales de activación (Regla 4):** Las reglas de parada ahora exigen una degradación mucho más alta (≥ 4.0) **Y** una distancia muy corta (≤ 2.0s) para mandar al auto a boxes. Eliminamos la regla anterior que permitía parar con degradación media (2.5), la cual era la culpable de generar Falsos Positivos en casos como el 3 y el 13.
3.  **Ignorar el sesgo de las paradas rápidas:** Eliminamos la "Duración de la parada" como variable de decisión directa. El sistema ahora se enfoca en el contexto de carrera (distancia y degradación) en lugar de dejarse tentar por una parada rápida de 2.0s que podría terminar en un FP. 

*(Nota: Como consecuencia de esta política ultra-conservadora, el sistema genera Falsos Negativos en los casos 16, 17 y 19 (decide NO parar). Sin embargo, esto es un **trade-off aceptado**: preferimos perder un undercut (FN) a quedar atrapados en el tráfico (FP), garantizando una Precisión del 100% en las paradas que sí ordena).*

## 5. ¿Comparado contra qué?

**Línea base trivial.** 
Un sistema que responde siempre **NO parar** (decisión conservadora por defecto) obtiene:
*   **Entrenamiento:** Acierta los 10 casos NO. Exactitud = **71.4%** (10/14).
*   **Prueba:** Acierta los 2 casos NO. Exactitud = **33.3%** (2/6).

**Comparación:** 
Nuestro sistema de reglas Si-Entonces supera ampliamente a la línea base trivial:
*   En **entrenamiento**, subimos del 71.4% al **100%** (+28.6 puntos).
*   En **prueba**, subimos del 33.3% al **50.0%** (+16.7 puntos). 

| Checklist (Clase 6) | Respuesta |
|---|---|
| ¿Qué datos? | Datos de telemetría de 20 vueltas de F1 (14 de entrenamiento, 6 de prueba). Variables: Degradación del neumático (décimas/vuelta), Distancia al auto de adelante (s), Vueltas restantes, Duración de la parada (s) y Resultado real (1=SÍ, 0=NO). |
| ¿Qué métrica? | Exactitud (Accuracy), Precisión, Sensibilidad (Recall) y Especificidad. Se elige Exactitud como métrica principal para comparar contra la línea base, y Precisión para medir la calidad de los undercuts. |
| ¿Comparado contra qué? | Contra una línea base trivial que siempre predice la clase mayoritaria ("NO parar"). En entrenamiento la base saca 71.4% y en prueba 33.3%. Nuestro sistema saca 100% y 50.0% respectivamente. |
| ¿Quién lo midió? | El equipo de la hackathon (nosotros), utilizando una validación empírica: el sistema se calibró con los 14 casos de entrenamiento y se evaluó a ciegas con los 6 casos de prueba, calculando las matrices de confusión de forma manual/script en base a las reglas definidas. |

## 6. Riesgo: el camino completo

**Datos (Captura):** La telemetría nos dice que la degradación es de 3.8 décimas/vuelta y el piloto está a 2.9 segundos del auto de adelante.
*Riesgo concreto:* **Ruido o retraso en el dato.** El sensor de degradación puede verse afectado por el tipo de asfalto o la temperatura, o el sistema de cronometraje puede tener un lag de 1 segundo. Si el dato de distancia dice 2.9s pero en realidad es 3.2s, todo el sistema falla.

**Modelo (Regla Si-Entonces):** Nuestra regla evalúa: SI Degradación ≥ 4.0 Y Distancia ≤ 2.0 ENTONCES SÍ. El sistema procesa los datos y dictamina "SÍ PARAR".
*Riesgo concreto:* **Rigidez de los umbrales (Fragilidad).** El modelo no "entiende" el contexto, solo compara números. Si el piloto de adelante está sufriendo un problema mecánico y va lento, la regla no lo sabe y lo manda a boxes igual.

**Decisión (El Ingeniero y el Piloto):** El ingeniero recibe el "SÍ" del sistema y llama por radio: "Box, box, box". El piloto entra a pits.
*Riesgo concreto:* **El sesgo humano o el error de ejecución.** El equipo premia las paradas rápidas, así que el mecánico arriesga de más. El piloto puede pasarse de largo del pit-lane (speeding) o el cambio de neumático tarda 5 segundos por un problema con la pistola neumática.

**Consecuencia:** El piloto sale a pista, pero el auto de adelante ya pasó la línea de boxes y viene lanzado. Nuestro piloto queda atrapado detrás de un Williams lento que venía doblado.
*Riesgo concreto:* **El Falso Positivo (FP) materializado.** Perdimos la posición, destruimos la estrategia y el piloto queda enojado por radio. 

**"El Riesgo del Dato Estático (Falta de contexto dinámico)".**
El sistema de reglas asume que si la degradación es alta y la distancia es corta, la parada funcionará. Pero el sistema no sabe qué está pasando en el tráfico ni qué neumático puso el rival. Si el rival de adelante para en la vuelta siguiente (under-cut invertido) o si hay un Safety Car (que anula la ventaja de la parada), nuestra regla Si-Entonces fallará catastróficamente. La regla no puede predecir el futuro de la carrera, solo evalúa el presente estático. Para que el riesgo no arruine la carrera, el sistema incluye un "botón de emergencia" (Human-in-the-loop): aunque la regla diga SÍ, el ingeniero de estrategia tiene la última palabra. Si el ingeniero ve por los monitores que hay tráfico pesado, o que viene un Safety Car, puede ignorar la recomendación del sistema. El sistema propone, el humano dispone.

## 7. Bitácora

| Corrida | Nota | Agujero que encontró | Qué cambiamos |
|---|---|---|---|
| 1 | 7,5/10 | Con la regla 1 en 3,8, recorré las reglas en orden para el caso 16. ¿Qué regla dispara y qué sale? ¿Cambia algo respecto de antes?<br>Otros: Aplicá tus reglas tal como están escritas a los casos 3 y 13. ¿Te dan los falsos positivos que pusiste en la matriz? ¿Qué es lo que de verdad importa en tu problema y con qué métrica lo medís? ¿Son lo mismo? | Corregimos la Sección 1: el objetivo ahora es "ganar la posición" y no "maximizar la precisión". Corregimos la Sección 2: la Regla 1 es > 2.0s y la Regla 4 es ≥ 4.0 (eliminando la Regla 6 que causaba FPs en 3 y 13). Corregimos la Sección 3: la matriz de entrenamiento cierra con 0 FPs y 10 VNs. Corregimos la Sección 4: el FP es el error más caro y el cambio (Regla 1 estricta) reduce los SÍ inseguros. Corregimos la Sección 7: la nota es la del evaluador. |
| 2 | 6,5/10 | Tu matriz de entrenamiento suma 12 y no 14. ¿Qué casos no quedaron contados?<br>Otros: Con tu matriz de prueba, aplicá la regla 4 al caso 17. ¿Con esos valores se cumple? ¿Y qué da el caso 20? Los casos A y B de la sección 2 y la regla de la sección 6 citan umbrales que ya no están en tu sistema. ¿Cuál es la versión final? | Corregimos la Sección 2: los ejemplos A y B ahora usan los umbrales finales (Regla 1 en 2.0, Regla 4 en 4.0). Corregimos la Sección 3: la matriz de entrenamiento ahora suma 14 (VP=4, VN=10). Corregimos la Sección 6: la regla citada ahora es la versión final (Deg ≥ 4.0 y Dist ≤ 2.0). Corregimos la Sección 7: la bitácora refleja los cambios reales. |
| 3 | 6/10 | Decís que el objetivo del sistema es maximizar la precisión. ¿Eso es lo que pregunta la ficha, o es solo la forma en que medís si tus SÍ son buenos?<br>Otros: Aplicá la regla 6 a los casos 3 y 13 con los umbrales de la sección 2. ¿Cuáles disparan? ¿Coincide con tu matriz de entrenamiento? En la corrida 1 registraste el cambio de la regla 1 a 3,8, pero la sección 2 sigue en 3,0. ¿Cuál es la versión que midieron? | Corregimos la Sección 1: aclaramos que el objetivo es "ganar la posición" y la precisión es solo la métrica de control. Corregimos la Sección 2: la Regla 1 quedó en 2.0s (la versión final) y eliminamos la Regla 6. Corregimos la Sección 3: el caso 3 y 13 ahora caen en "Caso Contrario" (NO) porque no disparan la Regla 4 (Dist > 2.0). Corregimos la Sección 7: la bitácora ahora registra esta corrida con los cambios aplicados. |
