# LLM Council — una skill para Claude Code

*[English version](./README.md)*

Convierte una decisión difícil en cinco análisis independientes, una revisión a ciegas con ranking obligatorio, y un único veredicto que dice dónde coinciden los asesores, dónde chocan y qué hacer primero.

Adaptada del [LLM Council de Andrej Karpathy](https://github.com/karpathy/llm-council). Su versión envía la consulta a varios modelos de distintos laboratorios, hace que se puntúen entre ellos de forma anónima, y deja que un presidente redacte la respuesta final. Esta es esa misma tubería reconstruida para funcionar dentro de Claude Code, con una sustitución que se explica más abajo sin adornos.

**Solo para Claude Code.** No está pensada ni probada para claude.ai.

---

## Qué hace

```
   tu pregunta
        │
   Fase 0 · encuadre         lee CLAUDE.md, memory/, archivos referenciados
        │                    → una pregunta neutral para los cinco
        ▼
   Fase 1 · cinco asesores   5 agentes en paralelo, un solo mensaje
        │                    Contrarian · First Principles · Expansionist
        │                    Outsider · Executor
        ▼
   Fase 2 · revisión ciega   respuestas barajadas como Response A–E
        │                    5 revisores, cada uno con su propia lente
        │                    → notas + puntos ciegos + FINAL RANKING
        ▼
   Fase 3 · presidente       rango promedio por asesor
        │                    → coincidencias / choques / puntos ciegos
        │                      / ranking / recomendación / primer paso
        ▼
   veredicto en el chat  +  council-transcripts/{tema}-{fecha}.md
        │
   Fase 5 · html             OPCIONAL, solo si lo pides
                             se reconstruye desde la transcripción, cuando sea
```

Las cinco lentes no son personajes. Están elegidas para crear tres tensiones permanentes: Contrarian contra Expansionist (riesgo contra oportunidad), First Principles contra Executor (replantear contra ejecutar), y el Outsider en medio, que atrapa lo que la maldición del conocimiento vuelve invisible.

Cuatro asesores leen los archivos de tu proyecto (`Read`, `Glob`, `Grep`) para que el consejo hable de tu situación y no de una genérica. El Outsider no lee nada a propósito: todo su valor está en no tener contexto.

**La búsqueda web viene desactivada.** Es, con diferencia, la parte más cara de una sesión. Un asesor puede hacer una búsqueda puntual si el contexto local de verdad no sostiene un dato concreto, y debe advertirlo en su respuesta. Para activarla durante toda la sesión, agrega `--research` o simplemente pídelo.

---

## En qué se diferencia del LLM Council de Karpathy

| Etapa | Original de Karpathy | Esta skill |
|---|---|---|
| Diversidad | 4 modelos de 4 laboratorios distintos (gpt-5.1, gemini-3-pro, claude-sonnet-4.5, grok-4) con el mismo prompt | 5 lentes de pensamiento sobre el mismo modelo. Claude Code *sí* permite asignar un modelo distinto a cada subagente; usar uno solo es una decisión deliberada, explicada abajo |
| Etapa 1 | La misma consulta a cada modelo, sin persona | Pregunta enmarcada + contexto del proyecto para cada asesor, cada uno con su ángulo asignado y acceso a herramientas |
| Etapa 2 | Respuestas anónimas como `Response A`, `B`, `C`… (cuatro etiquetas, una por modelo), evaluación una a una, y luego un bloque `FINAL RANKING:` de formato estricto, que se parsea y promedia | El mismo formato literal, incluido el promedio de rangos, sobre cinco etiquetas. Dos cambios: cada revisor conserva su lente para que cinco reseñas no colapsen en una, y solo puntúa las cuatro respuestas que no escribió |
| Criterio del ranking | "Precisión y perspicacia", según su README. Su prompt real de la etapa 2 no enuncia criterio alguno: pide qué hace bien y qué hace mal cada respuesta, y exige el ranking | **Cuál cambia más la decisión.** A unos asesores deliberadamente parciales no se los puede comparar por calidad de respuesta |
| Etapa 3 | El modelo presidente sintetiza con libertad | El agente principal sintetiza en cinco secciones fijas orientadas a decidir, y puede darle la razón al disidente frente a la mayoría |
| Contexto | Solo la consulta | La fase 0 lee `CLAUDE.md`, `memory/`, archivos referenciados y transcripciones previas. Sin equivalente en el original |
| Salida | Interfaz web con pestañas por etapa | Veredicto en el chat + transcripción completa en disco, más un informe HTML si lo pides |

### Por qué existen las diferencias

El consejo de Karpathy se apoya en una propiedad que hace todo el trabajo silencioso: **los jueces están ciegos de verdad.** Cuatro modelos de cuatro laboratorios responden lo mismo, y cuando se puntúan entre ellos como `Response A` a `D`, ninguno puede saber con fiabilidad cuál es la suya ni quién escribió las otras — entrenamientos distintos, estilos distintos, nada que reconocer. Todas las demás reglas de su diseño son seguras *porque* eso se cumple. (Con rigor: es la premisa del diseño, no un hecho medido; él no la comprueba, y este repositorio tampoco.)

Claude Code permite asignar un modelo distinto a cada subagente, así que una copia literal es técnicamente posible. No es lo que hace esta skill, por dos razones. Todos los modelos disponibles en el entorno vienen del mismo laboratorio, así que mezclarlos compra variación de gama y no la independencia entre laboratorios sobre la que se construye el original: los sesgos compartidos siguen compartidos. Y mezclar gamas corrompe el ranking: un asesor que corre en un modelo más pequeño y queda último informa de la capacidad del modelo, no de si su ángulo valía la pena. La diversidad se fabrica entonces por ángulo asignado, con todos en igualdad de condiciones.

Esa decisión tiene una consecuencia: un ángulo se identifica solo. El Contrarian se reconoce en su primera frase, para quien lo lee y para sí mismo. Si copias las reglas de Karpathy sin tocarlas sobre esa base, heredas la forma del método sin lo que lo hacía funcionar. Tres lugares donde eso importa:

**1. Los revisores no pueden compartir un mismo prompt.** En el original las cuatro reseñas difieren porque las escribieron cuatro modelos distintos — el prompt es idéntico para todos, y no pasa nada. Dale a cinco instancias de *un mismo* modelo ese prompt idéntico y obtienes cinco reseñas casi iguales: cinco agentes gastados para comprar una sola opinión. Aquí cada revisor conserva su lente mientras juzga, así que las reseñas divergen por la misma razón por la que divergieron las respuestas.

**2. "La mejor respuesta" no es una pregunta puntuable aquí.** Los modelos de Karpathy dan cada uno su mejor intento honesto, así que puntuarlos por precisión y perspicacia tiene sentido. A estos asesores se les ordena ser parciales; preguntar cuál es "la mejor" compara a un pesimista deliberado con un optimista deliberado. El criterio pasa a ser **cuál cambiaría más la decisión de alguien inteligente que enfrenta este problema**, una pregunta que sigue significando algo cuando las entradas están sesgadas por diseño.

**3. Un revisor no puede puntuarse a sí mismo.** En el original cada modelo puntúa todas las respuestas, la suya incluida. Conviene ser exacto aquí, porque su README y su código no coinciden: el README dice que cada modelo recibe las respuestas de los *otros*, pero `stage2_collect_rankings` en `backend/council.py` arma un único prompt con toda la etapa 1 y lo envía a cada modelo del consejo. O sea, cada uno sí ve y puntúa la suya. Ahí es inofensivo: un modelo no tiene forma de distinguir la propia entre cuatro desconocidas.

Las nuestras la distinguen al instante. En pruebas, un revisor puso su propia respuesta en primer lugar pese a la instrucción explícita de no favorecer su propio ángulo; leyó la regla, y el tirón de darse la razón ganó. Repetir la prohibición con más énfasis es el arreglo que se siente bien y no cambia nada: el modelo ya la había entendido. Así que aquí cada revisor puntúa solo las cuatro respuestas que no escribió.

**Honestidad sobre este último arreglo:** funciona, pero no siempre. En las corridas medidas, aproximadamente uno de cada cinco revisores sigue colando su propia respuesta, y casi siempre arriba. Por eso la fase 3 comprueba el cumplimiento antes de promediar, descarta el ranking que no cumple, declara cuántos se contaron y, si el resultado está apretado, recalcula con el ranking descartado corregido para verificar que el orden no dependa de esa decisión.

Cada una de estas es una desviación de la letra del método para conservar aquello de lo que el método depende. Están documentadas en vez de escondidas porque quien lee merece saber qué decisiones son de Karpathy y cuáles de esta bifurcación.

**Lo que la sustitución sigue costando:** cinco instancias de un mismo modelo comparten sesgos y puntos ciegos que cuatro laboratorios independientes no compartirían. La coincidencia entre las lentes es una hipótesis fuerte, nunca una prueba independiente — y eso no lo arregla ninguna ingeniería de prompts, ni cambiar a un segundo modelo del mismo laboratorio. Si necesitas juicio genuinamente independiente, corre el original contra cuatro APIs.

---

## Cuándo usarla

Buenas preguntas para el consejo:

- "¿Lanzo un taller de 97 dólares o un curso de 497?"
- "¿Cuál de estos tres ángulos de posicionamiento es más fuerte?"
- "Estoy pensando en cambiar de X a Y. ¿Estoy loco?"
- "Este es el texto de mi página de venta. ¿Qué está flojo?"
- "¿Contrato a un asistente o construyo primero la automatización?"

Malas preguntas para el consejo:

- "¿Cuál es la capital de Francia?" — tiene una sola respuesta correcta
- "Escríbeme un tuit" — es una tarea de creación, no una decisión
- "Resume este artículo" — es procesamiento, no juicio

El consejo se gana su costo cuando hay incertidumbre real y equivocarse sale caro. Si ya sabes la respuesta y buscas que te la validen, lo más probable es que te diga justo aquello que estabas evitando. De eso se trata.

---

## Instalación

Clónala en tu carpeta de skills de usuario para tenerla disponible en todos tus proyectos:

```bash
git clone https://github.com/Carlos-Padilla-Bravo/llm-council-cc.git ~/.claude/skills/llm-council
```

```powershell
git clone https://github.com/Carlos-Padilla-Bravo/llm-council-cc.git "$env:USERPROFILE\.claude\skills\llm-council"
```

O déjala en un solo proyecto, en `.claude/skills/llm-council/` dentro de la raíz del repositorio.

La carpeta contiene `SKILL.md` (la skill), `report-template.html` (solo para el informe HTML opcional), este archivo, el README en inglés y la licencia, que Claude Code ignora.

Para comprobar que quedó instalada: abre Claude Code y escribe `/llm-council`.

---

## Cómo se usa

Cualquiera de estas frases la activa:

- `council this: <tu pregunta>`
- `run the council on <tu pregunta>`
- `pressure-test this`
- `stress-test this`
- `war room this`

También se activa con un dilema real planteado con naturalidad: "¿hago X o Y?", "estoy entre dos opciones", "cuál conviene". No se activa con búsquedas de datos, tareas de redacción, ni un "¿debería...?" casual sin nada en juego.

El idioma de la salida sigue al tuyo: si preguntas en español, el veredicto sale en español. La skill en sí está escrita en inglés.

Salida resumida:

```markdown
## Veredicto del Consejo: precio del curso para principiantes

### Dónde coincide el consejo
El ángulo de principiantes tiene demanda real, pero el planteamiento actual
vende una herramienta y no un resultado, y el comprador no reconoce el nombre
de la herramienta.

### Dónde choca el consejo
El precio. El Contrarian dice que 297 es mucho frente al contenido gratuito;
el Expansionist dice que es poco para lo que vale con comunidad incluida.

### Puntos ciegos que cazó la revisión
Tres de cinco revisores señalaron la misma omisión: nadie calculó la carga de
soporte de un grupo sin conocimientos técnicos.

### Ranking del consejo
| Asesor | Rango promedio | Rankings contados |
|---|---|---|
| The Outsider | 1,6 | 4 |
| The Executor | 2,2 | 4 |
| ... | | |

### La recomendación
No construyas el curso todavía. Valida con una oferta de menor compromiso y
replantea alrededor del resultado, no de la herramienta.

### Lo primero que hay que hacer
Haz un taller en vivo de 97 dólares con un título que nombre el resultado y no
el software.
```

---

## Qué queda en disco

El veredicto se entrega en el chat. La transcripción completa se guarda siempre en `council-transcripts/{tema}-{AAAA-MM-DD}.md` dentro de tu directorio de trabajo: la pregunta enmarcada, los archivos de contexto consultados, las cinco respuestas completas, las cinco reseñas, el mapa de letra a asesor, la tabla de ranking y el veredicto.

El consejo lee esa carpeta en sesiones posteriores, así que no vuelve a discutir terreno que ya diste por resuelto.

### Informe HTML (opcional)

Pídelo — "dame el informe en HTML", "make an HTML report" — y obtienes una página autocontenida junto a la transcripción. Nunca se genera por su cuenta, porque la mayoría de los veredictos se leen una vez y se actúa.

Puedes pedirlo días después, en otra sesión: se reconstruye desde la transcripción en Markdown, así que el consejo no necesita volver a reunirse. El Markdown sigue siendo la fuente de verdad; el HTML es una vista de él.

**Estilo.** `report-template.html` viene en registro oscuro y tecnológico —fondo casi negro, metadatos en monoespaciada, acento eléctrico, filetes finos— con una variante clara que sigue la preferencia del sistema de quien lo lee. No lleva marca personal ni de empresa, y cada color, tipografía y medida es una variable CSS en las primeras 40 líneas. Dos formas de hacerlo tuyo:

- Edita esas variables una vez, en tu propia copia. Nada más del archivo necesita cambiar.
- O bien, si tienes instalada una skill de identidad de marca, el informe la detecta sola y sobrescribe esas variables con tus valores.

La plantilla debe seguir siendo neutra: la comparten todos los que instalen la skill. Si bifurcas este repositorio y le incrustas tu marca, se la rompes a quien venga después.

---

## Costo y advertencias

- **Unos 10 agentes por sesión.** Cinco asesores, cinco revisores, y el presidente en línea. Es el diseño, no un accidente. No la corras con preguntas que no lo merezcan.
- **La búsqueda web domina el costo** cuando se activa, y por eso es opcional. Para dar una escala: en una corrida medida sin búsqueda, cada asesor consumió alrededor de 23 a 25 mil tokens; buscar multiplica esa cifra por un factor que este repositorio no ha medido. El informe HTML, en cambio, es una lectura de plantilla y una escritura.
- **El panel lo construyó una persona.** Que cinco lentes coincidan es una hipótesis fuerte, jamás un consenso del campo.
- **La anonimización es parcial.** Las lentes fijas se delatan solas. Lee el ranking como una señal aproximada, no como un marcador.
- **El rango promedio no es un voto.** Una lente puede quedar última y aun así aportar la observación que decide la pregunta. El presidente tiene instrucciones de decirlo cuando ocurra, y de ponerse del lado del disidente cuando su razonamiento sea el más fuerte.

---

## Créditos

- **Método:** [Andrej Karpathy — llm-council](https://github.com/karpathy/llm-council). La estructura de tres etapas, la revisión anónima entre pares, el formato estricto `FINAL RANKING:` y el promedio de rangos son suyos. Ese repositorio **no tenía licencia** cuando se escribió este (verificado el 2 de agosto de 2026), así que aquí no se afirma nada sobre sus términos — y no hizo falta, porque no se reproduce nada de su código ni de su texto. Lo que se reutiliza es el método, que su README y su `backend/council.py` describen abiertamente.
- **Adaptación previa:** la idea de llevar el consejo a una skill de Claude con cinco lentes de pensamiento viene de una skill atribuida a [Ole Lehmann](https://x.com/itsolelehmann) y distribuida en [aiwithremy/claude-skills-llm-council](https://github.com/aiwithremy/claude-skills-llm-council), también sin licencia. Este repositorio **no comparte texto con ella** —medido: cero frases idénticas— y rehace el diseño: los revisores conservan su lente, se restaura el ranking de Karpathy, el presidente es el agente principal, los asesores leen archivos del proyecto, la búsqueda web es opcional y la salida es un veredicto en el chat más una transcripción.
- **Esta implementación:** [Carlos Padilla Bravo](https://github.com/Carlos-Padilla-Bravo).

El contenido de este repositorio se publica bajo licencia MIT. Ver [LICENSE](./LICENSE).
