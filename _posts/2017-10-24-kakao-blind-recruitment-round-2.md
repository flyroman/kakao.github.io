---
layout: post
title: '카카오 신입 공채 2차 코딩 테스트 문제 해설'
author: jesse.ha
date: 2017-10-24 10:00
tags: [blind-recruitment,coding,rest,json]
image: /files/covers/employment_cover.png
---

지난 10월 14일(토) 오후 2시부터 10시까지 8시간 동안 온라인 2차 코딩 테스트가 있었습니다. 1차 코딩 테스트와도 사뭇 다른 형식이라 신선했다는 의견도 있고, 당황한 지원자도 있었을 텐데요. 2차 문제에는 어떤 의도가 숨어있는지 살펴보겠습니다.

# 추가적으로 어떤 테스트를 해야할까?
1차 코딩 테스트는 어려운 알고리즘 문제가 아닌 자료구조, 알고리즘 등의 전산학 기초에 대해 충분히 학습하였다면 누구나 풀 수 있을만한 "구현" 위주의 문제들로 구성을 하였고, 총 7문제 중 4문제 이상 푼 지원자들에게 2차 기회가 주어졌습니다. 즉, 기본적인 구현 역량은 보유하고 있다고 할 수 있습니다. 그렇다면 추가적으로 테스트 해야하는 것은 무엇일까요?

온라인 2차 코딩 테스트 문제 출제 위원회에서는,
- 탄탄한 기본기를 바탕으로 새로운 것을 빠르게 습득하는 역량
- 요구사항을 꼼꼼하게 분석하고, 트레이드오프를 감안하여 디자인하여 구현하는 역량
- 결과를 모니터링하며 점진적으로 개선해나가는 역량
을 테스트할 수 있도록 2차 문제에 녹여내고자 하였습니다.

문제에 어떤 장치들이 있었는지 살펴볼까요?

# 이미지 특징값을 수집하는 Crawler 구현
## REST API, JSON Parsing을 이용하라!

2차 테스트 문제는 우선 토큰을 발급받고, 웰컴 서버로부터 5개 카테고리의 Seed URL을 받아온 후 각 카테고리의 문서를 읽어 그 안에 있는 이미지들을 가져와 특징값을 추출하여 서버에 저장/삭제하는 문제입니다.

쉽게 구현할 수 있는 수준의 요구사항을 주되, 자주 접해보지 않았을 (것으로 추정되는) REST API와 JSON Parsing을 포함시켜 단 시간에 학습하여 풀도록 하였습니다. 이 두가지 키워드는 언어별 편차가 심한 점을 감안하여 사전에 미리 제공하였습니다.

## 반복적인 모니터링과 최적화를 유도: 제한된 토큰 유효시간
 토큰의 유효기간은 10분으로 설정하고, 점수 현황판을 제공했습니다. 8시간 동안 스스로 최적화를 하고 한 번만 결과를 제출하도록 하면 일부 지원자는 반복 과정을 통해 최적화를 하지 않을 것 같았습니다. 그래서 다른 사람의 점수를 보고 계속해서 최적화를 수행할 수 있도록 토큰을 10분으로 제한하였습니다. 수행하는 도중에 문제점이나 병목을 찾으려면 적절히 로그도 남겨야 하고 모니터링도 자연스럽게 하게 될 것이라 생각했습니다.

## 전략수립은 평가 메트릭 분석부터!

  | 기호 | 설명                             | 가중치 |
  | ---- | -------------------------------- | ------ |
  | A    | 정상 저장된 이미지 수            | 1.0    |
  | B    | 저장되지 않고 누락된 이미지 수   | -0.8   |
  | C    | 삭제되지 않고 남아있는 이미지 수 | -1.2   |
  | D    | 잘못된 데이터 수                 | -3.0   |
  | E    | 총 쿼리양                        | -0.01  |

 최종점수는 아래 공식으로 산출합니다.
- max(0, 1.0 * A - 0.8 * B - 1.2 * C - 3.0 * D) + 512 - 0.01 * E

평가 메트릭은 최대한 현실적인 부분을 반영하려고 노력했습니다.

누락되거나 삭제되지 않은 이미지에는 페널티를 부여하였는데, 누락보다는 삭제되지 않은 경우에 더 큰 페널티를 주었습니다. 이는 실 상황을 더 반영한 것인데, 누락된 것보다는 삭제되지 않은 이미지가 더 치명적이기 때문입니다. 사용자에게 노출되지 말아야 할 이미지가 삭제되지 않고 노출된 것이, 노출되었으면 좋았을 이미지가 누락되어 노출되지 않은 것보다는 더 치명적일테니까요. 마찬가지로 잘못된 데이터의 경우 페널티가 가장 큽니다. 데이터가 잘못된 경우 시스템을 서서히 망가뜨리는 데다가, 이를 알아차리기가 쉽지 않기 때문입니다. 

총 쿼리 수에 마이너스 페널티를 준 것은 적절히 요청하지 않아도 되는 것들을 필터링하라는 의도가 반영되었습니다.

## 고득점을 하려면 병렬 처리를!
가장 빈번하게 호출해야 하는 API는 /image/feature입니다. 해당 API의 웰컴 서버에서의 처리 속도는 요청하는 개수에 따라 약 최소 16ms ~ 최대 120ms 까지 걸리도록 디자인하였습니다. 네트워크 지연 시간을 포함하면 싱글 스레드로는 초당 요청 제한인 50건을 채우지 못할 수 있습니다. 따라서 스레드 혹은 프로세스를 최소 2개 이상 생성해야 초당 요청 수 제한을 상회할 수 있습니다.

참고로 웰컴 서버 처리 속도는 배치 처리 개수에 따라 exp 그래프 형태를 취하고 있습니다. 최대 배치 처리 사이즈인 50개씩 보내는 것보다 30~40개 정도가 가장 효율이 좋도록 설계되어 있습니다. 대다수의 지원자가 단건, 혹은 50개씩 요청을 보냈는데, 30개씩 보낸 지원자도 있어 놀랐습니다. 


![배치 처리 개수에 따른 응답 시간](http://t1.kakaocdn.net/welcome2018/round2_batch.png)


## 예외처리는 필수
실제 이미지 크롤링을 한다고 생각해봅시다. 크롤링하려고 URL을 큐에 넣어두었는데 그 사이 사용자가 이미지를 삭제했을 수도 있고, 해당 서버가 잠시 내려갔을 수도 있습니다. 웰컴 서버도 이를 시뮬레이션했습니다. 모든 API는 특정 확률 값에 따라 실패하도록 되어 있습니다. 그리고 특정 이미지는 특징값 추출이 매번 실패하도록 고안되었습니다. 재시도 예외처리를 하지 않았다면 해당 요청은 누락되어 감점이 될 것이고, 반복적으로 특징값 추출에 실패하는 이미지를 블록처리하지 않았다면 무의미한 요청이 쌓여 페널티를 받게 됩니다.

## 다양한 시나리오에 대응하라!
문서 내 이미지들의 추가/삭제 간에는 다양한 시나리오가 존재합니다.

- 정상적으로 추가 혹은 삭제되는 경우
- 이미 기존에 추가한 이미지를 또 추가하는 경우
- 방금 추가한 이미지를 바로 삭제하는 경우
- 추가된 적 없는 이미지를 삭제하라고 오는 경우
- 추가, 삭제, 추가가 연달아 오는 경우

모두 현실에 있을 법한 유형들입니다. 보통 인터넷에는 아주 많은 중복 이미지가 존재하고, 이 모든 이미지를 색인하다면 공간 낭비가 크기 때문에, 기존에 이미 추가된 이미지인지 확인하는 단계가 필요합니다. 우리의 온라인 테스트에서는 간단한 메모리 캐시로도 충분합니다. 한 문서 내에 추가/삭제 명령이 연달아 오는 경우도 있습니다. 이 경우에는 웰컴 서버에 추가했다가 삭제할 것이 아니라 서로 상쇄시키면 불필요한 API 요청을 줄일 수 있습니다. 추가된 적 없는 이미지를 삭제하라고 오는 경우도 있습니다. 이 경우도 메모리 캐시를 사용하고 있다면 쉽게 걸러낼 수 있습니다. 마지막 경우는 추가/삭제를 상쇄하는 연산을 할 때 단순히 set을 사용하면 안 되도록 넣어두었습니다. Add A, Del A, Add A 가 올 경우 최종적으로 Add A 만 요청해야 하는데, set으로 합쳐버리면 Add A, Del A 만 남아서 서로 상쇄되어 누락될 것 입니다.

여기서 한 가지 더, 이미지 추가/삭제는 문서 전 영역에 걸쳐 일어납니다. 즉 첫 번째 문서에서 추가한 이미지를 50번째 문서에서 삭제할 수 있습니다. 따라서 50건이 쌓일 때마다 추가/삭제 연산을 요청한 지원자의 경우 이미지 누락을 피할 수 없습니다. 그렇다고 계속 쌓아두다가 막판에 넣으려 하다보면 제한 시간 10분을 초과하여 이미지를 다 넣지 못하는 사태가 생길 수도 있지요. 적절한 선에서 의사결정을 해야 합니다. (트레이드오프) 

## 총 5개의 카테고리
이 곳에도 장치가 마련되어 있습니다. 카테고리별로 이미지의 추가/삭제 비율과 next_url 이 갱신되는 시간에 차등이 있습니다. 시시각각 업데이트되는 news 카테고리 같은 경우 next_url 이 매우 빈번하게 갱신이 됩니다. 반면 art 카테고리는 상대적으로 next_url 이 업데이트 안 될 확률이 7~8배 높습니다. 이미지 데이터량 자체는 blog 카테고리가 가장 풍부합니다. 다만 유저가 올리는 만큼 삭제되는 이미지도 많고, 추가했다가 삭제하는 등의 변동도 가장 크도록 설계되었습니다. 반면 정제된 news는 추가/삭제는 큰 반면 변동은 적습니다.

실제 내부 베타 테스트 진행 시 30만 점을 획득한 한 카카오 개발자는 5개의 카테고리 중 2개는 제외하고 3개 카테고리만 수집하여 30만점 이상의 고득점을 올렸습니다. 이번 온라인 2차 테스트에서도 일부 지원자가 카테고리 1개씩만 시도하는 경우가 발견되었습니다. 단, 여기에도 물론 트레이드오프가 존재합니다. 1개 카테고리만 수집한다고 할 경우 병렬 처리를 하게 되면 문서 간 순서를 보장하는 것이 어려워집니다. 추가적인 구현이 필요하거나 혹은 순서가 뒤틀림에 따라 받게 되는 누락 페널티를 무시하고 양으로 승부할 수도 있겠지요. 다시 한 번, 모든 것이 다 트레이드오프입니다.

## 마치며
 여러분은 어느 정도까지 파악하여 진행하셨나요? 

 합격선은 8만 점으로 병렬 처리를 하지 않았더라도 예외처리를 하고, 배치 처리를 했다면 충분히 받을 수 있는 점수이지만, 문제출제위원회는 이렇게 다양하게 장치를 마련해두었고, 또 이런 장치들이 실무에서도 흔하게 발생하는, 그래서 여러분들이 앞으로 고민할 수도 있는 트레이드오프였다는 것을 이야기하고 싶습니다.

 오랜 시간 고생하셨습니다!

# 대회 이모저모

## 언어별 통계

![합격자의 언어 사용 현황](http://t1.kakaocdn.net/welcome2018/round2_language.png)

2차 합격자들의 언어 분포입니다. 1차 테스트에서는 C++가 25%로 가장 높았으나 2차 테스트에서는 파이썬이 42.1%로 압도적으로 높은 비율을 보였고, 자바가 35.1% 로 뒤를 이었습니다. 1차에서는 테스트 플랫폼의 제약 때문에 보이지 않던 C#, Objective-C도 보이고요. 차트에는 보이지 않지만, Go로 푼 지원자도 1명 있었습니다.

## 점수 분포

![최종점수 구간별 비율](http://t1.kakaocdn.net/welcome2018/round2_score.png)

 최고점은 235,969점이며 응시자 전체 평균은 47,705점입니다. 
 
 대략적인 코드 분석 결과는 아래와 같습니다.

|  병렬처리  |  배치처리  |  예외처리  |  최종점수 |
| ---------- | ----------- | ---------- | --------- |
| X | X | O | 5만점 내외 |
| X | O | O | 14만점 내외 |
| O | O | O | 25만점 내외 |

## 트래픽
 2차 온라인 코딩테스트를 준비하면서 가장 심혈을 기울인 부분은 "트래픽 테스트" 입니다. 서버와 통신을 주고받는 형태이다 보니 서버의 응답이 느려지거나 자칫 서버가 죽기라도 한다면 테스트에 치명적이기 때문입니다.

![초과 요청 및 요청량 그래프](http://t1.kakaocdn.net/welcome2018/round2_traffic.png)

 우선 일차적으로 토큰별 요청량은 초당 50개로 제한하였습니다. 첫 번째 그래프가 같은 토큰으로 초당 50개 이상씩 던지는 사용자의 요청을 Drop 한 결과입니다. 한 개의 토큰으로 최고 800개/초 요청을 보낸 지원자도 있습니다.

 스트레스 테스트는, 예상 트래픽을 산출(700여 명의 지원자 * 초당 50건 = 35,000건/초)한 후 약 2~3배의 트래픽에도 문제 없을 정도로 준비하였습니다. 다행스럽게도, 초당 트래픽의 최고치는 약 6000건으로 문제 없이 테스트를 치를 수 있었습니다.

## 이슈
2차 코딩테스트 진행 도중 두 가지 이슈가 있었습니다.
* 크로스 도메인(cross domain) 문제
  * 웹브라우저 환경에서 자바스크립트를 사용해 문제를 풀려고 시도한 지원자들이 크로스 도메인 문제에 막혀 한동안 서버에 접근할 수 없는 문제가 있었습니다. 웰컴 서버에 CORS([Cross Origin Resource Sharing](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing))를 활성화해 해결하였습니다.
* Non-minified JSON 파싱 문제
  * Feature Save/Delete API의 request body가 minified JSON 형태가 아닌 경우 일부 채점이 안되는 문제가 있었습니다. 테스트 종료 후 전수 검사를 통해 해당되는 지원자들은 불이익이 없도록 조치를 취했습니다.

매끄럽게 진행하지 못하여 불편을 겪으신 일부 지원자분들께 양해 말씀드립니다. 
