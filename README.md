# Examenes RV
## Examen Junio 2024

#### 1. Describa la etapa de “Rasterización e interpolación” que forma parte del proceso de renderizado. ¿Qué funciones se realizan en esta etapa? Describa brevemente como se desarrollan estas funciones.

La etapa de Rasterización e Interpolación es una etapa no programable, cuya entrada esta formada por primitivas ensambladas que referencian a los vértices cuyos atributos se calcularon en el Vertex Shader. Este se encarga de ver que vertices se quedan dentro y fuera del clippiing volume, se encarga de rasterizar que consiste en identificar que pixeles forman parte de la primitiva a dibujar y por ultimo de interpolar los valores de cada pixel como colores, normales y la coordenada de una textura.

- **Desechage de primitivas**: Las primitivas que quedan fuera del clipping volume son desechadas. Las que quedan parcialmente fuera deben adaptarse creando nuevos vertices sobre el borde del clipping volume. Los atributosd de los nuevos vértices se obtienen por interpolación lineal, salvo los atributos  declarados flat, que estos copian su valor a todos los vertices de la primitiva.

-**Rasterización**: Consiste en idientificar que pixeles forman parte de la primitiva a dibujar. Primero se calcula la orientacion de la primitiva, si el faceCulling esta activo se desecha las primitivas con orientación incorrecta. Luego se estudia que pixeles estan dentro de la primitiva. Para ello se estudia si el centro del picel se encuentra dentro del triángulo que forman los vértices.

-**Interpolacion**: Se calculan los valores de los atributos de cada pixel, para ellos se calculan los coeficientes de ionterpolación de cada pixel(a, b, c), considerando las coordenadas del centro del pixel p y la de los vertices (pa, pb, pc). Siendo p = a * pa + b *pb + c * pc. Donde a + b + c  = 1 y estan definidos entre 0 y 1.

#### 2. Describa cuales son los tipos de datos vectoriales y matriciales utilizados en el lenguaje GLSL, así como los operadores aritméticos de suma, resta, multiplicación y división sobre estos tipos de datos.

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

#### 3. Describa el modelo de iluminación de Phong y desarrolle el código del Vertex Shader y del Fragment Shader que implementan este modelo.

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
