---
title: "About"
permalink: /about/
toc: true
toc_sticky: true
---

멀티 플랫폼 게임 엔진을 만들어 온 그래픽스 개발자 **이창진**입니다.

성능 향상에 최선을 다하고, 동료가 편하게 쓸 수 있는 시스템을 만들기 위해 노력합니다.
자체 엔진의 렌더러를 밑바닥부터 설계·개발한 경험을 바탕으로, 현재는 Unity 커스텀 렌더러를 만들고 있습니다.

## 기술 스택

- **언어**: C++, C, Objective-C++
- **그래픽스 API**: Vulkan, Metal, OpenGL ES 3.0 (GLSL, HLSL)
- **엔진 / 툴**: Unity, RenderDoc

## 경력

### Com2uS — 렌더팀 · Unity 커스텀 렌더러 개발 (2026.01 ~ 현재)

- Unity 기반 커스텀 렌더러 제작

### Com2uS — 자체 게임 엔진 개발 (2020.05 ~ 2025.12)

멀티 플랫폼에서 동작하는 자체 게임 엔진의 렌더링 파트를 개발했습니다.

- **Forward 렌더러 개발**
  - Forward+ rendering (tile-based rendering, z-binning)
  - Batching system 개발 (SRP-batcher, static batching, GPU instancing)
  - RenderDoc을 이용한 디버깅 / 최적화
  - GPU resource 관리 및 개발
- **Vulkan / Metal / OpenGL ES 3.0 wrapping API 제작**
- **Shader variant system 개발**
- **Render Pass 설계** — OpaquePass, TransparentPass, UIPass, ShadowCastPass 등
- **각종 rendering techniques 적용**
  - 직접광 (directional / point / spot light)
  - 간접광 (global illumination, image based lighting) 및 shadow 구현
  - GPU skinning
  - Post processing (bloom, sharpen, tone mapping 등)
  - Parallel command buffer recording

## 학력 · 논문

- **서울대학교** 전기정보공학부 그래픽스 연구실 — 석사 (2018.08 ~ 2020.08)
- **고려대학교** 전기전자공학부 · 바이오의공학부 이중전공 — 학사 (2012.02 ~ 2018.08)

**논문**

- *Tight Normal Cone Merging for Efficient Collision Detection of Thin Deformable Objects* — EUROGRAPHICS 2021, 2저자
- 변형 가능한 모델의 충돌 탐지를 위한 효율적인 법선 원뿔 컬링 방법 — KCGS 2020, 1저자
- 실제적인 면 의류를 위한 가변형 위치 기반 다이나믹스 — KCGS 2019, 2저자

## 연락처

- **Email**: [ckdwls93@gmail.com](mailto:ckdwls93@gmail.com)
- **GitHub**: [github.com/OSgoodYZ](https://github.com/OSgoodYZ)
- **LinkedIn**: [linkedin.com/in/osg00d](https://www.linkedin.com/in/osg00d)
