# Examenes RV
## Examen Junio 2024

### 1. Describa la etapa de “Rasterización e interpolación” que forma parte del proceso de renderizado. ¿Qué funciones se realizan en esta etapa? Describa brevemente como se desarrollan estas funciones.

La etapa de Rasterización e Interpolación es una etapa no programable, cuya entrada esta formada por primitivas ensambladas que referencian a los vértices cuyos atributos se calcularon en el Vertex Shader. Este se encarga de ver que vertices se quedan dentro y fuera del clippiing volume, se encarga de rasterizar que consiste en identificar que pixeles forman parte de la primitiva a dibujar y por ultimo de interpolar los valores de cada pixel como colores, normales y la coordenada de una textura.

- **Desechage de primitivas**: Las primitivas que quedan fuera del clipping volume son desechadas. Las que quedan parcialmente fuera deben adaptarse creando nuevos vertices sobre el borde del clipping volume. Los atributosd de los nuevos vértices se obtienen por interpolación lineal, salvo los atributos  declarados flat, que estos copian su valor a todos los vertices de la primitiva.

- **Rasterización**: Consiste en idientificar que pixeles forman parte de la primitiva a dibujar. Primero se calcula la orientacion de la primitiva, si el faceCulling esta activo se desecha las primitivas con orientación incorrecta. Luego se estudia que pixeles estan dentro de la primitiva. Para ello se estudia si el centro del picel se encuentra dentro del triángulo que forman los vértices.

- **Interpolacion**: Se calculan los valores de los atributos de cada pixel, para ellos se calculan los coeficientes de ionterpolación de cada pixel(a, b, c), considerando las coordenadas del centro del pixel p y la de los vertices (pa, pb, pc). Siendo p = a * pa + b *pb + c * pc. Donde a + b + c  = 1 y estan definidos entre 0 y 1.

### 2. Describa cuales son los tipos de datos vectoriales y matriciales utilizados en el lenguaje GLSL, así como los operadores aritméticos de suma, resta, multiplicación y división sobre estos tipos de datos.

Datos vectoriales:
- **vec2, vec3, vec4**: vectores de 2,3,4 elementos float.
- **dvec2, dvec3, dvec4**: vectores de doubles.
- **ivec2, ivec3, ivec4**: vectores de enteros.
- **uvec2, uvec4, uvec4**: vectores de enteros sin signo.
- **bvec2, bvec4, bvec4**: vectores de booleanos.

Datos matriciales:
- **mat2, mat3, mat4**: matrices cuadradas 2x2, 3x3 y 4x4 tipo float.
- **matNxM**: son matrices de tipo float de distinto tamaño por ejemplo mat4x3.
- **dmat2, dmat2, dmat3**: matrices cuadradas 2x2, 3x3 y 4x4 tipo double.
- **dmatNxM**: matrices de tipo double de distinto tamaño por ejemplo dmat3x2.

Operadores:
- **Suma(+) y Resta(-)**: Sobre vectotores si sumas o restas dos vectores se aplica elemento a elemento, si lo haces vector a escalar, se hace la misma operacion en todos sus elementos, para matrices solo se puede hacer entre matrices del mismo tamaño o con un escalar que se aplica la misma operacion a todos los elementos.
- **Multiplicacion(*) y Division(/)**: Entre dos vectores se hace la multiplicacióon o division de elemento a elemento, con un escalar la misma operacion a todos. Entre matrices la multiplicacion hace la multiplicación de matrices matemática estándar. La división como no existe en algebra de matrices se divide elemento a elemento de la matriz.

### 3. Describa el modelo de iluminación de Phong y desarrolle el código del Vertex Shader y del Fragment Shader que implementan este modelo.

El modelo de iluminación de Phong asume que la luz que percibimos de un objeto es la suma de tres componentes independientes: **Ambiental, Difusa y Especular**.

El modelo establece que el **Color Final = Ambiental(Ca) + Difusa(Cd) + Especular(Cs)**.

##### Componente Ambiental(Ca):

Simula la luz que ha rebotado tantas veces en el entorno que viene de todas las direcciones con la misma intensidad, evita que las zonas de sombra se queden completamente negras.

**Ca = Ia * Ka**

Ia es la intensidad de la luz ambiental y Ka la reflectividad ambiental del material.

##### Componente Difusa(Cd):

Simula la dispersion uniforme de la luz al chocar contra superficies rugosas o mates. Su intendidad depende unicamente del angulo de incidencia de los rayos de luz respecto a la superficie sin importar donde esté la camara.

**Cd = Id * Kd * (-I) * N**

Id es la intensidad de luz difusa, Kd es la reflectividad difusa del material, I es el vector de dirección de la luz y N la normal de la superficie.

##### Componente Especular:

Simula el brillo o destello brillante del sol sobre materiales pulidos. Aquí si importa la posición de la cámara.

**Cs = Is * Ks * ( R * V)^S**

Is: Intensidad de la luz especular.
Ks: Reflectividad especular del material.
R: Vector de reflexion de la luz sobre la superficie **R = I - 2(I * N)N**.
V: Vector de dirección de la cámara.
S: Es el brillo (Shinniness). Cuanto mas alto sea el número, mas pequeño, concentrado y nitido será el destello especular.


##### Vertex Shader:

```glsl
#version 400

layout (location = 0) in vec3 VertexPosition;
layout (location = 1) in vec3 VertexNormal;
layout (location = 2) in vec2 VertexTexCoord;

uniform may4 MVP;
uniform mat4 ModelViewMatrix;
uniform mat3 NormalMatrix;

out vec3 PositionCam;
out vec3 NormalCam;
out vec2 TexCoord;

void main()
{
    NormalCam = normalize(NormalMatrix * VertexNormal);
    PositionCam = vec3(ModelViewMatrix * vec4(VertexPosition, 1.0));
    TexCoord = VertexTexCoord;
    gl_Position = MVP * vec4(VertexPosition,1.0);
}

```

##### Fragment Shader:

```glsl
#version 400

in vec3 PositionCam;
in vec3 NormalCam;
in vec2 TexCoord;

struct LightInfo{
    vec4 Position;
    vec3 La;
    vec3 Ld;
    vec3 Ls;
};

struct MaterialInfo{
    vec3 Ka;
    vec3 Kd;
    vec3 Ks;
    float Shininess;
}

uniform LightInfo Light;
uniform MaterialInfo Material;
uniform sample2D BaseTex;

layout (location = 0) out vec4 FragColor;

vec3 calcularPhong(vec3 n, vec3 pos){
    vec3 ambient = Light.La * Material.ka;

    vec3 s;
    if(Light.Position.w == 0.0){
        s = normalize(vec3(Light.Position));

    }else{
        s = normalize(vec3(Light.Position) - pos);
    }

    float dRate = max(dot(s, n), 0.0);
    vec3 diffuse = Light.Ld * Material.Kd * dRate;

    vec3 specular = vec3(0.0);
    if(dRate > 0.0){
        vec3 v = normalize(-pos);
        vec3 r = reflect(-s, n);

        float sRate = max(dot(r, v), 0.0);
        specular = Light.Ls * Material.Ks * pow(sRate, Material.Shininess);
    }
    return ambient + diffuse + specular;

}

void main(){
    vec3 n = normalized(NormalCam);
    vec3 iluminacion = calcularPhong(n, PositionCam);

    vec4 texColor = texture(BaseTex, TexCoord);
    FragColor = vec4(iluminacion * texColor.rgb, texColor.a);
}

```

### 4. ¿Qué es una textura? Describa el proceso de creación de texturas y la configuración del proceso de aplicación de texturas. ¿Qué es un Normal-Map y como se aplica?

Se denomina textura a la aplicación de una imagen a una primitiva geométrica. Los puntos de las texturas se denominan texels. Existen texturas 1D, 2D, 3D y Cubemaps.

El proceso de creación de texturas requiere de 3 etapas fundamentales:

1. **Obtención o pintado de la imagen**: La imagen 2D puede obtenerse de fotografias reales, Pintado digital, o texturas procedimentales que son patrones matemáticos generados en la CPU o GPU.

2. **Mapeo ST**: Las coordenadas del objeto son en 3D, (X,Y,Z). Las de la textura son en 2D se llaman (S,T) y están normalizadas en rango [0.0, 1.0].

3. **Carga en memoria de video**: Una vez creada la imagen se pasa a la tarjeta gráfica mediante la API de gráficos en nuestro caso OpenGL y FreeImage. Se genera un identificado único con glGenTextures. Se activa y vincula con glBindTexture. Se transfieren los bytes a la memoria de video de la GOU usando glTexImage2D.


Un Normal-Map es una textura expecial donde cada téxel no guarda un color RGB, sino un vector de dirección 3D normalizado, esto es util para crear rugosidades a la textura.

Como las componentes de un vector van de [-1.0,1.0] y los canales de una imagen van de [0.0, 1.0f], los vectores codifican matemáticamente eb ka textura con la formula Color = (Vector + 1.0)/2.0.
El eje X se guarda en el canal Rojo, el Y en el canal Verde y el Z en el canal Verde.

El objetico es sustituir la normal lisa interpolada del triangulo por la normal detallada que leemos de la textura en cada pixel. El proceso en el Fragment Shader se desarrolla de la siguiente manera:
1. **Lectura de la textura:** Obtenemos el color RGB de la capa de normales usando las coordenadas de texturas tradicionesles.
```glsl
vec3 colorNormal = texture(NormalMapTex, TexCoord).rgb;
```

2. **Decodificación del vector**: Pasamos el codigo de vuelta al rango matemático de un vector [-1.0, 1.0].
```glsl
vec3 normalTextura = normalize(colorNormal * 2.0 - 1.0);
```

3. **El cambio en el espacio**: Los vectores del Normal-Map estan calculados asumienendo que la superficie es plana y mira hacia delante. Por ello el Vertex Shader calcula un sistema de coordenadas local para cada triangulo llamado espacio de Tangene, formado por tres vectores: La Tangente, La Bitangente y la Normal.

4. **Transformación final:** Multiplicamos el vector decodificado de la textura por la matriz de transformación mat4(T, B, N) para orientarlo correctamente en el espacio de la escena.
```glsl
vec3 normalFinalCamara = normalize(TBNMatrix * normalTextura);
```

Apartir de aqui el Fragment Shader utiliza esta normalFinal Camara con todas sus micro-rugosidades para ralizar los productos escaleres del modelo de Phong.
 

### 5. ¿En qué consiste el Shader de Geometría? Desarrolle un shader de geometría que sustituya un triángulo por tres triángulos utilizando el punto medio de los vértices del triángulo original como aparece en la figura

El Shader de Geoetría es una etapa opcional del pipeline programable de OpenGL que se situa justo despues del Vertex Shader o del TES si hay teselado y justo antes de la rasterización. A diferencia del Vertex Shader que procesa los vertices de uno en uno de forma aislada, el Geometry Shader conoce la topología completa de la primitiva y puede destruir o crear geometría en tiempo real. Recibe de entrada primitivas completas, las proecesa pudiendolas descartar o amplificarlas para crear vertices nuevas.

```glsl
#version 400
// 1. Configuración de entrada: Recibimos triángulos completos (3 vértices)
layout (triangles) in;

// 2. Configuración de salida: Vamos a emitir tiras de triángulos.
// Como creamos 3 triángulos y cada uno necesita 3 vértices, el máximo total son 9 vértices.
layout (triangle_strip, max_vertices = 9) out;

// Recibimos las matrices para proyectar el punto medio correctamente a la pantalla
uniform mat4 ProjectionMatrix;

// Nota: gl_in[] es un array nativo que contiene los datos calculados por el Vertex Shader
void main()
{
    // Paso A: Calcular el punto medio (baricentro) en el espacio de la escena
    // Sumamos las posiciones de las 3 esquinas y dividimos entre 3.0
    vec4 puntoMedio = (gl_in[0].gl_Position + gl_in[1].gl_Position + gl_in[2].gl_Position) / 3.0;

    // --------------------------------------------------------
    // TRIÁNGULO 1: Formado por Vértice 0, Vértice 1 y Punto Medio
    // --------------------------------------------------------
    gl_Position = gl_in[0].gl_Position;
    EmitVertex();
    
    gl_Position = gl_in[1].gl_Position;
    EmitVertex();
    
    gl_Position = puntoMedio;
    EmitVertex();
    
    EndPrimitive(); // Cerramos el primer triángulo

    // --------------------------------------------------------
    // TRIÁNGULO 2: Formado por Vértice 1, Vértice 2 y Punto Medio
    // --------------------------------------------------------
    gl_Position = gl_in[1].gl_Position;
    EmitVertex();
    
    gl_Position = gl_in[2].gl_Position;
    EmitVertex();
    
    gl_Position = puntoMedio;
    EmitVertex();
    
    EndPrimitive(); // Cerramos el segundo triángulo

    // --------------------------------------------------------
    // TRIÁNGULO 3: Formado por Vértice 2, Vértice 0 y Punto Medio
    // --------------------------------------------------------
    gl_Position = gl_in[2].gl_Position;
    EmitVertex();
    
    gl_Position = gl_in[0].gl_Position;
    EmitVertex();
    
    gl_Position = puntoMedio;
    EmitVertex();
    
    EndPrimitive(); // Cerramos el tercer triángulo
}

```

### 6. ¿Qué es una subrutina de GLSL? Describa la forma de declararlas y utilizarlas.

Una surutian es el equivalente técnico a las interfaces, clases abstractas o punteros a funciób de los lengujaes de programación trasdicionales, pero adaptado a la arquitectura paralela de las tarjetas gráficas. Su proposito principal es permitir que el codigo de la CPU cambie dinámicamente que algoritmo o función ejecuta el shader en tiempo de ejecución, sin necesidad de recompilar el shader y sin sufrir la pènalización de rendimiento que provocan los condicionales dinamicos if/else en el silicio de la GPU.

Para declarar una subrutina en eñl shadr se define la firma de la funcion.
```glsl
subroutine vec3 tipoIluminacion(vec3 normal, vec3 pos);
```

Y se escribe las implementaciones reales

```glsl
subroutine(tipoIlumiancion)
vec3 calcularPhong(vec3 normal, vec3 pos){
    //codigo
    return ambiental + difusa + especular;
}

subroutine(tipoIluminacion)
vec3 calcularColorPlano(vec3 normal, vec3 pos){
    return vec3(1.0, 0.0, 0.0);
}

```

Se crea una variable especial de tipo **subroutine uniform** utilizando el nombre del tipo que definimos en el pasoA.

```glsl

subroutine uniform tipoIluminación seleccionIluminación;

void main(){
    vec3 colorFInal = selecciónIluminacion(NormalCam, PositionCam);
    FragColor = vec4(colorFInal, 1.0);
}
```

Para usarlo desde la CPU primero se solicitan los identificador numercos de la variable uniform de la subrutina:

```C++
GLuint indicePhong = glGetSubroutineIndex(programID, GL_FRAGMENT_SHADER, "calcularPhong");
GLuint indicePlano = glGetSubroutineIndex(programID, GL_FRAGMENT_SHADER, "calcularColorPlano");
GLint locUniformSub = glGetSubroutineUniformLocation(programID, GL_FRAGMENT_SHADER, "seleccionIluminacion");
```

Justo antes de enviar a dibujar la geomtria cargamos en la GPU un array de enteros con los inidces de las funciones que queremos activar

```C++
glUniformSubroutinesuiv(GL_FRAGMENT_SHADER, 1, &indicePhong);
glDrawElements(GL_TRIANGLES, numIndices, GL_UNSIGNED_SHORT, NULL)
```


## Examen Julio 2024

### 1. Describa el proceso de creación y uso de un programa gráfico y de los shaders que lo forman

Por cada shader que necesites(mínimo un VertexShader y un FragmentShader), debes seguir estos pasos en el codigo:
1. **Crear el Objeto Shader(glCreateShader)**: Pides a OpenGL que reserve memoria en la GPU para un nuevo shader, especificando su tipo. Esto devuelve un identificador.

2. **Carar el código fuente(glShaderSource)**: Le pasas a OpenGL el texto plano en lenguaje GLSL.

3. **Compilas el shader(glCompileShader)**: Ordenas que la GPU traduzca el texto plano GLSL a código máquina.

4. **Comprobación de errores**: Para ver el estado de compilación se usa glGetShaderiv.

Una vez tienes los shaders compilados con éxito, son piezas sueltas. Necesitas unirlas en el programa gráfico para que puedan pasarse datos entre si.

1. **Creas el programa gráfico(glCreateProgram)**: Solicitas la creación del un programa contenedor vacio.

2. **Adjuntas los Shaders(glAttachShader)**: Le dices al programa cuales son los shaders que lo van a formar.

3. **Enlazas el programa(glLinkProgran)**: OpenGL conecta todas las piezas y comprueba que todas las salidas del Vertex encajan con las entradas del Fragment y asigna ubicaciones físicas a las variables uniform y prepara el ejecutable final.

4. **Comprobacion de errores**: con glGetProgramiv comprobamos si el linkado a sido existoso.

Para usar el programa se debe activar primero mediante glUseProgram antes de cargar los uniform y pedir dibujar los elementos.

Los shaders que conforman un programa son:

1. **Vertex Shader (Obligatorio)**: Procesa los vertices 1 a 1 y los transforma geomtricamente y espacialmente.

2. **Teselation Shaders(Opcional)**: se compone del TCS que decide cuantos triangulos nuevos va a crear es decir el nivel de detalle. Y se compone del TES que deforma esos vertices recien creados.

3. **Geometry Shader(Opcional)**: Este recibe primitivas completas  se encarga de crear o destruir geometria.

4. **Fragment Shader(Obligatorio)**: Tras la etapa de Rasterización este shader se ejecuta una vez por cada fragmento de la pantalla, dandole el color exacto al pixel.

### 2. Supongamos que deseamos iluminar una escena desde un foco situado en una posición fija como indica la figura. ¿Qué modificaciones habría que realizar sobre el modelo de iluminación de Phong para calcular el color de cada píxel? Desarrolle el código del Fragment Shader que implementa este modelo.

Para pasar de una luz direccional y puntual estandar a un foco, hay que añadir restriciones geométricas y nuevos parámetros al modelo original. 
- La luz necesita obligatoriamente una posición, dirección y angulo de apertura.
- Antes de calcular el brillo hay que calcular si el rayo de luz alcanza el pixel. 
- Si el fragmento cae dentro del cono se evaluan las componentes difusa y especular de phong de forma normal, si cae fuera estas se anulan y solo se calcula la componente ambiental.

```glsl

in vec3 PositionCam;
in vec3 NormalCam;

struct LightInfo{
    vec4 Position;
    vec3 Direction;
    float CutOff;
    vec3 La;
    vec3 Ld;
    vec3 Ls;
};

struct MaterialInfo{
    vec3 Ka;
    vec3 Kd;
    vec3 Ks;
    float Shininess;
};

uniform LightInfo Light;
uniform MaterialInfo Material;

out vec4 FragColor;

void main()
{
    vec3 n = normalize(NormalCam);
    vec3 s = normalize(vec3(Light.Position) - PositionCam);

    vec3 ambient = Light.La * Material.Ka;

    vec3 diffuse = vec3(0.0);
    vec3 specular = vec3(0.0);

    vec3 spotDir = normalize(Light.Direction);
    float cosAngulo = dot(-s, spotDir);

    if(cosAngulo > Light.CutOff)
    {
        float dRate = dot(s,n);
        diffuse = Light.Ld * Material.Kd * dRate;

        if(dRate > 0.0){
            vec3 v = normalize(-PositionCam);
            vec3 r = reflect(-s, n);

            float sRate = dot(r,v);
            specular = Light.Ls * Material.Ks * pow(sRate, Material.Shininess);
        }
    }

    vec3 colorFinal = ambient + diffuse + specular;
    FragColor = vec4(colorFinal, 1.0);
}
```

### 3. ¿Qué es una textura Cubemap? Desarrolle el Vertex Shader y el Fragment Shader que permite generar un Skybox (una imagen de fondo en 360º) por medio de una textura Cubemap.

Un Cubemap es un tipo especial de textura que esta formada por una sola imagen 2D plana, está compuesta por 6 imagenes 6 imagenes 2D cuadradas. Estas seis imagenes se ensamblan en la memoria de la tarjeta grafica. Forma una caja que envuelve la escena es util para crear paisajes de cielo, espacio, fondo.

```glsl
#version 400

// La posición local de la esquina del cubo
layout (location = 0) in vec3 VertexPosition;

// Salida: El vector de dirección 3D que usará el Fragment Shader
out vec3 TexCoords;

uniform mat4 ProjectionMatrix;
uniform mat4 ViewMatrix;

void main()
{
    // Pasamos la posición local directamente como coordenada de textura 3D
    TexCoords = VertexPosition;

    // TRUCO 1: Eliminamos la traslación de la ViewMatrix.
    // Al convertir un mat4 a mat3 y volverlo a mat4, matamos la última columna
    // que es la que guarda el movimiento (X, Y, Z) de la cámara.
    mat4 viewSinTraslacion = mat4(mat3(ViewMatrix));

    // Calculamos la posición final proyectada
    vec4 pos = ProjectionMatrix * viewSinTraslacion * vec4(VertexPosition, 1.0);

    // TRUCO 2: Forzar la profundidad al fondo absoluto.
    // Asignamos la coordenada Z igual a W. 
    // Cuando la GPU haga la división de perspectiva (Z/W), dará W/W = 1.0
    // 1.0 es la profundidad máxima posible en el Depth Buffer.
    gl_Position = pos.xyww;
}
```

```glsl
#version 400

in vec3 TexCoords;

uniform samplerCube SkyBoxTex;

layout (location = 0) out vec4 FragColor;

void main()
{
    FragColor = texture(SkyboxTex, TexCoords);
}

```


### 4. Describa el proceso de generación automática de primitivas por medio de las etapas de teselado. ¿Cuales son las entradas y salidas predefinidas de cada etapa? ¿Cómo se configura el nivel de teselado a generar?

A diferencia del Vertex Shader (que trabaja con vértices aislados) o el Geometry Shader (que trabaja con triángulos enteros), el teselado trabaja con un nuevo tipo de primitiva llamada Parche (Patch). Un parche es simplemente una agrupación de vértices de control.

**Tesselation Control Shader(TCS)**: Es programable su única función lógica es evaluar el parche original y decidir cuántos polígonos nuevos se van a generar a partir de él.
- **Entradas predefinidas**: Los vértices de control del parche generados por el Vertex Shader (gl_in[]).
- **Salidas predefinidas**: Los vértices de control (sin alterar o modificados) que se pasan a la última etapa (gl_out[]), los niveeles de Teselado enviados atrabes de las variables gl_TessLevelInner y gl_TessLevelOuter.

**El generador de primitivas no es programable**: Coge los niveles de teselado que ha dictado el TCS y crea una malla de vertices nuevos en un espacio abstracto 2D.
- **Entradas predefinidas**: Los niveles de Teselado y la configuracion de topología.
- **Salidas predefinidas**: Coordenadas abstractas o baricentricas de los millones de vertices almacenadas en la variable nativa gl_TessCoord.

**Tessellation Evaluation Shader(TES) programable**: Actúa como un Vertex Shader para los vértices recién creados. Coge esas coordenadas abstractas y calcula su posición 3D final. Aquí es donde normalmente se lee una textura de relieve (Mapa de Desplazamiento) para empujar los vértices hacia arriba y crear montañas o baches.
- **Entradas predefinidas**: Las coordenadas abstractas generadas por el PG (gl_TessCoord) y los vértices de control originales (gl_in[]).
- **Salidas predefinidas:**: La posición final obligatoria de cada nuevo vértice en la pantalla (gl_Position), que se envía directamente a la etapa de Rasterización.

Para configurar el Nivel de Teselado a generar se configura exclusivamente dentro del TCS, escribiendo valores flotantes en dos arrays predefinidos. El hawedware utiliza dos conceptos distintos para dividir la malla:
- **gl_TessLevelOuter[] (Niveles Exteriores)**: Indica en cuántos segmentos se va a dividir cada uno de los bordes externos del parche. Configurar los bordes por separado es vital para conectar un coche muy detallado con un asfalto poco detallado sin que aparezcan grietas (cracks) entre ambos objetos.
- **gl_TessLevelInner[] (Niveles Interiores)**:
Indica cuántos "anillos" de subdivisión se van a generar en el área interior del parche.

### 5. Describa en qué consiste el algoritmo de generación de sombras denominado ShadowMap.

El algoritmo ShadowMap sirve para generar sombras dinámicas.

1. El punto de vista de la Luz.
En la primera pasdada no se dibuja la escena para el jugador, sino para la fuente de luz.
- **Mover la cámara**: Colocamos una camara virtual exactamente en la posicion de la bombilla o el solo mirando en la direccion que se emite la luz.
- **Rendereoizamos solo profundidad:** Dibujamos todos los objetos de la escena, pero apagamos el calculo de colores. Lo unico que nos interesa es la distancia desde la luz hasta el primer objeto que choca.
- **Guardar en textura**: Ese mapa de distancias se guarda en una textura especial lamada Shadow Map mediante un FBO.

2. El punto de vista d ela camara.
En la segunda pasada, la cámara vuelve a a cabeza del jugador y dibujamos la escena normalmente a todo color, pero con un paso de comprobacion extra en el Fragment Shader.
Por cada pixel de la pantalla(por ejemplo, un pùnto en el asfalto):
- **Calculo de la distancia real**: Se calcula la distancia matematica raal desde ese punto de asfalto hasta la luz.
- **Lectura del ShadowMap**: Proyectamos ese punto del asfalto hacia el punto de vista de la luz para saber que coordenada UV le toa en la textura y leemos la distancia que guardamos en el paso 1.
- **Se comparan la distancias**: Si la real es mayor que la del mapa significa qie el pixel esta en sombra y solo se le aplica la luz ambiental. Sino se le aplica el Phong completo.

### 6.¿Qué es un Transform Feedback Object? Describa como puede utilizarse para desarrollar sistemas de partículas con la técnica conocida como “buffer ping-pong”.

Un Transform Feedback Object (TFO) es una "trampilla" que puedes abrir en medio de este camino. Permite capturar los vértices justo después de que salgan del Vertex Shader (o del Geometry Shader), evitar que vayan al Rasterizador y, en su lugar, guardarlos directamente de vuelta en otro buffer (VBO) en la memoria de la tarjeta gráfica.

¿Para qué sirve? Para usar la GPU como una supercalculadora matemática sin tener que dibujar nada en la pantalla, guardando los resultados para usarlos más tarde.

Para animar 1.000.000 de copos de nieve, necesitas actualizar su posición en cada fotograma aplicando físicas elementales:$$\vec{p}_{\text{nueva}} = \vec{p}_{\text{vieja}} + \vec{v} \cdot \Delta t$$El método antiguo (Malo): La CPU calcula la nueva posición del millón de copos de nieve y envía el nuevo millón de posiciones a la GPU en cada fotograma. Esto colapsa el bus de datos y el juego va a tirones.El método GPU (El problema): Queremos que el Vertex Shader haga la suma matemática porque es hiperrápido. Sin embargo, en OpenGL no puedes leer de un buffer y escribir en ese mismo buffer al mismo tiempo. Si el shader lee la posición del copo de nieve del Buffer 1, no puede sobrescribir el resultado ahí mismo.3. La solución: La técnica "Ping-Pong Buffer"Para solucionar la prohibición de lectura/escritura simultánea, creamos dos buffers idénticos (Buffer A y Buffer B) y los alternamos en cada fotograma como si fueran palas de ping-pong.Para que lo visualices a la perfección y te quede grabado para el examen, he preparado este esquema interactivo. Dale al botón de avanzar fotograma y fíjate en cómo cambian los roles de lectura y escritura:


## Examen Junio 2023

### 1. Describa el proceso de creación y uso de un programa gráfico (Shader Program).
Por cada shader que necesites(mínimo un VertexShader y un FragmentShader), debes seguir estos pasos en el codigo:
1. **Crear el Objeto Shader(glCreateShader)**: Pides a OpenGL que reserve memoria en la GPU para un nuevo shader, especificando su tipo. Esto devuelve un identificador.

2. **Carar el código fuente(glShaderSource)**: Le pasas a OpenGL el texto plano en lenguaje GLSL.

3. **Compilas el shader(glCompileShader)**: Ordenas que la GPU traduzca el texto plano GLSL a código máquina.

4. **Comprobación de errores**: Para ver el estado de compilación se usa glGetShaderiv.

Una vez tienes los shaders compilados con éxito, son piezas sueltas. Necesitas unirlas en el programa gráfico para que puedan pasarse datos entre si.

1. **Creas el programa gráfico(glCreateProgram)**: Solicitas la creación del un programa contenedor vacio.

2. **Adjuntas los Shaders(glAttachShader)**: Le dices al programa cuales son los shaders que lo van a formar.

3. **Enlazas el programa(glLinkProgran)**: OpenGL conecta todas las piezas y comprueba que todas las salidas del Vertex encajan con las entradas del Fragment y asigna ubicaciones físicas a las variables uniform y prepara el ejecutable final.

4. **Comprobacion de errores**: con glGetProgramiv comprobamos si el linkado a sido existoso.

Para usar el programa se debe activar primero mediante glUseProgram antes de cargar los uniform y pedir dibujar los elementos.

Los shaders que conforman un programa son:

1. **Vertex Shader (Obligatorio)**: Procesa los vertices 1 a 1 y los transforma geomtricamente y espacialmente.

2. **Teselation Shaders(Opcional)**: se compone del TCS que decide cuantos triangulos nuevos va a crear es decir el nivel de detalle. Y se compone del TES que deforma esos vertices recien creados.

3. **Geometry Shader(Opcional)**: Este recibe primitivas completas  se encarga de crear o destruir geometria.

4. **Fragment Shader(Obligatorio)**: Tras la etapa de Rasterización este shader se ejecuta una vez por cada fragmento de la pantalla, dandole el color exacto al pixel.

### 2. Describa cuales son los tipos de datos utilizados en el lenguaje GLSL.

Datos Primitivos:
- float
- int
- uint
- bool
- structs y arrays


### 6. Describa brevemente en qué consiste el algoritmo de generación de sombras denominado Shadow Volumes

Para calcular las sombras en una escena con este método, la tarjeta gráfica realiza tres pasos secuenciales:

La CPU (o más modernamente el Geometry Shader) analiza las mallas 3D de los objetos respecto a la posición de la luz.

Detecta el contorno de silueta del objeto (las aristas que separan los polígonos que miran a la luz de los que están a oscuras).

"Extruye" o estira esos vértices de la silueta hacia el infinito en la dirección opuesta a la luz. Esto genera un volumen geométrico cerrado (un prisma invisible). Todo lo que quede dentro de este volumen está físicamente en sombra.

Se dibuja la escena entera con la cámara del jugador de forma normal, calculando únicamente la luz ambiental (la penumbra) para rellenar el búfer de color y activar el búfer de profundidad (Depth Buffer).

Aquí se aplica la magia del hardware. Se dibuja el volumen de sombra invisible (sin escribir color en la pantalla) en dos pasadas especiales (técnica conocida como Z-pass o la optimizada Z-fail / Carmack's Reverse):

Caras frontales del volumen: Cuando el volumen entra en un objeto (píxel), se incrementa (+1) el valor del Stencil Buffer de ese píxel.

Caras traseras del volumen: Cuando el volumen sale del objeto, se decrementa (-1) el valor del Stencil Buffer de ese píxel.


## Examen Julio 2023

### 3. ¿Qué es un Vertex Buffer Object? ¿Y un Vertex Array Object? Describa la forma de crearlos y configurarlos.

Un VBO es un array de bytes ubicada directamente dentro de la memoria de la tarjeta gráfica. Su única mision es almacenar los datos puros de los vertices para que los shaders puedan acceder a ellos rapidamente.

Un VAO es un objeto de configuracion o estado. Como los VBO almacenan datos a lo bruto la grafica no sabra leerlos, el VAO es quien guarda las instrucciones de como se interpreta el contenido de un VBO.

Para crearlos primero pedimos a OpenGL que nos reserve espacion en memoria para un VAO yu VBO

```glsl

GLuint vao, vbo;

glGenVertexArray(1, &vao);
glGenBuffers(1, &vbo);
```

Activamos el Vao para que grabe todo lo que vamos a hacer y luego el VBO para introducir los datos reales.

```glsl
glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER); 

float vertices [] = {
    // vertices
};

glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
```

EL VAO se configura diciendole donde se ubica cada atributo

```glsl
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)0);
glEnableVErtexAttribArray(0);
glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 5 * sizeof(float), (void*)(3 * sizeof(float)));
glEnableVertexAttribArray(1); // Activamos el atributo 1 en el VAO

// --- FINALIZACIÓN ---
// Desactivamos el VAO para dejar de grabar de forma segura
glBindVertexArray(0);
```

Para usarlo en el bucle de renderizado seria:

```glsl
// 1. Activamos el shader del objeto
glUseProgram(shaderProgramID);

// 2. Activamos el VAO (esto activa automáticamente el VBO y su formato por dentro)
glBindVertexArray(vaoID);

// 3. Ordenamos a la GPU que dibuje los triángulos
glDrawArrays(GL_TRIANGLES, 0, 3);

// 4. Desactivamos el VAO por limpieza
glBindVertexArray(0);
```

### 5. ¿Qué es una textura? ¿Que es un mipmap? Describa las diferentes formas de aplicar una textura considerando mipmaps (texture-filtering).

Una textura es, en esencia, una imagen bidimensional (una rejilla de píxeles) que se "pega" o proyecta sobre la superficie de un objeto 3D, de forma similar a un papel de regalo o una pegatina.Para diferenciar los píxeles de la pantalla de los píxeles de la imagen de origen, a los elementos de una textura se les denomina Téxeles (Texels). Para aplicar la textura, cada vértice del modelo 3D lleva asignadas unas coordenadas normalizadas llamadas Coordenadas UV (con valores que van de $0.0$ a $1.0$), las cuales indican qué punto exacto de la imagen corresponde a ese vértice.


Un Mipmap es una secuencia de copias de la textura original optimizadas y precalculadas a resoluciones cada vez menores. Cada nivel del mipmap reduce a la mitad el ancho y el alto de la imagen anterior.


¿Por qué son obligatorios en los videojuegos?

Evitan el Aliasing (Centelleo): Si un objeto con una textura muy detallada (como una carretera de asfalto) se aleja mucho de la cámara, un solo píxel de la pantalla tendrá que representar cientos de téxeles a la vez. Al moverse la cámara, el píxel saltará bruscamente de un téxel a otro, provocando un parpadeo o ruido visual horrible. El mipmap soluciona esto usando una versión ya difuminada y pequeña de la textura para los objetos lejanos.

Mejoran el rendimiento (Caché de Texturas): Al renderizar objetos lejanos con texturas pequeñas, la tarjeta gráfica no satura el ancho de banda de la VRAM y aprovecha de forma óptima la memoria caché de la GPU.


Cuando la GPU va a pintar un fragmento en la pantalla, debe decidir qué téxel elegir de la textura. Esto se configura en OpenGL mediante los filtros de minificación (GL_TEXTURE_MIN_FILTER), que determinan cómo se comporta el muestreo cuando el objeto está lejos.

Existen cuatro combinaciones fundamentales de filtrado con mipmaps, nombradas en OpenGL siguiendo el patrón GL_NIVEL-PIXEL_MIPMAP_NIVEL-MIPMAP:

A. Filtrado por Vecino Más Próximo en Mipmaps(GL_NEAREST_MIPMAP_NEAREST)

- El hardware calcula qué nivel de mipmap le toca al objeto por su distancia.

- Va a ese nivel de mipmap concreto y escoge el téxel más cercano a la coordenada UV.

- Resultado: Es el método más rápido pero el de peor calidad. Al moverse el objeto en la lejanía, se puede apreciar visualmente una línea brusca en el suelo donde la tarjeta gráfica salta de un nivel de mipmap al siguiente.

B. Filtrado Lineal en Mipmap Único (GL_LINEAR_MIPMAP_NEAREST)

- El hardware calcula qué nivel de mipmap le toca al objeto por su distancia.

- Va a ese nivel de mipmap y realiza una interpolación bilineal (promedia el color de los 4 téxeles que rodean a la coordenada UV).

- Resultado: Los píxeles individuales se ven mucho más suaves y sin patrones de bloques (pixelación), pero sigue notándose el corte o salto brusco de nitidez entre los diferentes niveles de mipmap en las distancias.


C. Filtrado Bilineal entre Mipmaps (GL_NEAREST_MIPMAP_LINEAR)

- El hardware calcula la distancia y selecciona los dos niveles de mipmap más cercanos

- En cada uno de esos dos niveles, toma el téxel individual más cercano (NEAREST).

- Realiza una mezcla lineal entre los dos colores resultantes según la distancia intermedia del objeto.

- Resultado: Consigue que la transición entre mipmaps sea completamente fluida y desaparezcan las líneas de corte en el horizonte, pero la textura en sí puede verse algo pixelada en distancias cortas.


D. Filtrado Trilineal (GL_LINEAR_MIPMAP_LINEAR)

- La GPU selecciona los dos niveles de mipmap más cercanos.

- En cada uno de ellos, realiza una interpolación bilineal (promedia los 4 téxeles circundantes).

- Finalmente, realiza una interpolación lineal entre los dos resultados obtenidos de ambos niveles.

- Resultado: Máxima suavidad. Desaparecen por completo tanto la pixelación interna del téxel como los saltos de transición en el horizonte de la escena.


### 6. ¿Qué es un Frame Buffer Object? Describa la forma de crearlo, darle contenido y utilizarlo.

Un Frame Buffer Object (FBO) es un objeto de OpenGL que actúa como un lienzo de dibujo invisible o alternativo (Off-screen rendering).

Un FBO es solo un "contenedor" o un marco vacío; por sí mismo no tiene memoria para guardar píxeles. Para que funcione, tienes que conectarle puntos de anclaje (Attachments). Existen dos tipos de contenidos que le puedes pegar:

Texture Attachments (Anclajes de textura): Conectas una textura vacía al FBO. La GPU pintará ahí los colores. La gran ventaja es que luego puedes usar esa textura en tus shaders normales como si fuera una imagen cualquiera.

Renderbuffer Objects (RBO): Es un buffer optimizado exclusivamente para que la GPU escriba datos rápidamente (como el Depth Buffer o el Stencil Buffer), pero que no está diseñado para ser leído después dentro de un shader.

Creacion: 

```glsl
GLuint fboID;
glGenFramebuffers(1, &fboID);
glBindFramebuffer(GL_FRAMEBUFFER, fboID); // Lo activamos para empezar a configurarlo
```
Paso B: Crear la textura de destino (Anclaje de Color)

Creamos una textura normal y corriente de 2D, pero en lugar de subirle una imagen desde el disco, la dejamos vacía especificando el tamaño de nuestra pantalla.

```glsl
GLuint texturaColorID;
glGenTextures(1, &texturaColorID);
glBindTexture(GL_TEXTURE_2D, texturaColorID);

// Dejamos el último parámetro en NULL porque no subimos datos, la GPU los rellenará al dibujar
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGB, 1920, 1080, 0, GL_RGB, GL_UNSIGNED_BYTE, NULL);

// Filtros mínimos obligatorios para que la textura funcione
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

// ¡EL ANCLAJE!: Conectamos la textura al punto de color 0 de nuestro FBO
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_COLOR_ATTACHMENT0, GL_TEXTURE_2D, texturaColorID, 0);
```

Paso C: Crear el Renderbuffer (Anclaje de Profundidad)

Si queremos dibujar objetos 3D dentro del FBO, necesitamos un Depth Buffer para que el hardware sepa qué polígonos tapan a cuáles.

```glsl
GLuint rboID;
glGenRenderbuffers(1, &rboID);
glBindRenderbuffer(GL_RENDERBUFFER, rboID);

// Reservamos espacio de almacenamiento con formato de profundidad (DEPTH_COMPONENT)
glRenderbufferStorage(GL_RENDERBUFFER, GL_DEPTH_COMPONENT, 1920, 1080);

// ¡EL ANCLAJE!: Conectamos el Renderbuffer al punto de profundidad de nuestro FBO
glFramebufferRenderbuffer(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_RENDERBUFFER, rboID);
```

Para usarlo:

Pasada 1: Dibujar la escena dentro de la textura (Off-screen)

```glsl
// 1. Redirigimos el dibujo al FBO en vez de a la pantalla
glBindFramebuffer(GL_FRAMEBUFFER, fboID);

// 2. Limpiamos el lienzo invisible
glClearColor(0.1f, 0.1f, 0.1f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// 3. Dibujamos nuestro mundo (el circuito, los coches, los enemigos...) de forma normal
dibujarMundo3D();
```

Pasada 2: Dibujar el resultado en la pantalla real aplicando post-procesado
```glsl
// 1. Volvemos a activar el lienzo por defecto (la pantalla del monitor)
glBindFramebuffer(GL_FRAMEBUFFER, 0);

// 2. Limpiamos la pantalla real
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);

// 3. Activamos un shader especial de post-procesado (ej. que ponga la imagen en blanco y negro)
glUseProgram(shaderPostProcesadoID);

// 4. Activamos la textura del FBO (que ahora contiene la foto de todo nuestro mundo 3D)
glBindTexture(GL_TEXTURE_2D, texturaColorID);

// 5. Dibujamos un cuadrado plano que ocupe toda la pantalla (Fullscreen Quad)
// Al pintarse el cuadrado, el Fragment Shader leerá la textura del FBO, aplicará el filtro y lo verás en tu monitor.
dibujarCuadradoPantallaCompleta();
```
