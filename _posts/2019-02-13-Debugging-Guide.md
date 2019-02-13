---
layout : default
title:  "Debugging Guide"
date:   2019-02-13 17:30:00
---

[home](/)

### Index
+ [수정한 코드가 안 돌아가요](#t1)
+ [값이 이상해요](#t2)
+ [seg가 떠요 or 엑세스 에러가 떠요](#t3)

#### 수정한 코드가 안 돌아가요<a name="t1"></a>
1. 기존 코드와 1:1 비교를 한다.
2. refactoring을 하였다면 빠진게 없는지 확인한다.

#### 값이 이상해요<a name="t2"></a>
1. 멀티 쓰레딩을 하였다면 스레딩 이슈의 유무를 확인한다.
    + private로 써야하는 변수를 shared로 쓰지 않는가?
    + 기본 데이터 타입이 아닌 자료형을 사용하였다면 이 자료형의 사용이 Thread safe한지 확인하였는가?
    + 여러 쓰레드에서 공통으로 사용하는 데이터가 thread safe 하게 연산되는가?

```C++
#pragma omp critical
                   {
                    simd_cmplx NLpsiLNL{};
                    for (size_t tap = 0; tap < m_numTaps; ++tap) {
                        NLpsiLNL += ch.m_NPsi[batch][tap] * tapL[tap];
                    }
                    for (size_t tap1 = 0; tap1 < m_numTaps; ++tap1) {
                        simd_cmplx psiLNL{};
                        for (size_t tap2 = 0; tap2 < m_numTaps; ++tap2) {
                            psiLNL += psiL[tap1][tap2] * tapL[tap2];
                        }
                        ch.m_k[batch][tap1] = psiLNL / (simd_cmplx(ch.m_gamma[0][batch] * lambdaDBar) + NLpsiLNL);
                    }

                   }
 ```
  이 코드에서 전체 쓰레드는 omp for 문으로 batch에 대해서 돌아간다. simd\_cplex는 local scope에서 선언이 되므로 이 값에대한 연산은 thread safe하다고 판단하였지만, 결과 값인 ch.m\_k\[batch\]\[tap1\]의 값에 오류가 발생하였고 해당 부분을 critical sectiion으로 두었더니 결과가 잘 나왔다. 해당 소스의 다른 부분들도 전부다 위와 같은 과정을 거치는데 thread 문제가 없었다. 현재는 방치해둔 상태지만 simd\_cmplx의 연산이 thread safe하게 돌지 않았다고 의심하는 중

2. 실수형 데이터에 대한 정확도 문제를 확인한다.
    + 연산되는 두 데이터의 크기 차이가 크면 오차가 심하다.

#### seg가 떠요 or 엑세스 에러가 떠요<a name="t3"></a>
1. 할당과 해제를 잘 하였는지 확인
    + sizeof()를 빼먹지는 않았는가.
    + iterator 범위가 어긋나지 않았는가
    + 해제 순서가 틀리지 않았는가?
    + 같은 포인터에 여러번 할당 or 해제를 하지는 않았는가
2. 할당 범위를 초과해서 접근하지 않았는지 확인
    + 접근시의 iterator 범위 변수와 할당시의 변수가 같은 것을 쓰는가?
    + 다른 것을 쓴다면 범위를 초과하는 경우가 발생하지 않는가?
