---
title: "MetalFX 업스케일링 — 스페이셜·템포럴 스케일러의 원리와 통합 포인트"
categories: [Graphics]
tags: [Metal, MetalFX, Upscaling, TAA, Apple, iOS, macOS]
toc: true
toc_sticky: true
wip: true
---

업스케일링은 이제 PC·콘솔 렌더러의 기본 장비가 되었습니다. DLSS, FSR, XeSS 같은 이름은 익숙한데, Apple 플랫폼의 답인 **MetalFX**는 상대적으로 덜 알려져 있습니다. 그런데 실제로 뜯어보면 입력도, 요구 조건도, 실수하는 지점도 다른 업스케일러와 거의 같습니다. Apple 스스로도 "다른 플랫폼에서 업스케일러를 이미 지원하는 엔진이라면 MetalFX 통합은 코드가 많이 필요하지 않다"고 말합니다.

이 글에서는 MetalFX가 어떤 구성으로 되어 있는지, 두 스케일러가 각각 **왜 그런 입력을 요구하는지**, 그리고 통합할 때 무엇을 틀리기 쉬운지를 정리합니다. API 나열보다 "그 프로퍼티가 왜 있는가"에 무게를 둡니다.

## 1. 업스케일링이 성립하는 이유

먼저 전제부터 짚어야 합니다. 업스케일링은 공짜가 아닙니다. 업스케일러 자체가 GPU 시간을 씁니다. 그런데도 이득이 되는 이유는 비용의 구조가 다르기 때문입니다.

- **셰이딩 비용은 픽셀 수에 비례합니다.** 1440p를 720p로 낮추면 픽셀 셰이딩·레이트레이싱·볼류메트릭 비용이 대략 4분의 1이 됩니다.
- **업스케일러 비용은 출력 해상도에만 묶인 고정 비용입니다.** 씬이 복잡하든 단순하든 거의 같은 시간이 듭니다.

그래서 MetalFX 문서의 정의는 정확히 이렇습니다.

> **"입력 해상도로 렌더링하고 업스케일하는 데 걸리는 시간이, 출력 해상도로 직접 렌더링하는 시간보다 짧아야 한다."**

이 부등식이 깨지면 업스케일링은 손해입니다. 셰이딩이 가벼운 씬(단순한 셰이더, 낮은 해상도 타깃)에서는 업스케일러 고정 비용이 오히려 더 클 수 있습니다. **업스케일링은 셰이딩 바운드인 씬에서만 이득**이라는 점을 기억해야 합니다.

아낀 시간을 어디에 쓸지는 앱의 선택입니다. 프레임레이트를 올리거나, 같은 프레임레이트에서 효과를 더 넣거나, 발열과 배터리를 아끼거나. Apple은 세 번째를 꽤 강조합니다. 모바일과 노트북이 주력인 플랫폼이니까요.

## 2. MetalFX의 구성

MetalFX는 2022년(iOS 16 / macOS 13)에 두 개의 스케일러로 출발했고, 이후 두 개의 새 효과가 붙었습니다.

| 효과 | 하는 일 | 필요한 입력 | 최소 OS |
|---|---|---|---|
| `MTLFXSpatialScaler` | 한 프레임만 보고 공간적으로 확대 | 컬러 | iOS 16 / macOS 13 |
| `MTLFXTemporalScaler` | 여러 프레임을 누적해 AA + 확대 | 컬러, 뎁스, 모션 벡터, 지터 | iOS 16 / macOS 13 |
| `MTLFXTemporalDenoisedScaler` | 레이트레이싱 결과를 디노이즈하면서 확대 | 위 + 노멀, 러프니스, 알베도 등 | macOS 26 (문서상 iOS 18) |
| `MTLFXFrameInterpolator` | 두 프레임 사이에 중간 프레임 생성 | 현재·이전 컬러, 뎁스, 모션 벡터 | iOS 26 / macOS 26 |

네 가지 모두 사용 패턴은 같습니다.

```
디스크립터를 만들고 포맷·크기를 채운다
  → 디스크립터에서 효과 인스턴스를 만든다 (비쌈, 시작 시 한 번)
  → 매 프레임: 입력 텍스처들을 프로퍼티에 꽂고 encodeToCommandBuffer:
```

인스턴스 생성이 비싸다는 점은 문서가 반복해서 강조합니다. **앱 시작 시 또는 디스플레이 해상도가 바뀔 때만** 만들고, 프레임마다 재사용해야 합니다. 이유는 뒤(5절)에서 다시 나옵니다.

프레임워크가 iOS 16부터 있다고 해서 모든 기기가 지원하는 건 아닙니다. 초기에는 M1 이상 Mac과 M1 iPad만 대상이었고, iPhone 지원은 WWDC23에서 추가되었습니다. 그래서 반드시 `supportsDevice:`로 확인하고 폴백 경로를 둬야 합니다.

## 3. 스페이셜 스케일러

한 장의 입력만 보고 크기를 키웁니다. 단일 프레임 정보만으로 새 샘플을 만들어야 하므로, **입력 자체가 이미 깨끗해야** 합니다.

> "가장 좋은 스페이셜 업스케일링 품질을 위해서는 컬러 입력이 안티에일리어싱되어 있고 노이즈가 없어야 한다. 노이즈와 에일리어싱은 에지 판정을 방해하고, 그것이 품질을 떨어뜨린다."

즉 스페이셜 스케일러는 AA를 **대신해 주지 않습니다.** 엔진의 AA(TAA, MSAA 등)가 끝난 결과를 받아서 키우는 물건입니다.

파이프라인 위치는 **톤매핑 직후**입니다. 디스크립터의 `colorProcessingMode`가 이 사실을 API로 드러냅니다.

| 모드 | 의미 |
|---|---|
| `MTLFXSpatialScalerColorProcessingModePerceptual` | 톤매핑 끝난 0~1 sRGB 입력. **성능 최상, 권장** |
| `MTLFXSpatialScalerColorProcessingModeLinear` | 리니어 컬러 공간 |
| `MTLFXSpatialScalerColorProcessingModeHDR` | HDR 컬러 공간 |

에지 판정은 사람 눈이 보는 대로(지각적 공간에서) 하는 게 가장 잘 맞고, 8비트 sRGB는 대역폭도 가장 작습니다. 그래서 perceptual 모드가 기본 권장입니다.

```objc
MTLFXSpatialScalerDescriptor *desc = [MTLFXSpatialScalerDescriptor new];
desc.inputWidth  = 960;  desc.inputHeight  = 540;
desc.outputWidth = 1920; desc.outputHeight = 1080;
desc.colorTextureFormat  = MTLPixelFormatBGRA8Unorm_sRGB;
desc.outputTextureFormat = MTLPixelFormatBGRA8Unorm_sRGB;
desc.colorProcessingMode = MTLFXSpatialScalerColorProcessingModePerceptual;  // 톤매핑 끝난 0~1 sRGB

if (![MTLFXSpatialScalerDescriptor supportsDevice:device]) { /* 폴백 */ }
id<MTLFXSpatialScaler> spatial = [desc newSpatialScalerWithDevice:device];
if (!spatial) { /* 폴백 */ }

// 매 프레임
spatial.colorTexture  = tonemappedColor;
spatial.outputTexture = upscaledColor;           // private storage 필수
[spatial encodeToCommandBuffer:cmd];
```

**쓸 곳** — 이미 잘 튜닝된 AA가 있거나, 모션 벡터·뎁스를 뽑을 수 없는 렌더러(UI 위주 앱, 2D, 비디오, 간단한 3D). Apple의 표현을 빌리면 "필요한 입력이 없거나 이미 좋은 AA가 있다면 스페이셜을 고려하라"입니다.

## 4. 템포럴 스케일러

여기가 본론입니다. 템포럴 스케일러는 **TAA와 업스케일링을 하나로 합친 것**입니다. 원리는 TAA와 같습니다.

```
매 프레임 카메라를 서브픽셀만큼 살짝 흔들어(jitter) 그린다
  → 이전 프레임들의 결과를 모션 벡터로 현재 위치에 되돌린다(reproject)
  → 뎁스로 "이전 프레임에는 가려져 있었던 곳"을 판별해 히스토리를 버린다
  → 살아남은 히스토리와 현재 샘플을 누적한다
```

한 픽셀에 대해 여러 프레임에 걸쳐 서로 다른 서브픽셀 위치의 샘플이 쌓입니다. 그 샘플들을 **출력 해상도 격자에 다시 놓으면** 입력보다 많은 정보를 가진 이미지가 됩니다. TAA가 "시간으로 슈퍼샘플링"을 한다면, 템포럴 업스케일링은 "시간으로 해상도를 복원"하는 셈입니다.

그래서 요구하는 입력 하나하나가 위 단계에 대응합니다. **지터**는 샘플이 격자 밖 위치를 커버하게 하고, **모션 벡터**는 리프로젝션에, **뎁스**는 가림 판정에 쓰입니다. 이 셋 중 하나라도 부정확하면 고스팅·떨림·번짐이 나옵니다. 아래는 각각을 어떻게 넣어야 하는지입니다.

### 4-1. 지터

Apple의 권장은 구체적입니다.

> "2배 업스케일링의 경우 **Halton (2,3) 시퀀스로 32개의 지터**를 사용하기를 권장한다. 출력 픽셀 하나당 약 8개의 샘플이 된다."

숫자의 근거를 풀면 이렇습니다. 2배 업스케일이면 입력 픽셀 하나가 출력 픽셀 4개를 담당합니다. 출력 픽셀당 8개 샘플을 모으려면 입력 픽셀당 32개가 필요합니다. 즉 **지터 개수 ≈ 8 × (스케일 비율)²** 이고, 1.5배면 18개 정도가 됩니다.

지터 오프셋은 항상 **[-0.5, 0.5] 픽셀** 범위이고, 정해진 개수 안에서 서로 달라야 합니다(같은 위치를 두 번 찍으면 그만큼 정보가 낭비됩니다). Halton처럼 저불일치(low-discrepancy) 시퀀스를 쓰는 이유는 적은 개수로도 픽셀 안을 고르게 덮기 때문입니다.

```objc
static float halton(int index, int base) {
    float f = 1, r = 0;
    for (int i = index; i > 0; i /= base) { f /= base; r += f * (i % base); }
    return r;
}

// 프레임 n의 지터. 픽셀 단위, [-0.5, 0.5)
int n = frameIndex % 32 + 1;
simd_float2 jitter = simd_make_float2(halton(n, 2) - 0.5f, halton(n, 3) - 0.5f);
```

이 지터를 렌더링에 적용하는 방법은 TAA와 같습니다. 픽셀 오프셋을 NDC 단위(`2 * jitter / 렌더해상도`)로 바꿔 투영 행렬의 x·y 평행이동 성분에 더합니다. 그러면 씬 전체가 서브픽셀만큼 밀려 그려집니다.

그리고 **같은 값**을 `jitterOffsetX/Y`에 넣습니다. MetalFX는 이 값으로 샘플이 픽셀 안 어디에 찍혔는지 알고, 출력 격자에 되돌려 놓습니다. 단위는 픽셀이고 좌표계는 Metal의 텍스처 좌표(좌상단 원점, y 아래 방향)입니다.

여기서 흔한 함정이 **부호**입니다. NDC는 y가 위로 향하고 텍스처 좌표는 아래로 향하며, "투영을 +δ 밀었다"와 "샘플이 +δ 위치에 찍혔다"는 방향이 반대입니다. Apple의 WWDC25 샘플 코드도 `jitterOffsetY = -pixelJitter.y`처럼 한 축을 뒤집어 넘깁니다. 엔진마다 규약이 달라서 공식을 외우는 것보다 **검증법**을 아는 게 낫습니다. 카메라를 완전히 정지시키고 지터만 켠 상태에서, 출력이 미세하게 떨리거나 흐려지면 부호가 틀린 것이고, 정지 화면이 또렷하게 수렴하면 맞은 것입니다.

### 4-2. 모션 벡터

문서의 규약은 명확합니다.

> "모션 벡터는 **렌더 해상도의 픽셀 공간**에서, **현재 프레임 위치에서 이전 프레임 위치를 향하는 방향**이어야 한다."

그리고 예시를 하나 줍니다. 좌상단이 (0,0)인 Metal 좌표계에서 오브젝트가 **오른쪽 아래로 10픽셀 움직였으면 모션 벡터는 (-10, -10)** 입니다. 현재 위치에서 "이전에 어디 있었는지"를 가리키기 때문입니다. 리프로젝션은 "현재 픽셀이 이전 프레임의 어디에서 왔는가"를 묻는 연산이니 이 방향이 자연스럽습니다.

문제는 엔진마다 모션 벡터를 저장하는 방식이 다르다는 점입니다. NDC 단위인 경우도 있고, UV 단위도 있고, 부호가 반대(이전→현재)인 경우도 많습니다. 이걸 흡수하는 장치가 `motionVectorScaleX/Y`입니다. MetalFX는 텍스처 값에 이 스케일을 곱해서 픽셀 단위로 해석합니다. **단위 변환과 부호 반전을 모두 이 스케일 하나에 담을 수 있습니다.**

WWDC26 세션의 셰이더는 카메라 모션 벡터를 이렇게 만듭니다.

```cpp
// Metal Shading Language — 정적 지오메트리의 카메라 모션 벡터
float4 clipCurrent  = viewProjCurrent  * float4(worldPos, 1.0);
float4 clipPrevious = viewProjPrevious * float4(worldPos, 1.0);
float2 ndcCurrent   = clipCurrent.xy  / clipCurrent.w;
float2 ndcPrevious  = clipPrevious.xy / clipPrevious.w;

float2 motion = ndcPrevious - ndcCurrent;          // 현재 → 이전, NDC 단위

// 지터 제거: 두 프레임의 서브픽셀 오프셋 차이를 빼준다
motion -= jitterPrevious - jitterCurrent;
```

이 값은 NDC 단위(폭 2)이므로 픽셀로 바꾸려면 x에 `W/2`를 곱합니다. y는 NDC가 위쪽 양수, 픽셀이 아래쪽 양수라 `-H/2`입니다.

```objc
scaler.motionVectorScaleX =  (float)renderW / 2;
scaler.motionVectorScaleY = -(float)renderH / 2;
```

반대로 엔진이 `ndcCurrent - ndcPrevious`(이전→현재)로 저장한다면 스케일을 `(-W/2, +H/2)`로 주면 됩니다. WWDC22 세션의 예시가 정확히 이 경우로, 1080p에서 `(-960, 540)`을 넣는 장면이 나옵니다. 셰이더를 고칠 필요가 없습니다.

위 코드의 마지막 줄, **지터 제거**도 중요합니다. 두 프레임의 투영이 각각 다른 서브픽셀만큼 밀려 있으니 그대로 빼면 모션 벡터에 지터 차이가 섞입니다. 최대 1픽셀 오차인데, 이 정도로도 **에지가 지글거립니다**(WWDC26에서 직접 지적한 문제입니다). 움직이는 지오메트리는 이전 프레임의 월드 위치를 정점마다 따로 들고 있어야 진짜 모션 벡터가 나옵니다.

### 4-3. 뎁스

뎁스는 "이전 프레임에 가려져 있다가 이번에 드러난 영역"을 찾는 데 쓰입니다(disocclusion). 이 영역은 히스토리가 없으니 현재 샘플만 써야 하고, 이걸 못 잡으면 고스팅이 생깁니다. 전경 에지의 AA 우선순위를 정하는 데도 쓰입니다.

필요한 설정은 `depthReversed` 하나입니다. Reversed-Z(가까울수록 1, 멀수록 0)를 쓰는 엔진은 반드시 켜야 합니다. 부동소수점 정밀도 때문에 요즘 엔진은 대부분 Reversed-Z이므로 거의 항상 `YES`일 겁니다.

### 4-4. 노출

템포럴 스케일러는 **톤매핑 전의 HDR 리니어 컬러**를 받습니다(4-9절 참고). 그런데 히스토리 누적과 에지 판정은 최종적으로 눈에 보이는 밝기 기준으로 해야 정확합니다. 그래서 스케일러가 "이 HDR 값이 화면에서 얼마나 밝게 보일지"를 알아야 하고, 그 정보가 노출입니다.

두 가지 방법이 있습니다.

- **`exposureTexture`** — 1×1 `R16Float` 텍스처. (0,0) 텍셀의 R 채널을 노출값으로 읽어 입력 컬러에 곱합니다. 엔진의 자동 노출이 GPU에서 이 값을 만들고 있다면 그대로 넘기면 됩니다. Apple도 "GPU에서 생성해 텍스처에 쓰는 것"을 성능상 권장합니다.
- **`autoExposureEnabled`** — 디스크립터에서 켜면 MetalFX가 프레임마다 스스로 계산합니다. 이 경우 `exposureTexture`는 무시됩니다.

WWDC23의 조언은 단순합니다. 1×1 노출 텍스처를 만들 수 있으면 그걸 쓰고, 아니면 자동 노출을 켜서 품질이 나아지는지 보라는 것입니다.

노출값이 톤매퍼와 안 맞는지 확인하는 도구도 있습니다. 환경 변수 `MTLFX_EXPOSURE_TOOL_ENABLED`를 켜면 업스케일러가 프레임 위에 회색 체커보드를 그리는데, **노출이 맞으면 일정한 중간 회색**으로 보이고 틀리면 너무 밝거나 어둡거나 게임 중에 밝기가 변합니다.

`preExposure`는 별개입니다. 입력 컬러에 이미 고정 상수를 곱해 놓은 특수한 경우에 그 값을 알려주는 용도이고, 문서 스스로 "보통은 설정할 필요가 없다"고 합니다.

### 4-5. 밉 바이어스

렌더 해상도를 낮추면 텍스처 샘플링 밉 레벨도 그만큼 낮은 해상도의 것이 선택됩니다. 출력은 고해상도인데 텍스처 디테일은 저해상도인 상태가 되죠. 그래서 업스케일링을 쓰는 렌더러는 **머티리얼 셰이더의 밉 바이어스를 음수로 걸어** 더 선명한 밉을 강제로 고릅니다.

Apple의 권장 공식은 다음입니다.

```
mipBias = log2(렌더 해상도 폭 / 출력 해상도 폭) - 1
```

2배 업스케일이면 -2, 1.5배면 약 -1.58입니다. 뒤의 `-1`이 TAA 누적을 감안한 추가 선명화입니다.

단, 이건 **출발점**입니다. 바이어스가 강하면 회로 기판 패턴처럼 고주파 텍스처에서 깜빡임이 생기고, WWDC22 데모는 -2 대신 -1로 완화하는 장면을 보여줍니다. 텍스처 종류에 따라 조정할 여지를 남겨두는 게 좋습니다. 그리고 이 항목은 "엔진이 머티리얼 셰이더의 LOD를 직접 제어할 수 있어야 한다"는 뜻이기도 합니다. WWDC23은 이를 통합 전제 조건으로 명시합니다.

### 4-6. 히스토리 리셋

`reset`을 `YES`로 넘기면 스케일러가 이전 프레임 데이터를 버립니다. **첫 프레임, 씬 컷, 급격한 카메라 이동**이 여기에 해당합니다. 컷 직후에 이전 씬의 히스토리가 몇 프레임 남아 있으면 그게 고스팅으로 보입니다. Apple은 WWDC23에서 "카메라 컷에서 히스토리 리셋을 잊지 말라"를 두 번 반복해서 말했습니다.

### 4-7. 리액티브 마스크

모션 벡터와 뎁스는 불투명 지오메트리만 기록합니다. **파티클·알파 블렌딩 효과·불꽃**처럼 뎁스와 모션에 안 쓰이는 것들은 스케일러 입장에서 "움직였다는 정보 없이 색만 바뀐" 픽셀입니다. 높은 스케일 비율에서 이런 픽셀은 배경과 섞이거나 고스팅이 남습니다.

리액티브 마스크(iOS 17.4 / macOS 14.4)는 이런 픽셀을 표시하는 텍스처입니다. 값의 의미는 다음과 같습니다.

| 값 | 의미 |
|---|---|
| 0.0 | 평소대로 처리 |
| 1.0 | 이 픽셀은 히스토리를 무시하고 현재 프레임만 쓴다 |
| (0, 1) | 비례해서 섞는다 |

디스크립터에서 `reactiveMaskTextureEnabled`와 포맷을 켜고, 프레임마다 `reactiveMaskTexture`에 넣습니다. 보통 반투명 패스에서 머티리얼 종류별로 반응도 값을 별도 렌더 타깃에 써서 만듭니다.

Apple이 붙인 주의사항 두 가지가 중요합니다. **"입력 해상도를 올리는 게 불가능할 때만 쓰라"**, 그리고 **"다른 업스케일러용으로 튜닝한 마스크를 그대로 쓰지 말라"**. 다른 업스케일러에서 문제였던 영역이 MetalFX에서는 멀쩡할 수 있고, 그 영역의 히스토리를 괜히 버리면 오히려 품질이 나빠집니다.

### 4-8. 동적 해상도

프레임 시간에 따라 렌더 해상도를 매 프레임 바꾸는 동적 해상도(DRS)도 지원합니다. 디스크립터의 `inputContentPropertiesEnabled`를 켜고 `inputContentMinScale`/`MaxScale`로 범위를 정한 뒤, 프레임마다 `inputContentWidth/Height`에 **이번 프레임의 실제 렌더 크기**를 넣습니다. 입력 텍스처 자체는 최대 크기로 만들어 두고 일부 영역만 쓰는 방식입니다.

지원 범위는 기기마다 다르므로 `supportedInputContentMinScale:` / `MaxScale:`로 질의해야 합니다. WWDC23 기준 최대 3배까지 지원하지만, WWDC25의 권장은 명확합니다.

> "최상의 품질을 위해, 필요하지 않다면 최대 스케일을 **2배보다 높게 설정하지 말라**."

### 4-9. 파이프라인 위치

템포럴 스케일러는 **포스트 프로세싱 전**에 들어갑니다.

> "템포럴 AA와 업스케일링은 어떤 포스트 프로세싱 효과보다도 앞서 실행해야 한다. 그 효과들이 업스케일링 결과를 방해하기 때문이다."

이유는 두 가지입니다. 첫째, 모션 블러·DOF·블룸·필름 그레인은 픽셀을 이웃과 섞거나 노이즈를 더하는 연산이라, 히스토리 누적과 에지 판정을 망칩니다. 둘째, 포스트 프로세싱은 출력 해상도에서 돌아야 최종 화질이 나오는데, 업스케일 후에 돌리면 자연스럽게 그렇게 됩니다. UI도 마찬가지로 업스케일 후에 그립니다.

```
지오메트리/라이팅 (렌더 해상도, 지터 적용, HDR 리니어)
  → 모션 벡터, 뎁스 (렌더 해상도)
  → [MetalFX 템포럴 스케일러]  ← 노출 텍스처
  → 포스트 프로세싱 (출력 해상도): 블룸, 모션 블러, DOF, 톤매핑
  → UI
```

컬러 포맷은 HDR 리니어이므로 `MTLPixelFormatRGBA16Float`가 표준이고, 모션 벡터는 `MTLPixelFormatRG16Float`, 뎁스는 `MTLPixelFormatDepth32Float`가 Apple 예시의 조합입니다.

### 4-10. 정리: 코드로 보면

```objc
// 시작 시 한 번
MTLFXTemporalScalerDescriptor *desc = [MTLFXTemporalScalerDescriptor new];
desc.inputWidth  = 1280; desc.inputHeight  = 720;
desc.outputWidth = 2560; desc.outputHeight = 1440;
desc.colorTextureFormat  = MTLPixelFormatRGBA16Float;   // HDR 리니어, 톤매핑 전
desc.depthTextureFormat  = MTLPixelFormatDepth32Float;
desc.motionTextureFormat = MTLPixelFormatRG16Float;
desc.outputTextureFormat = MTLPixelFormatRGBA16Float;
desc.autoExposureEnabled = NO;                          // 1x1 노출 텍스처를 직접 넘김
desc.inputContentPropertiesEnabled = YES;               // 동적 해상도
desc.inputContentMinScale = 1.0f;
desc.inputContentMaxScale = 2.0f;

if (![MTLFXTemporalScalerDescriptor supportsDevice:device]) { /* 폴백 */ }
id<MTLFXTemporalScaler> scaler = [desc newTemporalScalerWithDevice:device];
if (!scaler) { /* 폴백 */ }

// 출력 텍스처는 스케일러가 요구하는 usage를 포함하고 private 이어야 한다
MTLTextureDescriptor *outDesc =
    [MTLTextureDescriptor texture2DDescriptorWithPixelFormat:MTLPixelFormatRGBA16Float
                                                       width:2560 height:1440 mipmapped:NO];
outDesc.usage = scaler.outputTextureUsage;
outDesc.storageMode = MTLStorageModePrivate;
```

```objc
// 매 프레임
scaler.reset             = isFirstFrame || isSceneCut;
scaler.colorTexture      = sceneColorHDR;        // 지터 적용된 렌더 해상도 컬러
scaler.depthTexture      = sceneDepth;
scaler.motionTexture     = motionVectors;
scaler.exposureTexture   = exposure1x1;
scaler.outputTexture     = upscaledColor;
scaler.depthReversed     = YES;
scaler.inputContentWidth  = renderW;              // 이번 프레임의 실제 렌더 크기
scaler.inputContentHeight = renderH;
scaler.jitterOffsetX     = jitter.x;              // 픽셀 단위, 부호 규약 확인
scaler.jitterOffsetY     = jitter.y;
scaler.motionVectorScaleX =  (float)renderW / 2;  // NDC(현재→이전) → 픽셀
scaler.motionVectorScaleY = -(float)renderH / 2;
[scaler encodeToCommandBuffer:cmd];
```

C++ 엔진이라면 metal-cpp에 MetalFX 바인딩이 포함되어 있어(WWDC23부터) 같은 호출을 `scaler->setJitterOffsetX(...)`, `scaler->encodeToCommandBuffer(cmd)` 식으로 그대로 쓸 수 있습니다. Apple은 Objective-C 헤더 대비 측정 가능한 오버헤드가 없다고 밝히고 있습니다.

## 5. 통합할 때 틀리기 쉬운 것들

입력의 의미를 맞추는 것 외에, API 계약 수준에서 걸리는 지점들입니다.

- **텍스처 usage 요구사항.** 스케일러는 `colorTextureUsage`, `depthTextureUsage`, `motionTextureUsage`, `outputTextureUsage` 프로퍼티로 각 텍스처에 **최소한 켜져 있어야 하는 `MTLTextureUsage` 비트**를 알려줍니다. 텍스처 디스크립터를 만들 때 이 비트를 포함해야 합니다. 더 켜는 건 상관없습니다.
- **출력 텍스처는 `MTLStorageModePrivate` 스토리지.** 문서에 명시된 계약입니다.
- **같은 텍스처 객체를 매 프레임 넘길 필요는 없습니다.** MetalFX는 인스턴스 동일성을 추적하지 않습니다. 디스크립터에 적은 포맷·크기와 일치하면 어떤 텍스처든 됩니다. 프레임마다 다른 링 버퍼 슬롯을 꽂아도 됩니다.
- **생성은 비쌉니다.** 내부 업스케일러를 컴파일하기 때문입니다. 기본값(`requiresSynchronousInitialization = NO`)에서는 인스턴스를 빨리 돌려주고 **더 빠른 업스케일러를 백그라운드에서 컴파일**합니다. 그동안은 느린 임시 업스케일러로 동작하며, 컴파일이 끝나면 자동으로 교체됩니다. **출력 화질은 두 경로가 동일**하고 속도만 다릅니다. 로딩 화면에서 미리 만들어 두고 싶으면 `YES`로 켜서 동기적으로 컴파일시키면 됩니다.
- **거짓 의존성(false dependency).** 서로 의존하지 않는 두 패스가 같은 리소스를 읽기·쓰기로 바인딩하면 Metal이 불필요한 동기화를 걸고, 이게 MetalFX 성능을 깎습니다. 특히 프레임 사이의 거짓 의존성을 주의하라는 게 WWDC22의 지적입니다.
- **언트랙 리소스는 `fence`로.** 해저드 트래킹을 꺼둔(untracked) 리소스를 입력으로 쓰면 스케일러의 `fence` 프로퍼티에 펜스를 넘겨 동기화해야 합니다.

## 6. 그 너머: 디노이즈 업스케일러, 프레임 보간, 그리고 최근 변화

### 6-1. 템포럴 디노이즈 업스케일러

레이트레이싱을 쓰는 렌더러를 위한 변형입니다. 픽셀당 레이 수를 줄여 노이즈가 낀 결과를 넣으면 **디노이징과 업스케일링을 한 패스에서** 처리합니다. Apple의 포지셔닝은 "디노이징을 써서 레이를 줄이는 것이 레이트레이싱의 최선의 품질·성능 트레이드오프"입니다.

기존 템포럴 스케일러 입력에 **노이즈 없는 보조 버퍼**들이 추가됩니다. G-버퍼가 있는 엔진이라면 대부분 이미 가진 것들입니다.

| 입력 | 비고 |
|---|---|
| 노멀 | 월드 공간 권장 |
| 디퓨즈 알베도 | 디노이징에 **가장 강한 신호** |
| 러프니스 | 리니어 값 |
| 스페큘러 알베도 | 프레넬 항이 포함된, 노이즈 없는 스페큘러 근사 |
| 스페큘러 히트 거리 (선택) | 1차 표면에서 2차 바운스까지의 레이 길이 |
| 디노이즈 강도 마스크 (선택) | 디노이징이 필요 없는 영역(하늘 등) 표시 |
| 투명도 오버레이 (선택) | 알파로 블렌딩, 업스케일만 하고 디노이즈는 하지 않을 것(파티클, 포그, 볼류메트릭) |

위치는 템포럴 스케일러와 같이 메인 렌더링 직후, 포스트 프로세싱 직전입니다. 흔한 문제는 **입력이 너무 노이지한 것**이고, 해법은 디노이저가 아니라 샘플링 쪽에 있습니다. NEE, 중요도 샘플링, 기여하는 광원 위주 샘플링, 그리고 **상관된 난수를 피하는 것**입니다. 공간적·시간적 상관 모두 아티팩트를 만듭니다.

거울과 유리는 별도 요령이 필요합니다. 거울에는 반사된 지오메트리의 알베도·노멀·러프니스를 기록하고, 유리는 프레넬로 반사·굴절 속성을 섞어 넣는 "1차 표면 대체(primary surface replacement)" 기법을 씁니다. 디노이저 입장에서 "이 픽셀의 진짜 표면은 무엇인가"를 알려주는 것입니다.

### 6-2. 프레임 보간

두 렌더 프레임(N-1, N)과 모션 벡터·뎁스를 받아 **그 사이의 중간 프레임을 생성**합니다. 입력 두 프레임마다 한 프레임을 더 만드니 프레임레이트가 약 두 배가 됩니다. 디스크립터의 `scaler` 프로퍼티에 사용 중인 템포럴 스케일러를 연결하면 업스케일러가 이미 계산한 정보를 재활용합니다.

위치는 **톤매핑 후**, 즉 UI를 그리는 시점 근처입니다. 그래서 UI 처리가 핵심 설계 문제가 되고, Apple은 세 가지 방식을 제시합니다.

1. **합성된 UI** — UI 없는 프레임 N, UI 있는 프레임 N, 이전 프레임 N-1을 넘깁니다(`uiTextureComposited`). 가장 쉽습니다.
2. **오프스크린 UI** — UI를 별도 텍스처(`uiTexture`)로 넘기면 보간된 프레임 위에 얹어줍니다.
3. **매 프레임 UI** — 보간 프레임에도 UI를 직접 그립니다. 코드 변경이 가장 크지만 UI까지 부드러워집니다.

`deltaTime`, 니어/파 플레인, FOV 같은 카메라 파라미터를 정확히 넘겨야 합니다. 보간기가 모션 벡터 길이를 실제 시뮬레이션 시간에 맞게 조정하는 데 쓰이고, 틀리면 **가려짐 영역에 아티팩트**가 생깁니다. 입력 프레임레이트는 **최소 30fps**를 권장합니다. 그리고 생성한 프레임을 일정한 간격으로 제시(present)하는 **페이싱**이 별도의 난제인데, Metal HUD의 프레임 간격 히스토그램이 두 버킷 이내로 정리되면 맞은 것입니다.

### 6-3. Metal 4와 그 이후

Metal 4(iOS 26 / macOS 26)에서는 `MTL4FXTemporalScaler`처럼 `MTL4FX` 접두어가 붙은 프로토콜이 추가되어 Metal 4 커맨드 버퍼에 인코딩할 수 있습니다. 지원 여부는 `supportsMetal4FX:`로 확인하고, 생성은 `newTemporalScalerWithDevice:compiler:`로 Metal 4 컴파일러를 넘겨 합니다. 프로퍼티 계약은 기존과 같은 `...Base` 프로토콜을 공유합니다.

WWDC26에서는 템포럴 업스케일러가 **재설계**되었습니다. Neural Engine과 M5 Pro/Max의 Neural Accelerator를 함께 사용하는 신경망 업스케일러로, "훨씬 낮은 렌더 해상도에서도 세부를 복원한다"는 것이 Apple의 설명입니다. API 측면에서는 현재 베타 문서 기준으로 다음이 추가되었습니다.

- **서브렉트 처리** — `colorContentOffsetX/Y`, `outputOffsetX/Y` 등으로 텍스처의 일부 영역만 입력·출력으로 씁니다. 동적 해상도를 아틀라스식으로 구현할 때 유용합니다.
- **지터 포함 모션 벡터** — `jitteredMotionVectorsEnabled`를 켜면 4-2절의 지터 제거를 엔진이 하지 않아도 MetalFX가 `jitterOffset`으로 직접 빼줍니다.
- **출력 해상도 모션 벡터** — `outputResolutionMotionVectorsEnabled`. 모션 벡터를 출력 해상도로 넘기는 엔진용입니다.
- **왜곡 필드** — 프레임 보간기에 배럴 왜곡 같은 포스트 프로세싱 왜곡을 알려주는 `distortionTexture`.

이 항목들은 글 작성 시점(iOS 27 / macOS 27 베타)의 문서 기준이므로 정식 출시 후 이름이 바뀔 수 있습니다.

## 7. 다른 업스케일러에서 넘어온다면

입력 세트를 보면 DLSS나 FSR 2 계열과 사실상 같습니다. 지터 적용된 HDR 컬러, 뎁스, 모션 벡터, 지터 오프셋, 노출, 리액티브 마스크. 그래서 이미 이 중 하나를 붙인 엔진은 MetalFX 백엔드를 추가하는 일이 대부분 **규약 매핑**입니다.

- 모션 벡터 방향·단위 차이는 `motionVectorScaleX/Y`로 흡수합니다(4-2절).
- 지터 부호는 정지 화면 수렴 테스트로 확인합니다(4-1절).
- 리액티브 마스크는 **다시 튜닝**합니다. 다른 업스케일러 기준으로 만든 마스크를 그대로 쓰지 말라는 것이 Apple의 명시적 조언입니다(4-7절).
- 스케일 비율은 2배를 상한으로 시작합니다(4-8절).

눈에 띄는 차이는 **노출할 튠 파라미터가 거의 없다**는 점입니다. 샤프니스 슬라이더도, 품질 프리셋도 없습니다. 입력을 정확히 넣는 것이 튜닝의 전부입니다. Apple은 이걸 장점으로 내세우고, 실제로 통합 관점에서는 그렇습니다. 조절할 게 없으니 틀릴 곳도 입력밖에 없습니다.

## 마치며

MetalFX 통합에서 어려운 부분은 API가 아닙니다. 디스크립터 하나, 프로퍼티 열 몇 개, `encode` 한 줄이 전부입니다. 어려운 건 **입력의 의미를 정확히 맞추는 일**이고, 그건 TAA를 제대로 구현하는 일과 같은 난이도입니다.

정리하면 이렇습니다.

> **템포럴 업스케일링은 시간에 걸쳐 흩어진 서브픽셀 샘플을 출력 격자에 되돌려 놓는 연산이다. 지터는 샘플을 흩고, 모션 벡터는 되돌리고, 뎁스는 되돌릴 수 없는 곳을 가려낸다. 이 셋의 부호와 단위가 맞으면 나머지는 MetalFX가 한다.**

## 참고 자료

- [MetalFX — Apple Developer Documentation](https://developer.apple.com/documentation/metalfx)
- [Applying temporal antialiasing and upscaling using MetalFX (샘플 코드)](https://developer.apple.com/documentation/metalfx/applying-temporal-antialiasing-and-upscaling-using-metalfx)
- [WWDC22 — Boost performance with MetalFX Upscaling](https://developer.apple.com/videos/play/wwdc2022/10103/)
- [WWDC23 — Bring your game to Mac, Part 3: Render with MetalFX](https://developer.apple.com/videos/play/wwdc2023/10125/)
- [WWDC24 — Port advanced games to Apple platforms](https://developer.apple.com/videos/play/wwdc2024/10089/)
- [WWDC25 — Go further with Metal 4 games](https://developer.apple.com/videos/play/wwdc2025/211/)
- [WWDC26 — Build real-time neural rendering pipelines with Metal](https://developer.apple.com/videos/play/wwdc2026/359/)
