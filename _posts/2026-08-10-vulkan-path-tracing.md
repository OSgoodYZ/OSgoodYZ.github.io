---
title: "Vulkan 하드웨어 레이트레이싱으로 패스 트레이서 만들기"
categories: [Graphics]
tags: [Vulkan, RayTracing, PathTracing, GLSL]
toc: true
toc_sticky: true
---

개인 프로젝트로 만들고 있는 Vulkan 렌더링 엔진 OGHypeEngine에 `VK_KHR_ray_tracing_pipeline` 기반 패스 트레이서를 구현했습니다. glTF 샘플 모델인 `DragonAttenuation`의 유리 드래곤 — 투명한 재질에 두께에 따른 볼륨 감쇠가 들어간 모델 — 을 제대로 렌더링하는 것이 목표였습니다.

Vulkan RT 확장의 기초적인 사용법(BLAS는 무엇인가 같은)은 다루지 않습니다. 그건 이미 좋은 자료가 많으니, 여기서는 **왜 그렇게 짰는지**에 집중하겠습니다.

## 1. RHI에 레이트레이싱 얹기

엔진은 그래픽 API 중립 RHI 계층을 두고 그 아래에 Vulkan 백엔드를 두는 구조입니다. 레이트레이싱도 같은 원칙을 따랐습니다. 샘플 코드는 `OgAccelStructureHandle`, `OgRayTracingPipelineDescriptor` 같은 중립 타입만 보고, Vulkan 세부사항은 백엔드에 가둡니다.

레이트레이싱은 확장이라 지원 여부 판정이 먼저입니다. 확장 존재 여부만 보면 안 되고 **함수 포인터가 실제로 로드됐는지까지** 확인해야 합니다.

```cpp
if (!_vulkanDevice->HasExtension(VK_KHR_ACCELERATION_STRUCTURE_EXTENSION_NAME) ||
    !_vulkanDevice->HasExtension(VK_KHR_RAY_TRACING_PIPELINE_EXTENSION_NAME) ||
    !_vulkanDevice->HasExtension(VK_KHR_RAY_QUERY_EXTENSION_NAME))
{
    LOGD(OG_ID, "Ray tracing extensions not available");
    return;
}

if (!_vulkanDevice->vkCreateAccelerationStructureKHR ||
    !_vulkanDevice->vkCmdTraceRaysKHR ||
    !_vulkanDevice->vkGetRayTracingShaderGroupHandlesKHR ||
    /* ... */)
{
    LOGD(OG_ID, "Ray tracing function pointers not loaded");
    return;
}

_rayTracingSupported = true;
```

여기서 걸리면 조용히 래스터라이저로 돌아가면 됩니다. 확장은 있는데 포인터 로드가 빠져서 뒤늦게 널 크래시로 만나는 것보다, 초기화 시점에 한 번에 판정하는 편이 훨씬 낫습니다.

## 2. 가속 구조: 통합 버퍼와 인덱스 배달 문제

BLAS는 **메시 단위로** 하나씩 만들고, glTF 노드 인스턴스마다 TLAS 엔트리를 만듭니다. 정점·인덱스 데이터는 프리미티브별로 쪼개지 않고 **전부 하나의 큰 버퍼로 합칩니다.**

```cpp
for (const auto& geom : _geometries)
{
    GPUGeometryInfo info;
    info.vertexOffset = static_cast<uint32_t>(allVertices.size());
    info.indexOffset  = static_cast<uint32_t>(allIndices.size());
    info.materialIndex = geom.materialIndex;

    allGeometryInfos.push_back(info);
    allVertices.insert(allVertices.end(), geom.cpuVertices.begin(), geom.cpuVertices.end());
    allIndices.insert(allIndices.end(), geom.cpuIndices.begin(), geom.cpuIndices.end());
}
```

버퍼를 하나로 합치면 descriptor 개수가 지오메트리 수와 무관하게 고정된다는 장점이 있습니다. 대신 **셰이더가 "지금 맞은 삼각형이 이 거대 버퍼의 어디쯤인가"를 알아야 하는** 문제가 생깁니다.

Vulkan RT가 hit 셰이더에 공짜로 주는 정보는 `gl_InstanceCustomIndexEXT`(TLAS 인스턴스마다 CPU가 심어둔 24비트 값), `gl_GeometryIndexEXT`(BLAS 안에서 몇 번째 지오메트리인가), `gl_PrimitiveID`(그 지오메트리 안에서 몇 번째 삼각형인가) 정도입니다.

그래서 TLAS 인스턴스를 만들 때 `instanceCustomIndex`에 **그 메시의 첫 지오메트리 전역 인덱스**를 넣습니다.

```cpp
tlasInst.instanceCustomIndex = meshBlas.primitiveOffset; // 이 메시의 첫 geometry 전역 인덱스
```

그러면 셰이더에서 둘을 더하는 것만으로 오프셋 테이블을 찾아갈 수 있습니다.

```glsl
GeometryInfo geomInfo = geometryInfoBuffer.geometryInfos[
    gl_InstanceCustomIndexEXT + gl_GeometryIndexEXT];

uint i0 = indexBuffer.indices[geomInfo.indexOffset + gl_PrimitiveID * 3 + 0];
// ...
Vertex v0 = vertexBuffer.vertices[geomInfo.vertexOffset + i0];
```

간접 참조 한 번으로 정점·재질에 모두 도달합니다. 이 인덱스 배달 경로는 틀렸을 때 증상이 아주 고약합니다. 크래시가 아니라 **엉뚱한 재질과 노멀이 섞인 그럴듯한 그림**이 나오거든요. BLAS별 `firstGeometryIndex`를 `LOGD`로 찍어두면 검증이 훨씬 쉽습니다.

버퍼 usage 플래그도 함정입니다. AS 빌드 입력으로 쓰려면 `SHADER_DEVICE_ADDRESS`와 `ACCEL_STRUCTURE_BUILD_INPUT`이 반드시 필요합니다. 빠뜨리면 validation layer가 잡아주긴 하지만, 켜두지 않았다면 한참 헤맵니다.

## 3. SBT는 버퍼 하나로

Shader Binding Table은 raygen / miss / shadow miss / hit 네 그룹이지만, **버퍼는 하나만** 만들고 오프셋으로 나눠 씁니다.

```
[ raygen ][ miss ][ shadowMiss ][ hit ]
     0        1×A       2×A        3×A     (A = handleSizeAligned)
```

```cpp
sbt.raygenSBT = _raygenSBT;  sbt.raygenOffset = 0;         sbt.raygenSize = aligned;
sbt.missSBT   = _missSBT;    sbt.missOffset   = 1 * aligned;
sbt.missSize  = 2 * aligned;  // miss 2개(primary + shadow)가 연속
sbt.hitSBT    = _hitSBT;     sbt.hitOffset    = 3 * aligned;
```

`missSize`가 `2 * aligned`인 게 포인트입니다. `traceRayEXT`의 miss 인덱스(마지막에서 두 번째 인자)로 0이면 primary miss, 1이면 shadow miss가 선택되도록 두 핸들을 연속 배치했습니다.

정렬 값은 반드시 디바이스에서 조회해야 합니다(`shaderGroupHandleAlignment`). `handleSize`를 그대로 stride로 쓰면 특정 GPU에서만 깨지는, 제일 만나기 싫은 종류의 버그가 됩니다.

## 4. 경로 추적은 raygen이 전담한다

이 구현의 뼈대가 되는 결정입니다.

> **closest hit은 셰이딩하지 않는다. 표면 정보만 payload에 담아 돌아온다.**
> **경로 추적은 raygen의 `for` 루프가 전담한다.**

closest hit 셰이더는 정점 보간과 재질 조회까지만 하고 끝냅니다.

```glsl
// 표면 정보만 payload로 반환 — 셰이딩과 경로 연장은 raygen의 패스 트레이싱 루프에서 수행
payload.position   = position;
payload.hitT       = gl_HitTEXT;
payload.normal     = normal;
payload.transmission = material.transmissionFactor;
payload.baseColor  = baseColor.rgb;
payload.ior        = material.ior;
payload.emissive   = material.emissiveFactor.rgb;
payload.attenuationDistance = material.attenuationDistance;
payload.attenuationColor    = material.attenuationColor.rgb;
```

사실상 **레이 하나짜리 G-buffer**입니다. 그리고 raygen이 루프를 돕니다.

```glsl
vec3 radiance = vec3(0.0);
vec3 throughput = vec3(1.0);

for (uint bounce = 0u; bounce <= ubo.maxBounces; ++bounce)
{
    payload.hitT = -1.0;
    traceRayEXT(topLevelAS, gl_RayFlagsOpaqueEXT, 0xff, 0, 0, 0,
                origin, 0.001, dir, 10000.0, 0);

    if (payload.hitT < 0.0) {            // miss → 하늘 수집 후 종료
        radiance += throughput * skyColor(dir);
        break;
    }

    radiance += throughput * payload.emissive;
    // ... 재질에 따라 dir/origin/throughput 갱신 ...
}
```

`throughput`은 지금까지 경로가 살아남으며 곱해온 감쇠, `radiance`는 수집한 빛의 누적입니다. 매 바운스마다 `origin`과 `dir`을 갱신해 다음 세그먼트로 넘어갑니다.

이 구조를 택한 이유는 세 가지입니다.

**첫째, 셰이더 재귀는 하드웨어 자원입니다.** closest hit 안에서 다시 `traceRayEXT`를 부르는 구조라면 파이프라인 생성 시 그만큼 `maxRecursionDepth`를 요구해야 하고, 드라이버는 그 값을 근거로 스택을 잡습니다. 디바이스가 보장하는 `maxRayRecursionDepth`는 생각보다 작을 수 있어(스펙 최소 보장은 1입니다) 깊은 재귀를 전제한 파이프라인은 이식성이 떨어집니다.

여기서는 `traceRayEXT`가 **raygen에서만, 그것도 루프의 같은 위치에서만** 호출됩니다. 덕분에 요구 깊이는 2면 충분합니다.

```cpp
rtPipeDesc.maxRecursionDepth = 2; // 패스 트레이싱 루프는 raygen에서만 trace (재귀 없음)
```

2인 이유는 그림자 레이 한 단계 때문입니다.

**둘째, 경로 분기가 폭발하지 않습니다.** 유리 표면에서 반사와 굴절 양쪽으로 레이를 갈라 보내면 깊이 N에서 레이가 2^N개가 됩니다. 루프 구조에서는 프레넬을 확률로 써서 둘 중 하나만 고르므로 레이 개수가 항상 1개로 유지됩니다(다음 절).

**셋째, 경로 전체가 한 함수 안에 펼쳐집니다.** NEE, 러시안 룰렛, 볼륨 감쇠처럼 "경로의 이력"을 알아야 하는 기법들이 그냥 루프 안의 지역 변수로 해결됩니다. 재귀 구조였다면 payload에 상태를 욱여넣어야 했을 것들입니다.

## 5. 유리: 프레넬 확률 샘플링과 Beer-Lambert

유리 표면에서는 반사와 굴절 중 **하나만** 고릅니다. 프레넬 값을 혼합 비율이 아니라 **선택 확률**로 씁니다.

```glsl
float ior = payload.ior <= 0.0 ? 1.5 : payload.ior;
float eta = backface ? ior : (1.0 / ior);
vec3 refr = refract(dir, faceN, eta);

float f0 = pow((1.0 - ior) / (1.0 + ior), 2.0);
float fres = f0 + (1.0 - f0) * pow(clamp(1.0 - abs(NdotD), 0.0, 1.0), 5.0);
bool tir = dot(refr, refr) < 0.0001;   // refract()는 전반사 시 0 벡터를 반환

if (tir || randomFloat() < fres) {
    dir = reflect(dir, faceN);
    origin = payload.position + faceN * 0.001;
} else {
    dir = refr;
    origin = payload.position - faceN * 0.001;
    throughput *= payload.baseColor;    // glTF 스펙: 투과광은 baseColor로 틴트
}
```

레이 하나로 유지되지만, 여러 프레임에 걸쳐 평균을 내면 기댓값은 정확한 프레넬 혼합에 수렴합니다. 몬테카를로의 이점을 그대로 가져오는 부분입니다.

세부 사항 두 가지. GLSL 내장 `refract()`가 전반사에서 0 벡터를 돌려주므로 TIR 판정을 따로 계산할 필요가 없습니다. 그리고 굴절일 때는 origin을 노멀 **반대쪽**으로 밀어야 self-intersection을 피할 수 있습니다.

### 볼륨 감쇠

`DragonAttenuation` 모델의 핵심은 이름 그대로 **감쇠(attenuation)** 입니다. 유리를 통과한 빛이 두께에 비례해 흡수되어 색이 진해지는 현상이죠.

루프 구조에서는 이게 놀랍도록 간단합니다. **뒷면(backface)에 맞았다는 건 방금 유리 내부를 지나왔다는 뜻**이고, 그 거리는 `payload.hitT`에 이미 들어 있습니다.

```glsl
float NdotD = dot(payload.normal, dir);
bool backface = NdotD > 0.0;
vec3 faceN = backface ? -payload.normal : payload.normal;

// 유리 내부를 지나온 세그먼트의 Beer-Lambert 감쇠
if (backface && payload.transmission > 0.0 && payload.attenuationDistance > 0.0)
{
    vec3 absorb = -log(payload.attenuationColor + 0.001) / payload.attenuationDistance;
    throughput *= exp(-absorb * payload.hitT);
}
```

glTF의 `attenuationColor`는 "`attenuationDistance`만큼 통과했을 때 남는 색"으로 정의됩니다. 그래서 흡수 계수는 로그를 취해 역산합니다. `+ 0.001`은 완전 검정 감쇠색에서 `log(0)`이 터지는 걸 막는 가드입니다.

"이 레이가 지금 유리 안에 있는가"를 따로 추적할 필요 없이, 매 바운스 앞뒷면만 보면 된다는 게 핵심입니다.

## 6. NEE와 이중 계산 방지

디퓨즈 표면에서 코사인 반구 샘플링만으로 광원을 우연히 맞기를 기다리면 노이즈가 어마어마합니다. 그래서 **Next Event Estimation** — 표면마다 광원을 직접 조준하는 그림자 레이 — 을 씁니다.

문제는 여기서 발생합니다. NEE로 광원 기여를 이미 더했는데, 그 다음 바운스에서 레이가 우연히 광원에 또 맞으면 **같은 빛을 두 번 세게 됩니다.**

해결은 플래그 하나입니다.

```glsl
// 직전 바운스가 카메라/스페큘러일 때만 광원 발광을 직접 수집
// (디퓨즈는 NEE가 담당 → 이중 계산 방지)
bool specularPath = true;
```

디퓨즈 바운스 뒤에는 `specularPath = false`, 유리(반사·굴절) 바운스 뒤에는 `true`로 둡니다. 광원 구체에 맞았을 때 `specularPath`가 참일 때만 발광을 수집하면, 디퓨즈 경로의 광원 기여는 오직 NEE만 담당하게 됩니다. 유리 너머로 보이는 광원이 제대로 밝게 보이는 것도 이 규칙 덕분입니다(유리 경로는 NEE 대상이 아니니까요).

한 가지 더. 광원은 **씬 지오메트리로 넣지 않고 셰이더에서 해석적으로 판정**합니다.

```glsl
// 광원 구체와의 해석적 레이-구 교차
// (씬 지오메트리 대신 셰이더에서 판정 → 광원 이동 시 AS 재빌드 불필요)
float hitLightSphere(vec3 ro, vec3 rd)
{
    vec3 oc = ro - ubo.lightPos.xyz;
    float b = dot(oc, rd);
    float c = dot(oc, oc) - LIGHT_RADIUS * LIGHT_RADIUS;
    float h = b * b - c;
    if (h < 0.0) return -1.0;
    float t = -b - sqrt(h);
    return t > 0.001 ? t : -1.0;
}
```

광원을 메시로 넣었다면 위치를 바꿀 때마다 TLAS를 다시 빌드해야 합니다. UI 슬라이더로 광원을 끌고 다니고 싶은데 매 프레임 AS 재빌드는 곤란하죠. 구 하나쯤은 셰이더에서 푸는 게 훨씬 쌉니다. 유니폼 버퍼만 갱신하면 끝입니다.

NEE에서 광원 표면의 임의 점을 고르는 것도 같은 반경을 씁니다. 점광원이 아니라 면광원으로 근사되므로 그림자 경계가 부드러워집니다.

```glsl
vec3 lightPoint = ubo.lightPos.xyz + randomUnitVector() * LIGHT_RADIUS; // 면광원 근사 → 부드러운 그림자
```

## 7. 그림자 레이 두 발과 cullMask

유리 그림자를 어떻게 할 것인가. 유리는 빛을 통과시키니 완전히 검은 그림자를 만들면 안 됩니다. 그렇다고 그림자 레이에서 유리를 통째로 무시하면 이번엔 아예 그림자가 없어집니다.

제대로 하려면 굴절 코스틱까지 시뮬레이션해야 하지만, 실시간에서는 과합니다. 그래서 **그림자 레이를 두 발** 쏘고 cullMask로 구분합니다.

```glsl
// 그림자 레이 1: 불투명 물체만 (cullMask 0x01) → 가려지면 완전 차단
shadowHitValue = vec3(0.0);
traceRayEXT(topLevelAS, shadowRayFlags, 0x01, 0, 0, 1,
            shadowOrigin, 0.001, L, lightDist, 1);
vec3 shadowOpaque = shadowHitValue;

// 그림자 레이 2: 유리 포함 (cullMask 0xff) → 유리에만 가려지면 빛 일부 투과
shadowHitValue = vec3(0.0);
traceRayEXT(topLevelAS, shadowRayFlags, 0xff, 0, 0, 1,
            shadowOrigin, 0.001, L, lightDist, 1);
vec3 shadowFactor = shadowOpaque * mix(vec3(0.4), vec3(1.0), shadowHitValue);
```

이게 성립하려면 CPU 쪽에서 투과 재질을 가진 인스턴스의 마스크를 미리 빼놔야 합니다.

```cpp
// 이 메시에 투과(유리) 재질이 있으면 그림자 레이(cullMask 0x01)에서 제외
tlasInst.mask = transmissive ? 0xFE : 0xFF;
```

`0xFE`는 하위 1비트만 0인 값입니다. 즉 유리는 `0x01` 마스크 레이에 걸리지 않고, `0xff` 레이에는 걸립니다. 결과적으로:

| 상황 | 레이 1 (0x01) | 레이 2 (0xff) | 결과 |
|---|---|---|---|
| 아무것도 없음 | 통과 | 통과 | 완전 조명 |
| 유리만 가림 | 통과 | 차단 | 40% 밝기 |
| 불투명 물체가 가림 | 차단 | 차단 | 완전 그림자 |

`0.4`라는 숫자는 물리적 근거가 없는 눈대중입니다. 다만 "유리 그림자가 흐리게 진다"는 인상은 이 두 줄로 충분히 얻어집니다.

그림자 레이에는 플래그 세 개를 겁니다.

```glsl
uint shadowRayFlags = gl_RayFlagsTerminateOnFirstHitEXT
                    | gl_RayFlagsOpaqueEXT
                    | gl_RayFlagsSkipClosestHitShaderEXT;
```

가려졌는지만 알면 되니 **가장 가까운 교차점을 찾을 필요도, closest hit 셰이더를 실행할 필요도 없습니다.** miss 셰이더가 `shadowHitValue = vec3(1.0)`을 쓰는지 여부만 보면 됩니다. 그림자 레이는 전체 레이 수의 상당 부분을 차지하므로 여기서 아끼는 게 큽니다.

## 8. 프로그레시브 누적을 이미지 한 장으로

패스 트레이싱은 프레임을 누적해야 노이즈가 걷힙니다. 정석은 선형 HDR 누적 버퍼를 따로 두고, 화면 출력용으로 톤매핑한 이미지를 별도로 쓰는 것입니다.

여기서는 버퍼를 하나만 씁니다. **인코딩을 가역으로 만들어** 이미지 한 장이 두 역할을 겸하게 했습니다.

```glsl
// 표시용 인코딩(Reinhard 톤매핑 + 감마). 역변환이 가능해서
// 하나의 이미지로 선형 공간 누적과 화면 표시를 겸한다
vec3 encodeDisplay(vec3 c)
{
    c = c / (c + vec3(1.0));
    return pow(c, vec3(1.0 / 2.2));
}

vec3 decodeDisplay(vec3 d)
{
    vec3 c = pow(d, vec3(2.2));
    c = min(c, vec3(0.999));
    return c / (vec3(1.0) - c);
}
```

Reinhard `x/(1+x)`의 역함수가 `y/(1-y)`라는 점을 이용했습니다. 누적은 이렇게 됩니다.

```glsl
if (ubo.frameCount > 0)
{
    vec3 prevLinear = decodeDisplay(imageLoad(image, ivec2(launchID)).rgb);
    float weight = 1.0 / float(ubo.frameCount + 1);
    color = mix(prevLinear, color, weight);
}
imageStore(image, ivec2(launchID), vec4(encodeDisplay(color), 1.0));
```

읽어서 **선형으로 되돌리고**, 누적 평균을 내고, 다시 표시용으로 인코딩해 저장합니다. 가중치 `1/(n+1)`은 running average라 프레임 수에 무관하게 정확한 평균이 나옵니다.

정직하게 말하면 이건 **절충안**입니다. `min(c, 0.999)` 클램프에서 보이듯 아주 밝은 값은 왕복 과정에서 정밀도를 잃습니다. 렌더 타겟은 `rgba32f`라 포맷 자체의 여유는 있지만, Reinhard가 `[0,1)`로 압축하는 순간 하이라이트 구간의 유효 비트가 깎이니까요. 별도 누적 버퍼가 정답이고, 나중에 디노이저를 붙일 때는 어차피 선형 버퍼가 필요합니다. 다만 지금 단계에서는 리소스 셋 바인딩 하나와 이미지 하나를 아끼는 값어치가 있었습니다.

누적은 카메라가 움직이면 무효화됩니다.

```cpp
glm::mat4 currentView = _camera->GetViewMatrix();
if (_previousViewMatrix != currentView)
{
    _frameCount = 0;
    _previousViewMatrix = currentView;
}
```

광원 위치, 바운스 수, SPP를 바꾸는 세터들도 전부 `_frameCount = 0`을 겁니다. 하나라도 빠뜨리면 이전 설정의 이미지가 새 설정과 섞여 유령처럼 남습니다.

## 9. 러시안 룰렛과 난수

경로를 고정 깊이에서 자르면 에너지가 사라져 어두워집니다. 러시안 룰렛은 **확률적으로 자르되 살아남은 경로를 그 확률만큼 밝게** 해서 기댓값을 보존합니다.

```glsl
if (bounce > 3u)
{
    float p = clamp(max(throughput.r, max(throughput.g, throughput.b)), 0.05, 0.95);
    if (randomFloat() > p) break;
    throughput /= p;
}
```

throughput이 이미 작아진 경로 — 즉 화면에 거의 기여하지 않을 경로 — 가 먼저 죽습니다. 처음 3바운스는 무조건 통과시키는데, 초반 바운스는 기여가 크고 여기서 노이즈를 만들면 손해이기 때문입니다.

난수는 PCG 해시입니다.

```glsl
uint pcg_hash(uint i)
{
    uint state = i * 747796405u + 2891336453u;
    uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
    return (word >> 22u) ^ word;
}

rngState = pcg_hash(launchID.y * 9277u + launchID.x * 1973u + ubo.frameCount * 26699u);
```

시드에 `frameCount`가 들어가는 게 중요합니다. 이게 없으면 매 프레임 같은 난수열을 써서 아무리 누적해도 노이즈 패턴이 그대로 고정됩니다.

## 10. 구현 시 걸리기 쉬운 지점

**리사이즈 때 리소스 셋도 다시 만들어야 합니다.** 렌더 타겟만 재생성하고 descriptor를 놔두면 이전 크기의 이미지에 기록하게 되어 GPU가 죽습니다.

```cpp
// 리소스 셋이 이전 렌더 타겟 이미지를 그대로 참조하고 있으므로 함께 재생성한다.
// (재생성하지 않으면 새 크기로 TraceRays할 때 이전 크기의 이미지에 기록해 GPU 크래시)
```

**정점 구조체는 scalar layout으로 다룹니다.** `vec3`를 나열하면 std430 패딩 규칙에 걸려 C++ 구조체와 오프셋이 어긋납니다. `GL_EXT_scalar_block_layout`을 켜고 정점은 아예 `float`을 나열합니다.

```glsl
struct Vertex {
    float px, py, pz;   // position
    float nx, ny, nz;   // normal
    float u, v;         // texCoord
    vec4  tangent;
};
```

명시적이고 못생겼지만, C++ 구조체와 1바이트도 어긋나지 않습니다. 셰이더-CPU 경계에서는 이 편이 안전합니다.

## 마치며

이 구현의 중심에는 하나의 원칙이 있습니다.

> **셰이더 재귀는 알고리즘을 표현하는 수단이 아니라 관리해야 할 하드웨어 자원이다.**

closest hit을 "표면 정보를 채워 돌아오는 함수"로 두고 경로 추적 전체를 raygen 루프에 모아두면, 요구 재귀 깊이가 2로 끝날 뿐 아니라 NEE·러시안 룰렛·Beer-Lambert 감쇠가 전부 "루프 안에 몇 줄 추가"로 해결됩니다. 구조가 맞으면 기능이 싸집니다.

남은 과제는 명확합니다.

- **디노이저** — SPP 1~4에서 쓸 만한 그림을 얻으려면 필수입니다. 별도 선형 누적 버퍼도 이때 같이 도입해야 합니다.
- **제대로 된 BSDF** — 지금 불투명 재질은 순수 램버시안입니다. `metallic`/`roughness`를 payload로 받아만 두고 쓰지 않고 있어서, GGX 기반 마이크로패싯으로 확장해야 합니다.
- **다중 광원과 광원 샘플링** — 현재는 점광원 하나를 구로 근사한 게 전부입니다.
- **TLAS 갱신** — `updateTLAS()`가 아직 비어 있습니다. 동적 씬을 하려면 `ALLOW_UPDATE` 플래그와 함께 채워야 합니다.

<!-- TODO: 렌더 결과 스크린샷 추가 (유리 드래곤 / 감쇠 on-off 비교 / 누적 프레임별 노이즈 감소) -->
