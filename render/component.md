[<< 返回到主页](index.md)

**这里将介绍渲染引擎组成部分的博客文章**  

上一篇简单介绍了场景及场景物体的组织，并不涉及到物体真实渲染过程。   
接下来介绍物体被渲染的分解组成。  

根据OpenGL骨架代码可以清楚的认识，可以先分3个主要模块：  
**基础模块：OpenGL函数**  
**材质模块：着色器和贴图**  
**几何数据：模型文件**  

渲染引擎，实质上是对OpenGL图形库以数据驱动的模式做的上层封装，将绘制业务逻辑抽象出来，需要绘制的数据和OpenGL状态设定以参数的形式在程序运行的过程中输入。  

先直观感受下材质文件的配置信息，在后面的博客中会分析这些参数：  
1. material  
2. technique   
3. pass  
4. sampler  
5. renderState  
6. ......
等是怎样封装使用的。  

看一下材质文件（house_12.material）：  
```
material house_12
{
    technique 0
    {
        pass 0
        {
            //shaders and defines
            vertexShader = res/shaders/textured.vert
            fragmentShader = res/shaders/textured.frag
            defines = DIRECTIONAL_LIGHT_COUNT 1

            //uniforms
            u_worldViewProjectionMatrix = WORLD_VIEW_PROJECTION_MATRIX
            u_inverseTransposeWorldViewMatrix = INVERSE_TRANSPOSE_WORLD_VIEW_MATRIX

            //samplers
            sampler u_diffuseTexture
            {
                path = house_12_2.png
                mipmap = true
            }
            renderState
            {
                depthTest = true
            }
        }
    }
}
```

对材质文件结构有一个认识之后，其中需要设定2个必要的参数:vertexShader和fragmentShader， defines是可选的，主要针对着色器文件的宏定义。    

先直观感受下着色器文件的配置信息，在后面的博客中会分析这些限定符参数：    
1. attribute  
2. uniform  
是怎么封装和使用的，varying是针对着色器之间数据共享，应用程序不能设定值。  

看一下着色器文件，如下：  
顶点着色器 textured.vert     
```
#ifndef DIRECTIONAL_LIGHT_COUNT
#define DIRECTIONAL_LIGHT_COUNT 0
#endif
#ifndef SPOT_LIGHT_COUNT
#define SPOT_LIGHT_COUNT 0
#endif
#ifndef POINT_LIGHT_COUNT
#define POINT_LIGHT_COUNT 0
#endif
#if (DIRECTIONAL_LIGHT_COUNT > 0) || (POINT_LIGHT_COUNT > 0) || (SPOT_LIGHT_COUNT > 0)
#define LIGHTING
#endif
///////////////////////////////////////////////////////////
// Atributes
attribute vec4 a_position;

#if defined(SKINNING)
attribute vec4 a_blendWeights;
attribute vec4 a_blendIndices;
#endif

attribute vec2 a_texCoord;

#if defined(LIGHTMAP)
attribute vec2 a_texCoord1;
#endif

#if defined(LIGHTING)
attribute vec3 a_normal;

#if defined(BUMPED)
attribute vec3 a_tangent;
attribute vec3 a_binormal;
#endif

#endif

#if defined(VERTEX_COLOR)
attribute vec4 a_color;
#endif

///////////////////////////////////////////////////////////
// Uniforms
uniform mat4 u_worldViewProjectionMatrix;
#if defined(SKINNING)
uniform vec4 u_matrixPalette[SKINNING_JOINT_COUNT * 3];
#endif

#if defined(LIGHTING)
uniform mat4 u_inverseTransposeWorldViewMatrix;

#if defined(SPECULAR) || (POINT_LIGHT_COUNT > 0) || (SPOT_LIGHT_COUNT > 0)
uniform mat4 u_worldViewMatrix;
#endif

#if defined(BUMPED) && (DIRECTIONAL_LIGHT_COUNT > 0)
uniform vec3 u_directionalLightDirection[DIRECTIONAL_LIGHT_COUNT];
#endif

#if (POINT_LIGHT_COUNT > 0)
uniform vec3 u_pointLightPosition[POINT_LIGHT_COUNT];
#endif

#if (SPOT_LIGHT_COUNT > 0)
uniform vec3 u_spotLightPosition[SPOT_LIGHT_COUNT];
#if defined(BUMPED)
uniform vec3 u_spotLightDirection[SPOT_LIGHT_COUNT];
#endif
#endif

#if defined(SPECULAR)
uniform vec3 u_cameraPosition;
#endif

#endif

#if defined(TEXTURE_REPEAT)
uniform vec2 u_textureRepeat;
#endif

#if defined(TEXTURE_OFFSET)
uniform vec2 u_textureOffset;
#endif

///////////////////////////////////////////////////////////
// Varyings
varying vec2 v_texCoord;

#if defined(LIGHTMAP)
varying vec2 v_texCoord1;
#endif

#if defined(VERTEX_COLOR)
varying vec4 v_color;
#endif

#if defined(LIGHTING)

#if !defined(BUMPED)
varying vec3 v_normalVector;
#endif

#if defined(BUMPED) && (DIRECTIONAL_LIGHT_COUNT > 0)
varying vec3 v_directionalLightDirection[DIRECTIONAL_LIGHT_COUNT];
#endif

#if (POINT_LIGHT_COUNT > 0)
varying vec3 v_vertexToPointLightDirection[POINT_LIGHT_COUNT];
#endif

#if (SPOT_LIGHT_COUNT > 0)
varying vec3 v_vertexToSpotLightDirection[SPOT_LIGHT_COUNT];
#if defined(BUMPED)
varying vec3 v_spotLightDirection[SPOT_LIGHT_COUNT];
#endif
#endif

#if defined(SPECULAR)
varying vec3 v_cameraDirection;
#endif

#include "lighting.vert"

#endif

#if defined(SKINNING)
#include "skinning.vert"
#else
#include "skinning-none.vert"
#endif

void main()
{
    vec4 position = getPosition();
    gl_Position = u_worldViewProjectionMatrix * position;
    
#if defined(LIGHTING)
    vec3 normal = getNormal();
    // Transform the normal, tangent and binormals to view space.
    mat3 inverseTransposeWorldViewMatrix = mat3(u_inverseTransposeWorldViewMatrix[0].xyz, u_inverseTransposeWorldViewMatrix[1].xyz, u_inverseTransposeWorldViewMatrix[2].xyz);
    vec3 normalVector = normalize(inverseTransposeWorldViewMatrix * normal);
    
#if defined(BUMPED)
    
    vec3 tangent = getTangent();
    vec3 binormal = getBinormal();
    vec3 tangentVector  = normalize(inverseTransposeWorldViewMatrix * tangent);
    vec3 binormalVector = normalize(inverseTransposeWorldViewMatrix * binormal);
    mat3 tangentSpaceTransformMatrix = mat3(tangentVector.x, binormalVector.x, normalVector.x, tangentVector.y, binormalVector.y, normalVector.y, tangentVector.z, binormalVector.z, normalVector.z);
    applyLight(position, tangentSpaceTransformMatrix);
    
#else
    
    v_normalVector = normalVector;
    applyLight(position);
    
#endif
    
#endif
    
    v_texCoord = a_texCoord;
    
#if defined(TEXTURE_REPEAT)
    v_texCoord *= u_textureRepeat;
#endif
    
#if defined(TEXTURE_OFFSET)
    v_texCoord += u_textureOffset;
#endif
    
#if defined(LIGHTMAP)
    v_texCoord1 = a_texCoord1;
#endif
    
    // Pass the vertex color
#if defined(VERTEX_COLOR)
    v_color = a_color;
#endif
}

```

片段着色器 textured.frag :   
```
#ifdef OPENGL_ES
precision highp float;
#endif

#ifndef DIRECTIONAL_LIGHT_COUNT
#define DIRECTIONAL_LIGHT_COUNT 0
#endif
#ifndef SPOT_LIGHT_COUNT
#define SPOT_LIGHT_COUNT 0
#endif
#ifndef POINT_LIGHT_COUNT
#define POINT_LIGHT_COUNT 0
#endif
#if (DIRECTIONAL_LIGHT_COUNT > 0) || (POINT_LIGHT_COUNT > 0) || (SPOT_LIGHT_COUNT > 0)
#define LIGHTING
#endif

///////////////////////////////////////////////////////////
// Uniforms
uniform vec3 u_ambientColor;

uniform sampler2D u_diffuseTexture;

#if defined(LIGHTMAP)
uniform sampler2D u_lightmapTexture;
#endif

#if defined(LIGHTING)

#if defined(BUMPED)
uniform sampler2D u_normalmapTexture;
#endif

#if (DIRECTIONAL_LIGHT_COUNT > 0)
uniform vec3 u_directionalLightColor[DIRECTIONAL_LIGHT_COUNT];
#if !defined(BUMPED)
uniform vec3 u_directionalLightDirection[DIRECTIONAL_LIGHT_COUNT];
#endif
#endif

#if (POINT_LIGHT_COUNT > 0)
uniform vec3 u_pointLightColor[POINT_LIGHT_COUNT];
uniform vec3 u_pointLightPosition[POINT_LIGHT_COUNT];
uniform float u_pointLightRangeInverse[POINT_LIGHT_COUNT];
#endif

#if (SPOT_LIGHT_COUNT > 0)
uniform vec3 u_spotLightColor[SPOT_LIGHT_COUNT];
uniform float u_spotLightRangeInverse[SPOT_LIGHT_COUNT];
uniform float u_spotLightInnerAngleCos[SPOT_LIGHT_COUNT];
uniform float u_spotLightOuterAngleCos[SPOT_LIGHT_COUNT];
#if !defined(BUMPED)
uniform vec3 u_spotLightDirection[SPOT_LIGHT_COUNT];
#endif
#endif

#if defined(SPECULAR)
uniform float u_specularExponent;
#endif

#endif

#if defined(MODULATE_COLOR)
uniform vec4 u_modulateColor;
#endif

#if defined(MODULATE_ALPHA)
uniform float u_modulateAlpha;
#endif

#if defined(TEXTURE_DISCARD_ALPHA)
uniform float u_alphaThreshold;
#endif

///////////////////////////////////////////////////////////
// Variables
vec4 _baseColor;
///////////////////////////////////////////////////////////
// Varyings
varying vec2 v_texCoord;

#if defined(VERTEX_COLOR)
varying vec4 v_color;
#endif

#if defined(LIGHTMAP)
varying vec2 v_texCoord1;
#endif

#if defined(LIGHTING)

#if !defined(BUMPED)
varying vec3 v_normalVector;
#endif

#if defined(BUMPED) && (DIRECTIONAL_LIGHT_COUNT > 0)
varying vec3 v_directionalLightDirection[DIRECTIONAL_LIGHT_COUNT];
#endif

#if (POINT_LIGHT_COUNT > 0)
varying vec3 v_vertexToPointLightDirection[POINT_LIGHT_COUNT];
#endif

#if (SPOT_LIGHT_COUNT > 0)
varying vec3 v_vertexToSpotLightDirection[SPOT_LIGHT_COUNT];
#if defined(BUMPED)
varying vec3 v_spotLightDirection[SPOT_LIGHT_COUNT];
#endif
#endif
#if defined(SPECULAR)
varying vec3 v_cameraDirection; 
#endif
#include "lighting.frag"
#endif
void main()
{
    _baseColor = texture2D(u_diffuseTexture, v_texCoord);
 
    #if defined(VERTEX_COLOR)
    _baseColor *= v_color;
    #endif
    
    gl_FragColor.a = _baseColor.a;

    #if defined(TEXTURE_DISCARD_ALPHA)
    if (gl_FragColor.a < u_alphaThreshold)
        discard;
    #endif

    #if defined(LIGHTING)

    gl_FragColor.rgb = getLitPixel();
    #else
    gl_FragColor.rgb = _baseColor.rgb;
    #endif

    #if defined(LIGHTMAP)
    vec4 lightColor = texture2D(u_lightmapTexture, v_texCoord1);
    gl_FragColor.rgb *= lightColor.rgb;
    #endif

    #if defined(MODULATE_COLOR)
    gl_FragColor *= u_modulateColor;
    #endif

    #if defined(MODULATE_ALPHA)
    gl_FragColor.a *= u_modulateAlpha;
    #endif
}
```

顶点和片段着色器引入了光照程序 lighting.vert 和 lighting.frag    
lighting.vert 如下：  
```
#if defined(BUMPED)
void applyLight(vec4 position, mat3 tangentSpaceTransformMatrix)
{
    #if (defined(SPECULAR) || (POINT_LIGHT_COUNT > 0) || (SPOT_LIGHT_COUNT > 0))
    vec4 positionWorldViewSpace = u_worldViewMatrix * position;
    #endif
    
    #if (DIRECTIONAL_LIGHT_COUNT > 0)
    for (int i = 0; i < DIRECTIONAL_LIGHT_COUNT; ++i)
    {
        // Transform light direction to tangent space
        v_directionalLightDirection[i] = tangentSpaceTransformMatrix * u_directionalLightDirection[i];
    }
    #endif
    
    #if (POINT_LIGHT_COUNT > 0)
    for (int i = 0; i < POINT_LIGHT_COUNT; ++i)
    {
        // Compute the vertex to light direction, in tangent space
        v_vertexToPointLightDirection[i] = tangentSpaceTransformMatrix * (u_pointLightPosition[i] - positionWorldViewSpace.xyz);
    }
    #endif
    
    #if (SPOT_LIGHT_COUNT > 0)
    for (int i = 0; i < SPOT_LIGHT_COUNT; ++i)
    {
        // Compute the vertex to light direction, in tangent space
        v_vertexToSpotLightDirection[i] = tangentSpaceTransformMatrix * (u_spotLightPosition[i] - positionWorldViewSpace.xyz);
        v_spotLightDirection[i] = tangentSpaceTransformMatrix * u_spotLightDirection[i];
    }
    #endif
    
    #if defined(SPECULAR)
    // Compute camera direction and transform it to tangent space.
    v_cameraDirection = tangentSpaceTransformMatrix * (u_cameraPosition - positionWorldViewSpace.xyz);
    #endif
}
#else
void applyLight(vec4 position)
{
    #if defined(SPECULAR) || (POINT_LIGHT_COUNT > 0) || (SPOT_LIGHT_COUNT > 0)
    vec4 positionWorldViewSpace = u_worldViewMatrix * position;
    #endif

    #if (POINT_LIGHT_COUNT > 0)
    for (int i = 0; i < POINT_LIGHT_COUNT; ++i)
    {
        // Compute the light direction with light position and the vertex position.
        v_vertexToPointLightDirection[i] = u_pointLightPosition[i] - positionWorldViewSpace.xyz;
    }
    #endif

    #if (SPOT_LIGHT_COUNT > 0)
    for (int i = 0; i < SPOT_LIGHT_COUNT; ++i)
    {
        // Compute the light direction with light position and the vertex position.
        v_vertexToSpotLightDirection[i] = u_spotLightPosition[i] - positionWorldViewSpace.xyz;
    }
    #endif

    #if defined(SPECULAR)  
    v_cameraDirection = u_cameraPosition - positionWorldViewSpace.xyz;
    #endif
}

#endif
```

lighting.frag 如下:  
```

vec3 computeLighting(vec3 normalVector, vec3 lightDirection, vec3 lightColor, float attenuation)
{
    float diffuse = max(dot(normalVector, lightDirection), 0.0);
     vec3 diffuseColor = lightColor * _baseColor.rgb * diffuse * attenuation;

    #if defined(SPECULAR)

    // Phong shading
    //vec3 vertexToEye = normalize(v_cameraDirection);
    //vec3 specularAngle = normalize(normalVector * diffuse * 2.0 - lightDirection);  
    //vec3 specularColor = vec3(pow(clamp(dot(specularAngle, vertexToEye), 0.0, 1.0), u_specularExponent)); 

    // Blinn-Phong shading
    vec3 vertexToEye = normalize(v_cameraDirection);
    vec3 halfVector = normalize(lightDirection + vertexToEye);
    float specularAngle = clamp(dot(normalVector, halfVector), 0.0, 1.0);
    vec3 specularColor = vec3(pow(specularAngle, u_specularExponent)) * attenuation;
   
    #if defined(BUMPED)
    //gloss map in normap map texture's alpha channel
    float gloss = clamp(texture2D(u_normalmapTexture, v_texCoord).a * 4.0, 0.0, 1.0);
    return diffuseColor + specularColor * gloss;
    #else
    return diffuseColor + specularColor;
    #endif

    #else
    
    return diffuseColor;
    
    #endif
}

vec3 getLitPixel()
{
    #if defined(BUMPED)
    
    vec3 normalVector = normalize(texture2D(u_normalmapTexture, v_texCoord).rgb * 2.0 - 1.0);
    
    #else
    
    vec3 normalVector = normalize(v_normalVector);
    
    #endif
    
    vec3 ambientColor = _baseColor.rgb * u_ambientColor;
    vec3 combinedColor = ambientColor;

    // Directional light contribution
    #if (DIRECTIONAL_LIGHT_COUNT > 0)
    for (int i = 0; i < DIRECTIONAL_LIGHT_COUNT; ++i)
    {
        #if defined(BUMPED)
        vec3 lightDirection = normalize(v_directionalLightDirection[i] * 2.0);
        #else
        vec3 lightDirection = normalize(u_directionalLightDirection[i] * 2.0);
        #endif 
        combinedColor += computeLighting(normalVector, -lightDirection, u_directionalLightColor[i], 1.0);
    }
    #endif

    // Point light contribution
    #if (POINT_LIGHT_COUNT > 0)
    for (int i = 0; i < POINT_LIGHT_COUNT; ++i)
    {
        vec3 ldir = v_vertexToPointLightDirection[i] * u_pointLightRangeInverse[i];
        float attenuation = clamp(1.0 - dot(ldir, ldir), 0.0, 1.0);
        combinedColor += computeLighting(normalVector, normalize(v_vertexToPointLightDirection[i]), u_pointLightColor[i], attenuation);
    }
    #endif

    // Spot light contribution
    #if (SPOT_LIGHT_COUNT > 0)
    for (int i = 0; i < SPOT_LIGHT_COUNT; ++i)
    {
        // Compute range attenuation
        vec3 ldir = v_vertexToSpotLightDirection[i] * u_spotLightRangeInverse[i];
        float attenuation = clamp(1.0 - dot(ldir, ldir), 0.0, 1.0);
        vec3 vertexToSpotLightDirection = normalize(v_vertexToSpotLightDirection[i]);

        // TODO: Let app normalize this! Need Node::getForwardVectorViewNorm
        #if defined(BUMPED)
            vec3 spotLightDirection = normalize(v_spotLightDirection[i] * 2.0);
        #else
            vec3 spotLightDirection = normalize(u_spotLightDirection[i] * 2.0);
        #endif

        // "-lightDirection" is used because light direction points in opposite direction to spot direction.
        float spotCurrentAngleCos = dot(spotLightDirection, -vertexToSpotLightDirection);

        // Apply spot attenuation
        attenuation *= smoothstep(u_spotLightOuterAngleCos[i], u_spotLightInnerAngleCos[i], spotCurrentAngleCos);
        combinedColor += computeLighting(normalVector, vertexToSpotLightDirection, u_spotLightColor[i], attenuation);
    }
    #endif

    return combinedColor;
}

```