# factos_brother
 
Análisis cuantitativo de las **Elecciones Presidenciales del Perú 2026**. Este Jupyter Notebook estima la distribución de los votos aún no contabilizados por la ONPE y proyecta el ranking final de candidatos usando álgebra lineal y simulación de Monte Carlo.
 
---
 
## Tabla de Contenidos
 
- [Motivación](#motivación)
- [Metodología](#metodología)
- [Simulación Monte Carlo](#simulación-monte-carlo)
- [Requisitos](#requisitos)
- [Uso](#uso)
- [Limitaciones](#limitaciones)
- [Notas](#notas)
---
 
## Motivación
 
Durante la jornada electoral, la ONPE publica resultados parciales a medida que las actas son procesadas. La pregunta natural es: **dados los votos ya contados, ¿qué se espera del resto?** Este notebook responde esa pregunta bajo un supuesto explícito de *prorrateo regional* y cuantifica la incertidumbre asociada mediante simulación estocástica.
 
---
 
## Metodología
 
### 1. Recolección de datos
 
Se utiliza **Selenium** + **BeautifulSoup** para extraer dos fuentes de la ONPE:
 
- Votos por candidato por región (endpoint de resultados presidenciales)
- Participación ciudadana por región (electores hábiles, asistentes, ausentes)
### 2. Estructuras de datos
 
Se construye la **matriz de votos** $V \in \mathbb{N}^{c \times r}$, donde $c$ es el número de candidatos (incluyendo Blanco y Nulo) y $r$ el número de regiones:
 
$$
V_{ij} = \text{votos del candidato } i \text{ en la región } j
$$
 
Y una tabla de participación por región con electores hábiles ($E_j$), asistentes ($A_j$) y ausentes ($U_j$).
 
### 3. Matriz de Popularidad
 
A partir de $V$ se deriva la **matriz de popularidad** $P \in [0,1]^{c \times r}$ normalizando por columnas:
 
$$
P_{ij} = \frac{V_{ij}}{\sum_{k=1}^{c} V_{kj}}
$$
 
Por construcción, cada columna suma 1:
 
$$
\sum_{i=1}^{c} P_{ij} = 1 \quad \forall j
$$
 
> **Supuesto clave:** los votos restantes en cada región se distribuirán según la popularidad observada hasta el momento. Este es el punto más débil del modelo y es justamente lo que la simulación de Monte Carlo busca robustecer.
 
### 4. Votos restantes por región
 
La identidad contable que debe cumplirse en cada región es:
 
$$
E_j = A_j + U_j + R_j
$$
 
donde $R_j$ son los electores cuyas actas aún no han sido contabilizadas. De estos, se estima cuántos efectivamente asistirán asumiendo que la **tasa de asistencia observada** se mantiene:
 
$$
\tau_j = \frac{A_j}{A_j + U_j}
$$
 
$$
R_j^{\text{asist}} = \tau_j \cdot R_j = \frac{A_j \cdot R_j}{A_j + U_j}
$$
 
Esto produce el vector de votos restantes $\mathbf{r} \in \mathbb{N}^{r}$ con $r_j = R_j^{\text{asist}}$.
 
### 5. Proyección determinista
 
Los votos esperados por candidato provenientes de las actas pendientes se obtienen por un producto matriz-vector:
 
$$
\mathbf{v}^{\text{esp}} = P \cdot \mathbf{r}, \quad \mathbf{v}^{\text{esp}} \in \mathbb{R}^{c}
$$
 
El vector de votos totales proyectados es:
 
$$
\mathbf{v}^{\text{total}} = \mathbf{v}^{\text{real}} + \mathbf{v}^{\text{esp}}
$$
 
donde $v^{\text{real}}_i = \sum_{j=1}^{r} V_{ij}$ son los votos ya contabilizados. Se rankean los candidatos según $\mathbf{v}^{\text{total}}$.
 
---
 
## Simulación Monte Carlo
 
La proyección determinista asume que la popularidad observada es exactamente la popularidad real. Para modelar la incertidumbre, se ejecutan $N = 10{,}000$ simulaciones estocásticas.
 
### Distribución Dirichlet
 
Por cada simulación y cada región, se perturba el vector de popularidad observado $\mathbf{p}_j = (P_{1j}, \ldots, P_{cj})$ usando una **distribución de Dirichlet**:
 
$$
\tilde{\mathbf{p}}_j \sim \text{Dirichlet}(\alpha_j), \quad \alpha_j = \kappa \cdot \mathbf{p}_j
$$
 
donde $\kappa = 500$ es el **parámetro de concentración** (Dirichlet strength).
 
**¿Qué es la distribución Dirichlet y qué significa $\kappa$?**
 
La Dirichlet es la generalización multidimensional de la distribución Beta: genera vectores de probabilidades que suman 1. En términos simples, es como lanzar un "dado sesgado" donde los sesgos mismos son inciertos.
 
El parámetro $\kappa$ controla **qué tan concentrada está la distribución alrededor del vector observado**:
 
- $\kappa \to \infty$: las muestras son prácticamente idénticas a $\mathbf{p}_j$ (cero ruido, equivalente al modelo determinista).
- $\kappa \to 0$: las muestras son casi aleatorias (ruido extremo, cualquier candidato podría ganar cualquier región).
- $\kappa = 500$: las muestras varían moderadamente alrededor de $\mathbf{p}_j$, simulando que la popularidad observada es una estimación razonable pero no exacta.
Intuitivamente, $\kappa = 500$ equivale a decir *"confío en la popularidad observada con una fuerza equivalente a haber observado 500 votos por región"*, lo cual produce una banda de incertidumbre consistente con el tamaño muestral de una elección.
 
### Distribución Multinomial
 
Dada la popularidad perturbada $\tilde{\mathbf{p}}_j$ de la simulación, los votos restantes de la región se reparten entre candidatos usando una **Multinomial**:
 
$$
\mathbf{s}_j \sim \text{Multinomial}(r_j, \tilde{\mathbf{p}}_j)
$$
 
### Algoritmo completo
 
Para cada simulación $n = 1, \ldots, N$:
 
1. **Perturbar** la popularidad por región: $\tilde{\mathbf{p}}_j^{(n)} \sim \text{Dirichlet}(\kappa \cdot \mathbf{p}_j)$ para cada $j$.
2. **Repartir** los restantes: $\mathbf{s}_j^{(n)} \sim \text{Multinomial}(r_j, \tilde{\mathbf{p}}_j^{(n)})$.
3. **Sumar** votos reales y simulados:
$$
\mathbf{v}^{(n)} = \mathbf{v}^{\text{real}} + \sum_{j=1}^{r} \mathbf{s}_j^{(n)}
$$
 
4. **Rankear** candidatos y registrar la posición de cada uno.
Tras las $N$ simulaciones, se estiman probabilidades empíricas como:
 
$$
\mathbb{P}(\text{candidato } i \text{ pasa a segunda vuelta}) \approx \frac{1}{N} \sum_{n=1}^{N} \mathbb{1}\left[\text{rank}_i^{(n)} \leq 2\right]
$$
 
Se reportan los percentiles P5, P50 y P95 de los votos totales por candidato, y las probabilidades $\mathbb{P}(\text{Top 1})$, $\mathbb{P}(\text{Top 2})$ y $\mathbb{P}(\text{Top 3})$.
 
---
 
## Requisitos
 
- Python 3.10+
- `selenium`, `beautifulsoup4`, `lxml`
- `pandas`, `numpy`, `matplotlib`
- Google Chrome + ChromeDriver (para el scraping)
---
 
## Uso
 
```bash
jupyter notebook analisis_electoral_peru_2026.ipynb
```
 
Si Selenium no logra conectarse a la ONPE, el notebook cae automáticamente a datos hardcodeados de respaldo para permitir ejecutar el análisis sin conexión.
 
---
 
## Limitaciones
 
- **Supuesto de prorrateo regional:** se asume que los votos restantes en cada región siguen la distribución observada. Si existen sesgos sistemáticos en el orden de conteo (por ejemplo, actas rurales procesadas al final), el modelo subestima la varianza real. La simulación Monte Carlo mitiga parcialmente este problema, pero no lo elimina.
- **Tasa de asistencia fija:** se asume que $\tau_j$ permanece constante entre las actas contadas y las pendientes.
- **Independencia regional:** la simulación trata cada región como independiente; no modela correlaciones macropolíticas entre regiones.
---
 
## Notas
 
- Debido a la ausencia de una API documentada de la ONPE, la recolección vía Selenium es relativamente lenta. Para las próximas elecciones se planea reemplazar el scraping por llamadas directas a los endpoints internos de la plataforma, habilitando análisis en tiempo real.
- El código original fue escrito en una sola noche durante la jornada electoral y posteriormente refactorizado con ayuda de Claude AI para mejorar legibilidad y estructura. **El análisis, los supuestos estadísticos y la lógica del modelo son propios**; el uso de asistentes de IA se limitó a la optimización del scraping y a la presentación del notebook.
